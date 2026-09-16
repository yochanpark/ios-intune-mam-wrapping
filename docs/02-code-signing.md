# 2. Apple Developer 자산 등록과 재서명

### 4.3 Apple Developer 자산 등록

#### App ID 7개 등록

기존 계정(`{WORK_ACCOUNT}`)은 소속사 팀 내 일반 멤버 권한으로, App ID 신규 등록 버튼 자체가 노출되지 않았습니다. **관리자 권한 계정으로 전환하여 진행**했습니다.

| Description | Bundle ID | Capability |
|---|---|---|
| INTUNE Test | `org.example.wraptestb` | App Groups, iCloud(CloudKit), Siri, Push |
| Telegram Share | `...b.Share` | App Groups |
| Telegram Widget | `...b.Widget` | App Groups |
| Telegram Siri | `...b.SiriIntents` | App Groups, Siri |
| Telegram NotifContent | `...b.NotificationContent` | App Groups |
| Telegram NotifService | `...b.NotificationService` | App Groups |
| Telegram Broadcast | `...b.BroadcastUpload` | App Groups |

#### App Group 및 iCloud Container 등록

```
App Group:        group.org.example.wraptestb
iCloud Container: iCloud.org.example.wraptestb
```

App Group은 Extension이 메인 앱과 데이터를 공유하는 유일한 통로이므로, Extension 유지를 위해 필수 항목이었습니다.

#### Provisioning Profile 발급 (7개)

각 App ID마다 iOS App Development 프로필을 생성했습니다.

**이 단계에서 겪은 시행착오**: 키체인에 동일한 이름의 인증서가 국가 코드(US/KR)만 다르게 2개 존재했고, 프로필마다 다른 인증서를 선택하면서 "프로필-인증서 불일치"로 설치가 반복 실패했습니다.

```
Apple Development: {DEVELOPER} ({SIGNING_CERT_ID})  ← C=US
Apple Development: {DEVELOPER} ({SIGNING_CERT_ID})  ← C=KR
```

**해결**: 프로필 생성 시 해당 이름의 인증서를 **전부 체크**하여, 어느 쪽으로 서명해도 매칭되도록 구성했습니다.

---

### 4.4 Entitlements 구성 및 수동 재서명

#### Bundle ID 일괄 변경

```bash
/usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier org.example.wraptestb" \
  Payload/Telegram.app/Info.plist

/usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier org.example.wraptestb.Share" \
  Payload/Telegram.app/PlugIns/ShareExtension.appex/Info.plist
# (나머지 5개 Extension 동일 처리)
```

#### Entitlements 파일 작성 (7개)

메인 앱용 최종 구성:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>application-identifier</key>
    <string>{TEAM_ID}.org.example.wraptestb</string>
    <key>com.apple.developer.team-identifier</key>
    <string>{TEAM_ID}</string>
    <key>aps-environment</key>
    <string>development</string>
    <key>com.apple.developer.icloud-container-identifiers</key>
    <array><string>iCloud.org.example.wraptestb</string></array>
    <key>com.apple.developer.icloud-services</key>
    <array><string>CloudKit</string></array>
    <key>com.apple.developer.ubiquity-kvstore-identifier</key>
    <string>{TEAM_ID}.org.example.wraptestb</string>
    <key>com.apple.developer.siri</key>
    <true/>
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.org.example.wraptestc</string>
        <string>group.org.example.wraptestb</string>
    </array>
    <key>get-task-allow</key>
    <true/>
    <key>keychain-access-groups</key>
    <array>
        <string>{TEAM_ID}.org.example.wraptestb</string>
        <string>{TEAM_ID}.com.microsoft.intune.mam</string>
        <string>{TEAM_ID}.com.microsoft.adalcache</string>
    </array>
</dict>
</plist>
```

Extension용은 `application-identifier`와 App Group만 포함하는 최소 구성으로 작성했습니다.

#### 서명 순서 (중요)

iOS 코드 서명은 **가장 안쪽 바이너리부터 바깥으로** 진행해야 합니다. 순서가 틀리면 바깥쪽 서명이 안쪽 서명을 무효화합니다.

```
Frameworks (.dylib 34개 + .framework 5개)
  ↓
Extensions (.appex 6개)
  ↓
메인 앱 (.app)
```

```bash
CERT="{SIGNING_CERT_SHA1}"

# 1단계: 내장 프레임워크 전체
for item in Payload/Telegram.app/Frameworks/*; do
  codesign -f -s "$CERT" "$item"
done

# 2단계: Extension별 entitlements 지정 서명
codesign -f -s "$CERT" --entitlements share.entitlements \
  Payload/Telegram.app/PlugIns/ShareExtension.appex
# (나머지 5개 동일)

# 3단계: 메인 앱
codesign -f -s "$CERT" --entitlements main_app.entitlements Payload/Telegram.app

# 검증
codesign --verify --deep --strict --verbose=2 Payload/Telegram.app
```

#### 이 단계에서 겪은 설치 실패와 원인 규명

실기기 설치를 시도하며 세 차례 서로 다른 에러를 만났고, 각각 원인이 달랐습니다.

**1) `A valid provisioning profile for this executable was not found` (0xe8008015)**

```
원인: Development 프로필인데 entitlements의 get-task-allow가 false
      (false는 배포용, Development는 true여야 함)
해결: 전 entitlements 파일의 get-task-allow를 true로 수정
```

**2) `The identity used to sign the executable is no longer valid` (0xe8008018)**

```
대상: Payload/Telegram.app/Frameworks/MtProtoKitFramework.framework
원인: 내장 프레임워크(.dylib 34개 + .framework 5개)를 서명하지 않음
      --deep 옵션만으로는 커버되지 않았음
해결: Frameworks 폴더 내 전체 항목을 명시적으로 개별 서명
```

**3) BroadcastUploadExtension만 프로필 불일치**

```
원인: 해당 프로필 생성 시 다른 인증서(C=KR)를 선택했음
해결: 프로필 재발급 시 관련 인증서 전부 포함
```

---

