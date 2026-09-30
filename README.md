# iOS 앱 Intune MAM Wrapping 기술 검증 작업 기록

| 한눈에 | |
|---|---|
| 질문 | App Extension이 많은 iOS 앱을, **Extension이 살아 있는 채로** Intune App Wrapping하고 MAM 정책으로 제어할 수 있는가 (고객사 모바일 보안 PoC) |
| 한 일 | 오픈소스 Telegram(Extension 6개)으로 App ID·Provisioning Profile 7개 발급 → 안쪽부터 바깥 순서 수동 재서명 → 실기기 크래시 2건(iCloud·Siri) 원인 규명 → Entra ID 앱 등록 → Wrapping Tool의 plist 기록 실패를 수동 주입으로 우회 → 앱 보호 정책 배포 |
| 결과 | **가능함을 실기기로 실증** — MAM 로그인·화면 캡처 차단 등 DLP 동작 확인. 커스텀 앱은 조건부 액세스 "앱 보호 정책 필요"를 통과할 수 없는 **구조적 제약**을 규명 |
| 기술 | iOS 코드 서명 · Entitlements · Xcode · Intune App Wrapping Tool · Microsoft Entra ID · MSAL · 조건부 액세스 |

---

## 1. 배경 및 목적

### 1.1 요구사항 발생 배경

고객사(고객사 A) 모바일 보안 PoC 진행 중, iOS MAM(Mobile Application Management) 영역에서 다음 질문이 제기되었습니다.

> "대상 앱 B처럼 App Extension(공유, 위젯, 알림 등)을 다수 포함한 앱을, Extension 기능이 살아있는 상태로 Intune App Wrapping 처리하고 MAM 정책으로 제어할 수 있는가?"

이 질문은 단순 설정 확인이 아니라 iOS 코드 서명 체계, Apple Developer Program 권한 구조, Microsoft Entra ID 인증 체계가 모두 얽힌 기술 검증 사안이었습니다.

### 1.2 검증 대상 선정

실제 대상 앱인 대상 앱 B는 개발사(앱 개발사)의 원본 keystore 및 미암호화 IPA 협조가 선행되어야 하므로, 즉시 착수가 불가능했습니다.

대신 **Telegram 오픈소스 버전**을 대체 검증 대상으로 선정했습니다.

| 선정 근거 | 내용 |
|---|---|
| 구조 유사성 | 대상 앱 B와 동일하게 다수의 App Extension 보유 |
| 접근성 | GitHub 공개 릴리스로 unsigned IPA 배포 |
| 검증 가치 | Extension이 살아있는 상태의 Wrapping 가능 여부를 동일 조건에서 확인 가능 |

### 1.3 최종 목표

```
원본 앱 확보 → Extension 유지 재서명 → 실기기 설치/실행 검증
→ Intune App Wrapping → MAM 정책 적용 → 실제 기능 제어 확인
```

---

## 2. 작업 환경

### 2.1 개발 환경

| 항목 | 내용 |
|---|---|
| OS | macOS 26.6.2 |
| IDE | Xcode 26.2 (Build 17C52) |
| 빌드 시스템 | Bazel 8.4.2 (Telegram-iOS 소스 빌드용) |
| Wrapping Tool | Microsoft Intune App Wrapping Tool for iOS 21.8.0 |
| 테스트 기기 | iPad Pro 12.9-inch (4th gen), iPadOS 18.7.8 / iPad Air 13-inch (M2), iPadOS 26.6 |

### 2.2 계정 및 인증 체계

| 구분 | 값 |
|---|---|
| Apple Developer Team | 소속사 법인 (Team ID: `{TEAM_ID}`) |
| 서명 인증서 | Apple Development: yochan park (`{SIGNING_CERT_ID}`) |
| 인증서 SHA-1 | `{SIGNING_CERT_SHA1}` |
| Entra ID 테넌트 | 소속사 IT서비스데스크 (`{TENANT_ID}`) |
| 테스트 계정 | {TEST_ACCOUNT} |

---

## 3. 전체 작업 흐름

```
[1] 소스 빌드 경로 시도 (Telegram-iOS Bazel 빌드)
     ↓ 한계 확인 → 경로 전환
[2] 릴리스 IPA 확보 (Extension 6개 포함)
     ↓
[3] Apple Developer 자산 등록 (App ID 7개, App Group, iCloud Container)
     ↓
[4] Provisioning Profile 발급 (7개)
     ↓
[5] Entitlements 구성 및 수동 재서명
     ↓
[6] 실기기 설치 및 크래시 원인 규명 (iCloud → Siri)
     ↓
[7] Entra ID 앱 등록 및 API 권한 부여
     ↓
[8] Intune App Wrapping 실행
     ↓
[9] Intune 앱 보호 정책 생성 및 배포
     ↓
[10] 실기기 정책 적용 검증
```

---

## 4. 단계별 상세 작업 내역

상세 작업 내역은 단계별로 나누어 `docs/` 아래 두었다.

| 문서 | 내용 |
|---|---|
| [1. 소스 빌드 시도와 IPA 구조 분석](docs/01-ipa-analysis.md) | 소스 빌드 경로의 한계 확인, 릴리스 IPA 확보 및 Extension 구조 분석 |
| [2. Apple Developer 자산 등록과 재서명](docs/02-code-signing.md) | App ID·App Group·iCloud Container 등록, entitlements 구성, Extension 유지 재서명 |
| [3. 실기기 설치와 런타임 크래시 규명](docs/03-device-verification.md) | 실기기 설치, 크래시 원인 추적, Extension 실동작 확인 |
| [4. Microsoft Entra ID 앱 등록](docs/04-entra-id-registration.md) | 앱 등록, 리디렉션 URI, API 권한 부여 |
| [5. Intune App Wrapping 실행](docs/05-app-wrapping.md) | Wrapping Tool 실행, plist 기록 실패 우회, MAM 설정 수동 주입 |
| [6. Intune 앱 보호 정책 구성](docs/06-protection-policy.md) | 정책 생성·배포, DLP 제어 동작 확인 |
| [7. 조건부 액세스 정책 이슈](docs/07-conditional-access.md) | 커스텀 앱이 "앱 보호 정책 필요" 조건을 통과하지 못하는 원인 |

