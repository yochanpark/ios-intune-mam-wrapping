## 6. 주요 기술적 발견 정리

### 6.1 iOS 코드 서명 구조

- 서명은 **안쪽(Frameworks) → 중간(Extensions) → 바깥(App)** 순서가 강제됨
- `--deep` 옵션은 내장 프레임워크를 완전히 커버하지 못하므로 개별 서명 필요
- Development 프로필은 `get-task-allow: true`, 배포 프로필은 `false`
- Extension은 각각 독립된 App ID와 Profile이 필요하며, App Group으로만 메인 앱과 통신

### 6.2 Entitlement 누락 시 동작 패턴

| 유형 | 해당 Capability | 진단 방법 |
|---|---|---|
| 강제 크래시 | iCloud/CloudKit, Siri | 크래시 리포트 스택의 프레임워크명 확인 |
| 기능 무동작 | Push, Associated Domains 등 | 로그 확인, 앱은 정상 실행 |

### 6.3 Intune MAM 인증 체계

```
앱 실행
  ↓
IntuneMAM-info.plist의 ClientID/Authority 읽기
  ↓
MSAL로 Entra ID 인증 (리디렉션 URI 매칭 필요)
  ↓
Microsoft Mobile Application Management API 접근
  ↓ (DeviceManagementManagedApps.ReadWrite 권한 필요)
Intune 서버에 앱 체크인
  ↓
할당된 앱 보호 정책 수신 및 적용
```

이 흐름 중 어느 한 단계라도 누락되면 전체가 실패하며, 각 단계마다 다른 에러 메시지가 출력됩니다.

### 6.4 계정 체계 분리

| 체계 | 용도 | 필요 권한 |
|---|---|---|
| Apple Developer Program | App ID, Profile, 서명 | Admin (App ID 등록 시) |
| Microsoft Entra ID | 앱 등록, API 권한 | 앱 등록 권한 + 관리자 동의 |
| Microsoft Intune | 앱 보호 정책 | Intune 관리자 |

세 체계는 완전히 독립적이며, **어느 하나라도 권한이 없으면 프로젝트가 중단**됩니다. 프로젝트 착수 전 3개 계정 권한 확보 여부를 먼저 확인해야 합니다.

---

