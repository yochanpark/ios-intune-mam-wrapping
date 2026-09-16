# 5. Intune App Wrapping 실행

### 4.7 Intune App Wrapping 실행

#### 도구 준비

```
https://github.com/microsoftconnect/intune-app-wrapping-tool-ios/releases
```

버전 선택 시 주의가 필요합니다.

| 앱 컴파일 Xcode 버전 | 사용해야 할 Wrapping Tool |
|---|---|
| Xcode 26 | 21.x |
| Xcode 16 | 20.x 최신 |

소스 zip 내부에 `Microsoft Intune Application Restrictions Packager for iOS.dmg`가 포함되어 있으며, 마운트하면 실행 파일이 나옵니다.

```
/Volumes/IntuneMAMAppPackager/IntuneMAMPackager/Contents/MacOS/IntuneMAMPackager
```

#### 사전 분석 (-xe)

실제 Wrapping 전에 Extension 구조와 필요 entitlement를 확인할 수 있습니다.

```bash
IntuneMAMPackager -xe -i ~/Desktop/Telegram_extensions.ipa
```

```
확장 접미사: .Share
자격: {
    "com.apple.security.application-groups" = ("group.org.example.wraptestb");
    "get-task-allow" = 1;
}
(6개 Extension 전부 정상 인식)
```

#### Wrapping 실행

```bash
IntuneMAMPackager \
  -i ~/Desktop/Telegram_extensions.ipa \
  -o ~/Desktop/Telegram_wrapped.ipa \
  -p ~/Desktop/telegram_profiles/intune_test_dev4.mobileprovision \
  -c {SIGNING_CERT_SHA1} \
  -ac "{CLIENT_ID}" \
  -ar "msauth.org.example.wraptestb://auth" \
  -aa "https://login.microsoftonline.com/{TENANT_ID}" \
  -x '<array><string>/경로/Telegram_NotifService.mobileprovision</string>...</array>' \
  -v
```

`-x` 파라미터로 Extension별 프로필을 배열 형태로 전달합니다.

**주의**: `-x` 값은 반드시 **작은따옴표**로 감싸야 합니다. 큰따옴표를 쓰면 zsh가 `<`, `>`를 리다이렉션 문자로 해석하여 `parse error near '<'`가 발생합니다.

#### 핵심 이슈: IntuneMAM-info.plist 미기록

Wrapping이 "애플리케이션을 패키징했습니다" 성공 메시지로 종료되었음에도, 생성된 IPA를 검증한 결과 문제를 발견했습니다.

```bash
unzip -o -q Telegram_wrapped.ipa
plutil -p Payload/Telegram.app/IntuneMAM-info.plist | grep -i intune
# → 아무 결과 없음
```

클라이언트 ID, Authority 등 MAM 설정이 **전혀 기록되지 않은 상태**였습니다. 로그를 역추적한 결과 원인을 찾았습니다.

```
경고: 인증서 해지 확인에 대한 시스템 설정을 확인할 수 없습니다.
다음 오류로 인해 기본값을 실행하지 못했습니다.
Error Domain=IntuneAppPackager Code=1 "/usr/bin/defaults이(가) 오류로 인해 종료되었습니다."
```

**Wrapping Tool이 내부적으로 `/usr/bin/defaults` 명령으로 plist에 값을 기록하는데, macOS 26.6.2 환경에서 이 단계가 실패**하고 있었습니다. 더 문제는 이 실패가 **치명적 오류로 처리되지 않고 성공 메시지가 출력**된다는 점입니다.

검증 시도:

| 시도 | 결과 |
|---|---|
| `defaults` 단독 실행 테스트 | 정상 동작 |
| sudo로 Wrapping 실행 | 키체인 접근 불가로 codesign 실패 |
| 키체인 파티션 리스트 재설정 후 재실행 | defaults 에러는 사라졌으나 plist는 여전히 미기록 |
| 도구 재마운트 후 재실행 | 동일 |

#### 해결: plist 수동 주입

도구를 우회하여 필요한 값을 직접 기록했습니다.

```bash
PLIST="Payload/Telegram.app/IntuneMAM-info.plist"

/usr/libexec/PlistBuddy -c "Add :IntuneMAMSettings dict" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :IntuneMAMSettings:IntuneMAMClientID string {CLIENT_ID}" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :IntuneMAMSettings:IntuneMAMAuthority string https://login.microsoftonline.com/{TENANT_ID}" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :IntuneMAMSettings:IntuneMAMRedirectURI string msauth.org.example.wraptestb://auth" "$PLIST"
```

```bash
plutil -p "$PLIST" | grep -A3 IntuneMAMSettings
```

```
"IntuneMAMSettings" => {
    "IntuneMAMAuthority" => "https://login.microsoftonline.com/{TENANT_ID}"
    "IntuneMAMClientID" => "{CLIENT_ID}"
    "IntuneMAMRedirectURI" => "msauth.org.example.wraptestb://auth"
}
```

plist를 수정했으므로 앱 번들 무결성이 깨집니다. **메인 앱 재서명 후 재압축**이 필요합니다.

```bash
codesign -f -s "$CERT" --entitlements main_app.entitlements Payload/Telegram.app
codesign --verify --deep --strict --verbose=2 Payload/Telegram.app
zip -qr ~/Desktop/Telegram_wrapped_manual.ipa Payload
```

이후 실기기에서 **MAM 로그인 화면이 정상적으로 노출**되는 것을 확인했습니다.

```
"조직 데이터를 보호하려면 이 앱을 관리해야 합니다.
 이 작업을 완료하려면 회사 또는 학교 계정으로 로그인하세요."
```

원본 Telegram에는 존재하지 않는 화면으로, **MAM SDK가 앱에 정상 주입되어 동작 중**임을 의미합니다.

#### 추가 발견: keychain-access-groups 누락

수동 주입 버전에서 MAM 화면은 떴으나 로그인이 완료되지 않는 문제가 발생했습니다. Wrapping Tool이 생성한 IPA와 수동 재서명 IPA의 entitlements를 비교하여 원인을 특정했습니다.

```bash
diff <(codesign -d --entitlements :- A/Payload/Telegram.app 2>&1) \
     <(codesign -d --entitlements :- B/Payload/Telegram.app 2>&1)
```

Wrapping Tool 버전에만 존재하던 항목:

```xml
<key>com.apple.developer.team-identifier</key>
<string>{TEAM_ID}</string>

<key>keychain-access-groups</key>
<array>
    <string>{TEAM_ID}.org.example.wraptestb</string>
    <string>{TEAM_ID}.com.microsoft.intune.mam</string>      <!-- MAM 전용 -->
    <string>{TEAM_ID}.com.microsoft.adalcache</string>       <!-- MSAL 토큰 캐시 -->
</array>
```

**Wrapping Tool이 자동 추가하는 항목을, 수동 작성한 entitlements에서 누락**했던 것입니다. MAM SDK가 인증 토큰을 저장/공유할 키체인 공간이 없어 인증 흐름이 완결되지 못했습니다.

해당 항목을 entitlements에 추가하여 재서명했습니다.

---

