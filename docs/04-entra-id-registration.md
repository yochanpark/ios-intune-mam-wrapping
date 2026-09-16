# 4. Microsoft Entra ID 앱 등록

### 4.6 Microsoft Entra ID 앱 등록

Intune App Wrapping Tool 21.x부터 `-ac`(클라이언트 ID), `-ar`(리디렉션 URI)가 **필수 파라미터**가 되었습니다. 즉 Entra ID 앱 등록 없이는 Wrapping 실행 자체가 불가능합니다.

#### 테넌트 확보 과정

이 단계에서 상당한 우회를 거쳤습니다.

| 시도 | 결과 |
|---|---|
| 회사 Entra 계정 사용 | 계정 미보유 (Apple Developer 계정과 별개 체계) |
| M365 개발자 프로그램 샌드박스 | 자격 미달로 거부 |
| 개인 Microsoft 계정으로 앱 등록 | "디렉터리 외부 앱 생성 불가" 정책으로 차단 |
| Azure 무료 평가판으로 테넌트 생성 | **성공** (임시 검증용) |
| 회사 정식 관리자 계정 확보 | **최종 사용** |

**교훈**: Apple Developer 계정과 Microsoft Entra/Intune 계정은 완전히 별개 체계입니다. iOS MAM 프로젝트는 **양쪽 모두의 권한 확보가 선행 조건**입니다.

#### 앱 등록 설정

```
이름:            Telegram Wrapping Test
계정 유형:        단일 테넌트만
플랫폼:          퍼블릭 클라이언트/네이티브 (모바일 및 데스크톱)
리디렉션 URI:     msauth.org.example.wraptestb://auth
클라이언트 ID:    {CLIENT_ID}
테넌트 ID:       {TENANT_ID}
```

추가로 **"공용 클라이언트 흐름 허용"을 활성화**해야 합니다. iOS 네이티브 앱은 client secret 없이 인증하므로, 이 설정이 꺼져 있으면 MSAL 인증이 거부됩니다.

#### API 권한 부여 — 명칭 함정

MAM 인증에 필요한 API를 찾는 과정에서 시행착오가 있었습니다.

```
❌ "Intune"으로 검색 → Microsoft Intune API만 노출
   (여기에는 MicrosoftTunnelGatewayEnrollment 권한만 존재, 인증용 권한 없음)

✅ "Microsoft Mobile Application Management"로 검색
   → DeviceManagementManagedApps.ReadWrite 권한 확인
```

이 권한을 부여하지 않으면 앱 실행 시 다음 에러가 발생합니다.

```
MSALErrorDomain 오류 -50000

Entra 로그인 로그 상세:
"Invalid resource. The client has requested access to a resource which is not
 listed in the requested permissions in the client's application registration."
```

권한 추가 후 **관리자 동의 부여**까지 완료하면 인증이 통과됩니다.

---