이 과정에서 얻은 기술적 발견은 [findings.md](findings.md), 실무 적용 절차는 [checklist.md](checklist.md)에 정리했다.

---

## 5. 최종 검증 결과

### 5.1 성공 항목

| 검증 항목 | 결과 |
|---|---|
| Extension 6개 포함 IPA 확보 | ✅ |
| App ID / App Group / iCloud Container 등록 | ✅ |
| Provisioning Profile 7개 발급 | ✅ |
| Extension 유지 재서명 (Frameworks 39개 포함) | ✅ |
| 실기기 설치 및 정상 실행 | ✅ |
| Share / Widget / Siri Extension 실동작 | ✅ |
| Entra ID 앱 등록 및 API 권한 부여 | ✅ |
| Intune App Wrapping 실행 | ✅ |
| MAM 로그인 화면 정상 노출 | ✅ |
| Intune 앱 보호 정책 생성 및 배포 | ✅ |
| 화면 캡처 차단 등 DLP 제어 동작 | ✅ |

### 5.2 제약 사항

| 항목 | 내용 |
|---|---|
| 조건부 액세스 "앱 보호 정책 필요" | 커스텀 앱은 Microsoft 승인 앱 목록 부재로 통과 불가 |
| Wrapping Tool plist 기록 실패 | macOS 26.6.2 환경에서 `defaults` 호출 실패, 수동 주입 필요 |
| 원본 Capability 일부 재현 불가 | Associated Domains, In-App Payments (도메인/계약 소유권 필요) |
| 대상 앱 B 직접 적용 | 개발사 원본 keystore 및 미암호화 IPA 협조 선행 필요 |

---

## 6. 주요 기술적 발견 → [findings.md](findings.md)

## 7. 실무 적용 체크리스트 → [checklist.md](checklist.md)

---

## 8. 결론

### 8.1 검증 결론

**App Extension을 유지한 상태의 iOS 앱 Intune Wrapping은 기술적으로 가능합니다.**

Extension 6개를 전부 포함한 오픈소스 Telegram 앱을 대상으로, 재서명부터 Wrapping, 실기기 설치, MAM 정책 적용, DLP 제어 동작까지 전 과정을 실증했습니다.

### 8.2 실무 적용 시 선행 조건

1. **개발사 협조**: 대상 앱의 미암호화 IPA 확보 (App Store 배포본으로는 불가)
2. **3개 계정 권한**: Apple Developer / Entra ID / Intune 각각의 관리 권한
3. **조건부 액세스 검토**: "앱 보호 정책 필요" 조건이 활성화된 환경에서는 커스텀 앱 통과 불가
4. **운영 프로세스**: 원본 앱 버전업 및 Wrapping Tool 업데이트(6~8주 주기)마다 재작업 필요

### 8.3 대상 앱 B 적용 시 예상 이슈

| 항목 | 예상 난이도 | 비고 |
|---|---|---|
| 미암호화 IPA 확보 | 높음 | 앱 개발사 협조 필수 |
| ShareExtension 처리 | 낮음 | 본 검증에서 동일 구조 성공 |
| 원본 Capability 재현 | 중간 | 앱별 entitlements 사전 분석 필요 |
| 조건부 액세스 통과 | 높음 | 정책 조정 또는 예외 처리 협의 필요 |
| 지속 운영 | 중간 | 버전업 대응 프로세스 수립 필요 |

---

## 부록: 최종 산출물 목록

| 파일 | 내용 |
|---|---|
| `Telegram_extensions.ipa` | Extension 6개 포함 재서명 IPA |
| `Telegram_wrapped.ipa` | Intune Wrapping 완료 IPA |
| entitlements 파일 7종 | 메인 앱 + Extension 6개 |
| Provisioning Profile 7종 | 메인 앱 + Extension 6개 |

---

## 식별정보 마스킹

고객사 PoC 기록이라, 공개 가능한 형태로 식별정보를 치환해 두었다.
기술적 내용(코드 서명 구조, Wrapping 절차, 트러블슈팅)은 원본 그대로다.

| 자리표시자 | 원래 값의 성격 |
|---|---|
| `고객사 A` | PoC를 의뢰한 고객사명 |
| `대상 앱 B` / `앱 개발사` | 최종 적용 대상 그룹웨어 앱과 그 개발사 |
| `소속사` / `소속사 법인` | Apple Developer Program 계정을 보유한 소속 법인 |
| `{TENANT_ID}` / `{CLIENT_ID}` | Entra ID 테넌트·앱 클라이언트 ID |
| `{TEAM_ID}` / `{SIGNING_CERT_ID}` | Apple Developer Team ID·서명 인증서 ID |
| `{SIGNING_CERT_SHA1}` | 서명 인증서 SHA-1 지문 |
| `{WORK_ACCOUNT}` / `{TEST_ACCOUNT}` | 관리자·테스트 계정 |
| `{ADMIN}` | Intune 정책을 만든 관리자 계정 ID |
| `org.example.*` | 검증용 Bundle ID (수행 시점 날짜 제거) |
