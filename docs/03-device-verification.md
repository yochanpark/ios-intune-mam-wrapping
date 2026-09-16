# 3. 실기기 설치와 런타임 크래시 규명

### 4.5 실기기 설치 및 런타임 크래시 규명

서명 검증을 통과하고 설치에 성공했으나, 앱 실행 시 즉시 크래시가 발생했습니다. 크래시 리포트(`.ips`)를 분석하여 원인을 특정했습니다.

#### 크래시 1: iCloud/CloudKit

```
Exception Type: EXC_BREAKPOINT (SIGTRAP)
Faulting Thread: 9

스택 트레이스:
  0-2, 5: CloudKit.framework    ← 크래시 지점
  6:      TelegramCoreFramework
  7:      SwiftSignalKitFramework
```

**분석**: 앱 초기화 과정에서 CloudKit을 사용하려는데, entitlements에서 iCloud 관련 항목을 제거했기 때문에 프레임워크가 강제 트랩을 발생시켰습니다.

iCloud/CloudKit은 권한이 없을 때 조용히 실패하는 것이 아니라 **프로세스를 강제 종료**시키는 타입임을 확인했습니다.

**해결**:
1. developer.apple.com에서 iCloud Container 신규 등록
2. App ID에 iCloud Capability 활성화 (Include CloudKit support)
3. Profile 재발급
4. entitlements에 3개 키 추가 후 재서명

이후 앱이 정상 실행되어 Telegram 로그인 화면까지 도달했습니다.

#### 크래시 2: Siri (전화번호 인증 직후)

```
Exception Type: EXC_BREAKPOINT (SIGTRAP)

핵심 프레임:
  Intents -[INPreferences _THROW_EXCEPTION_FOR_PROCESS_MISSING_ENTITLEMENT_com_apple_developer_siri]
```

**분석**: 함수명 자체가 원인을 명시하고 있습니다. `com.apple.developer.siri` entitlement 누락 상태에서 `INPreferences` 호출 시 의도적으로 예외를 던지도록 Apple이 설계해둔 것입니다.

**해결**:
1. 메인 앱 + SiriIntents Extension 양쪽 App ID에 Siri Capability 활성화
2. 두 Profile 재발급
3. `main_app.entitlements` 및 `siri.entitlements`에 `com.apple.developer.siri` 추가
4. 재서명

#### 이 과정에서 얻은 패턴

iOS의 entitlement 누락은 두 가지 양상으로 나타납니다.

| 유형 | 해당 Capability | 증상 |
|---|---|---|
| **강제 크래시형** | iCloud/CloudKit, Siri | 프로세스 즉시 종료 |
| **기능 무동작형** | Push, Associated Domains, Sign in with Apple, CarPlay | 해당 기능만 조용히 실패 |

강제 크래시형은 반드시 선제적으로 처리해야 하며, 크래시 리포트의 스택 트레이스에서 해당 프레임워크명을 확인하면 원인을 즉시 특정할 수 있습니다.

#### 실기기 동작 검증 결과

```
✅ 앱 정상 실행 (크래시 없음)
✅ Share Extension — 사진/사파리 공유 시트에 노출
✅ Widget Extension — 위젯 추가 목록에 노출
✅ SiriIntents — 설정 > Siri 항목에 노출
```

**Extension 6개를 유지한 상태의 재서명 및 실기기 배포가 기술적으로 가능함을 실증**했습니다.

---

