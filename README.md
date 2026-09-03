# Daily Quest RPG v4

공부/할 일을 RPG 퀘스트, 집중 원정, 재화, 장비 강화, 던전 탐험, 캐릭터 해금으로 바꾸는 개인 생산성 웹앱입니다.

## v4에서 구현된 것

- 오늘의 퀘스트: 난이도별 XP/골드 보상
- 무제한 완료 + 일일 반복 보상 감쇠
  - 1~8개: 100%
  - 9~16개: 80%
  - 17~24개: 65%
  - 25개 이후: 50%
- 집중 원정: 15/25/50/90분 타이머
- 하루 누적 집중량에 따른 집중 XP 감쇠
  - 0~3시간: 100%
  - 3~6시간: 85%
  - 6시간 이후: 70%
- 집중 완료 보상을 바로 던전에 소비하지 않고 `원정 인장`, `강화 광석`으로 축적
- 원정 인장 3개를 사용자가 원하는 시점에 소비해 던전 1칸 탐험
- 골드/보석/유물 조각/강화 광석/원정 인장/휴식 불씨 인벤토리
- 무기 제작/장착/최대 +10 강화
- 아이템 상점
- 캐릭터 5종 + 장기 집중 조건 기반 자동 해금
- 일별 통계 저장
- 이번 주/이번 달/올해 기준 XP·집중·퀘스트·레벨업 리더보드
- JSON 내보내기/불러오기
- 로컬 저장(localStorage)
- Firebase 설정 시 Google 로그인 + Cloud Firestore 동기화
- PWA manifest/service worker 포함

## 바로 실행

`index.html`을 브라우저에서 열면 로컬 모드로 대부분의 기능을 바로 사용할 수 있습니다.

서비스 워커/PWA와 Google 로그인을 테스트하려면 HTTP 서버로 여는 편이 좋습니다.

Windows에서 Python이 설치되어 있다면 프로젝트 폴더에서:

```bash
py -m http.server 8080
```

그 뒤 브라우저에서 `http://localhost:8080`을 엽니다.

## Firebase + Google 로그인 연결

이 앱은 Firebase JavaScript SDK 12.18.0의 브라우저 모듈을 필요할 때만 불러옵니다.

1. Firebase Console에서 프로젝트를 만듭니다.
2. Web App을 추가합니다.
3. Authentication → Sign-in method → Google을 활성화합니다.
4. Firestore Database를 생성합니다.
5. 이 폴더의 `firestore.rules`와 같은 규칙을 적용합니다.
6. 앱에서 `클라우드 설정` 버튼을 누릅니다.
7. Firebase가 제공한 `firebaseConfig` 객체를 **JSON**으로 붙여 넣습니다.
8. Google 로그인을 누릅니다.
9. GitHub Pages/Firebase Hosting 등으로 배포할 경우 Authentication → Settings → Authorized domains에 배포 도메인을 추가합니다.

예시 형식:

```json
{
  "apiKey": "...",
  "authDomain": "...",
  "projectId": "...",
  "storageBucket": "...",
  "messagingSenderId": "...",
  "appId": "..."
}
```

Firebase Web config의 `apiKey`는 서버 비밀키처럼 취급하는 키가 아닙니다. 실제 데이터 보호는 Firestore Security Rules와 인증으로 해야 합니다.

## Firestore 데이터 경로

현재 개인용 v4는 단순성을 위해 사용자 상태를 한 문서에 저장합니다.

```text
users/{uid}/app/main
  state: { ...전체 앱 상태... }
  updatedAt: serverTimestamp()
```

개인 사용에는 충분하지만, 향후 여러 기기 동시 편집/대규모 통계를 강화하려면 다음처럼 분리하는 것을 권장합니다.

```text
users/{uid}/profile/main
users/{uid}/quests/{questId}
users/{uid}/focusSessions/{sessionId}
users/{uid}/dailyStats/{YYYY-MM-DD}
users/{uid}/inventory/{itemId}
users/{uid}/activity/{eventId}
```

## 밸런스 핵심

### 퀘스트

기본 XP:

- 쉬움 10
- 보통 20
- 어려움 35
- 보스 60

실제 보상 = `기본 XP × 오늘 완료량 감쇠 계수`

완료 취소 시 실제 지급받았던 XP/골드를 정확히 회수하도록 기록합니다.

### 집중

기본 집중 XP는 대략 다음 형태입니다.

```text
base = min(70, round(8 + 5.2 × sqrt(집중분)))
reward = base × 일일 집중량 감쇠 × 아이템 보정
```

긴 세션이 더 큰 보상을 주지만 시간에 정비례하지 않아 15분 세션을 무한 반복하거나 90분 세션만 강제하는 양쪽 극단을 줄였습니다.

### 던전/장비

집중 완료 → 원정 인장/광석 축적 → 나중에 사용

즉 공부 직후 보상이 자동 소비되지 않고, 사용자가 쌓인 보상을 보면서 원하는 시점에 RPG 콘텐츠를 즐길 수 있습니다.

## 보안/치팅 한계

현재 버전은 개인 생산성 앱이므로 게임 경제 계산이 브라우저에서 실행됩니다. 개발자 도구로 값을 바꾸려는 사용자를 완전히 막을 수는 없습니다.

공개 경쟁 리더보드까지 확장할 경우에는 다음 단계가 필요합니다.

- 보상 계산을 Cloud Functions 같은 서버 쪽으로 이동
- 집중 세션 시작/종료를 서버 타임스탬프로 검증
- 중복 보상 방지 idempotency key
- Firestore Rules에서 직접 재화 임의 수정 차단
- App Check 적용

## GitHub 참고 방향

이 프로젝트는 공개 프로젝트의 코드를 복사하지 않고 구조적 아이디어만 참고하도록 설계했습니다.

- Habitica: 현실 할 일을 RPG 보상 구조로 연결하는 대표 사례
- Vikunja: 장기적으로 확장 가능한 할 일/프로젝트 관리 구조 참고
- Firebase quickstart-js / Firebase Web Docs: Google Auth + Firestore 사용 방식 참고

특히 Habitica는 자체 라이선스 및 기여 정책이 있으므로 코드를 그대로 가져오지 않는 것이 안전합니다. Vikunja는 AGPL 계열이므로 코드 재사용 시 라이선스 의무를 반드시 검토해야 합니다. 이 v4 코드는 위 저장소 코드를 복사하지 않고 새로 작성된 독립 구현입니다.

## 다음 버전 추천

1. 캐릭터 전용 PNG/WebP 일러스트 에셋 추가
2. 장비 외형이 캐릭터 일러스트에 반영되는 레이어 시스템
3. 과목별 퀘스트 통계(수학/영어/과학 등)
4. 시험기간 시즌 패스 대신 `학습 챕터` 구조
5. 주간 보스: 그 주 누적 집중 시간으로 HP를 깎는 방식
6. 여러 기기 동시 사용을 위한 Firestore 컬렉션 분리
7. Cloud Functions 기반 보상 검증
8. GitHub Actions로 자동 배포
