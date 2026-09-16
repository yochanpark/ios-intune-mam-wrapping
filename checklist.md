## 7. 실무 적용을 위한 체크리스트

### 7.1 착수 전 확인 사항

```
□ Apple Developer Program 계정 (App ID 등록 권한 보유자 확인)
□ Microsoft Entra ID 계정 (앱 등록 권한 + 관리자 동의 가능 여부)
□ Microsoft Intune 관리 권한 (앱 보호 정책 생성)
□ macOS + Xcode 환경 (Wrapping Tool 버전 호환 확인)
□ 테스트 실기기 (UDID를 Profile에 등록 필요)
□ 대상 앱의 미암호화 IPA (App Store 배포본 불가)
□ 조직 내 조건부 액세스 정책 현황 파악
```

### 7.2 대상 앱 분석 절차

```bash
# 1. Extension 목록 확인
ls Payload/앱.app/PlugIns/

# 2. 각 Extension Bundle ID 확인
/usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" 각.appex/Info.plist

# 3. 원본 entitlements 확인
codesign -d --entitlements :- Payload/앱.app

# 4. Wrapping Tool 사전 분석
IntuneMAMPackager -xe -i 앱.ipa
```

이 4단계로 **필요한 App ID 개수, Capability 목록, 재현 가능 여부**를 사전에 판단할 수 있습니다.

### 7.3 트러블슈팅 레퍼런스

| 증상 | 원인 | 해결 |
|---|---|---|
| `0xe8008015` provisioning profile not found | `get-task-allow` 불일치 또는 프로필-인증서 미매칭 | entitlements 수정 / 프로필 재발급 |
| `0xe8008018` identity no longer valid | 내장 프레임워크 미서명 | Frameworks 개별 서명 |
| `errSecInternalComponent` | 키체인 접근 승인 미등록 | `security set-key-partition-list` |
| `MSALErrorDomain -50000` | API 권한 누락 또는 리디렉션 URI 불일치 | Entra 권한 부여 / URI 확인 |
| "이 앱을 사용하도록 설정하지 않았습니다" | 조건부 액세스 차단 | 정책 확인 및 예외 처리 |
| plist에 MAM 설정 미기록 | `defaults` 실행 실패 | PlistBuddy 수동 주입 후 재서명 |
| `parse error near '<'` | zsh의 `<`, `>` 해석 | `-x` 값을 작은따옴표로 감싸기 |

---

