# 1. 소스 빌드 시도와 IPA 구조 분석

### 4.1 소스 빌드 경로 시도 및 한계 확인

#### 진행 내용

Telegram-iOS 공식 리포지토리를 클론하여 Bazel 기반 빌드 환경을 구성했습니다.

```bash
git clone --recursive https://github.com/TelegramMessenger/Telegram-iOS.git
python3 build-system/Make/Make.py --cacheDir="$HOME/telegram-bazel-cache" \
  generateProject \
  --configurationPath=build-system/template_minimal_development_configuration.json \
  --xcodeManagedCodesigning
```

빌드 설정 파일(`template_minimal_development_configuration.json`)에 Bundle ID, Team ID, Telegram API 자격증명을 기입해야 합니다.

```json
{
  "bundle_id": "org.example.wraptestb",
  "api_id": "1",
  "api_hash": "0000000000000000000000000000000",
  "team_id": "{TEAM_ID}",
  "enable_siri": false,
  "enable_icloud": false
}
```

#### 발견한 구조적 한계

`Make.py` 소스를 분석한 결과, 다음 코드를 확인했습니다.

```python
if arguments.xcodeManagedCodesigning is not None and arguments.xcodeManagedCodesigning == True:
    disable_extensions = True
```

**Xcode 자동 서명(`--xcodeManagedCodesigning`)을 사용하면 Extension이 무조건 강제 제거되도록 설계되어 있었습니다.** Telegram 개발팀이 의도적으로 넣은 제약으로, Xcode 자동 서명 방식으로는 Extension 6개의 App ID/Profile을 안정적으로 처리할 수 없기 때문입니다.

이 발견으로 **소스 빌드 + 자동 서명 경로로는 Extension 유지가 불가능**함을 확인하고, 릴리스 IPA 직접 재서명 방식으로 전환했습니다.

#### 부수적으로 해결한 이슈

| 이슈 | 원인 | 해결 |
|---|---|---|
| `No MODULE.bazel found` | git submodule 미초기화 | `git submodule update --init --recursive` |
| `Required Xcode version is 26.2, but 26.6` | 버전 불일치 | `xcodes install 26.2` 후 `xcode-select -s`로 전환 |
| `Directory not empty` 캐시 삭제 실패 | Bazel 산출물 immutable 플래그 | `chflags -R nouchg` 후 삭제 |
| `errSecInternalComponent` 서명 실패 | 키체인 접근 승인 미등록 | `security set-key-partition-list`로 codesign 영구 허용 |

---

### 4.2 릴리스 IPA 확보 및 구조 분석

#### IPA 다운로드

GitHub Releases에서 unsigned 빌드를 확보했습니다.

```
https://github.com/TelegramMessenger/Telegram-iOS/releases/tag/build-26855
버전: Telegram 10.0.3 (build 26855), 81.8MB
```

unsigned 배포판이라는 점이 중요했습니다. App Store 배포본은 Apple이 자동 암호화하여 Wrapping이 불가능하지만, 이 빌드는 그 제약이 없습니다.

#### Extension 구조 확인

```bash
unzip -q Telegram.ipa
ls Payload/Telegram.app/PlugIns/
```

```
BroadcastUploadExtension.appex
IntentsExtension.appex
NotificationContentExtension.appex
NotificationServiceExtension.appex
ShareExtension.appex
WidgetExtension.appex
```

**Extension 6개 전부 포함**되어 있음을 확인했습니다.

#### 원본 Bundle ID 및 Entitlements 분석

```bash
for ext in Payload/Telegram.app/PlugIns/*.appex; do
  echo "=== $(basename "$ext")"
  /usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" "$ext/Info.plist"
done
```

| Extension | 원본 Bundle ID |
|---|---|
| 메인 앱 | `ph.telegra.Telegraph` |
| BroadcastUpload | `ph.telegra.Telegraph.BroadcastUpload` |
| IntentsExtension | `ph.telegra.Telegraph.SiriIntents` |
| NotificationContent | `ph.telegra.Telegraph.NotificationContent` |
| NotificationService | `ph.telegra.Telegraph.NotificationService` |
| Share | `ph.telegra.Telegraph.Share` |
| Widget | `ph.telegra.Telegraph.Widget` |

원본 entitlements를 추출한 결과, 예상보다 훨씬 복잡한 권한 구조를 확인했습니다.

```bash
codesign -d --entitlements :- Payload/Telegram.app
```

| Capability | 재현 가능 여부 |
|---|---|
| App Group (`group.ph.telegra.Telegraph`) | 가능 (자체 그룹으로 대체) |
| Push Notification | 가능 |
| Siri | 가능 |
| iCloud (CloudKit, KVStore) | 가능 (자체 컨테이너로 대체) |
| Sign in with Apple | 제외 판단 |
| Associated Domains (`t.me`) | **불가** (도메인 소유권 필요) |
| In-App Payments (merchant ID 16개) | **불가** (결제사 계약 기반) |
| CarPlay / PushKit VoIP | 제외 판단 |

**판단 근거**: PoC 목적은 "Extension이 살아있는 상태의 Wrapping 가능 여부 검증"이므로, Extension 동작에 직접 관련된 권한(App Group, Push, Siri, iCloud)만 유지하고 재현 불가능하거나 무관한 항목은 제거하는 전략을 선택했습니다.

---

