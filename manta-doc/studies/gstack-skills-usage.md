# Codex에서 gstack skills 사용하기

## 이 챕터의 목표

이 문서는 Manta 작업 중 Codex에 등록된 gstack skills를 언제, 어떻게 쓰면 좋은지 정리한다.

여기서 말하는 gstack은 별도의 앱이나 라이브러리 API가 아니라,
현재 Codex 세션의 skills로 등록된 `/qa`, `/investigate`, `/review`, `/ship` 같은 작업 도구들을 뜻한다.

중요한 것은 모든 skill 이름을 외우는 것이 아니다.
아래 질문에 답할 수 있으면 된다.

- 지금 작업은 어떤 종류의 도움을 받아야 하는가?
- Codex에게 어떤 문장으로 요청하면 되는가?
- Manta 작업 흐름에서는 어느 시점에 어떤 skill이 맞는가?
- skill을 쓰면 안 되는 상황은 무엇인가?

---

## 1. gstack skill은 Codex에게 맡기는 작업 모드다

gstack skill은 사용자가 직접 셸에서 실행하는 명령어라기보다,
Codex가 특정 종류의 작업을 더 체계적으로 처리하도록 만드는 작업 지침이다.

예를 들어 다음 요청은 서로 다른 skill을 부른다.

```text
/investigate 이 에러 원인 찾아줘
/qa 로컬 사이트 테스트하고 깨진 부분 고쳐줘
/review 현재 diff 리뷰해줘
/ship PR 만들어줘
```

항상 슬래시 명령처럼 말해야 하는 것은 아니다.
Codex는 요청이 skill 설명과 잘 맞으면 해당 skill을 사용할 수 있다.

```text
이 버그 왜 나는지 원인부터 찾아줘
사이트 QA 해줘
배포 전에 diff 리뷰해줘
작업 맥락 저장해줘
```

다만 중요한 작업에서는 skill 이름을 직접 쓰는 편이 더 명확하다.
특히 `ship`, `land-and-deploy`, `guard`, `freeze`처럼 작업 범위나 부작용이 큰 흐름은 명시적으로 요청하는 것이 좋다.

---

## 2. Manta 작업 흐름에서 자주 쓰는 skill

Manta의 기본 흐름은 문서로 작업을 정의하고, 코드 repo에서 구현하고, 검증한 뒤 리뷰와 커밋으로 닫는 것이다.
gstack skills는 이 흐름의 각 지점에서 역할이 다르다.

| 상황 | 사용할 skill | 요청 예시 |
|---|---|---|
| 문제 원인을 먼저 밝혀야 함 | `/investigate` | `/investigate task list가 빈 결과를 내는 원인 찾아줘` |
| 구현 전 설계를 더 단단히 만들고 싶음 | `/plan-eng-review` | `/plan-eng-review 이 구현 계획 검토해줘` |
| 제품 범위나 우선순위를 재검토하고 싶음 | `/plan-ceo-review` | `/plan-ceo-review 이 기능 범위가 맞는지 봐줘` |
| UI나 화면 품질을 확인하고 싶음 | `/qa` | `/qa 로컬 앱에서 task 생성 흐름 테스트하고 고쳐줘` |
| 버그 리포트만 받고 싶음 | `/qa-only` | `/qa-only 수정하지 말고 문제만 보고해줘` |
| diff를 머지 전에 검토하고 싶음 | `/review` | `/review 현재 변경사항 리뷰해줘` |
| 문서와 changelog를 맞추고 싶음 | `/document-release` | `/document-release 이번 변경에 맞춰 문서 업데이트해줘` |
| PR 생성까지 진행하고 싶음 | `/ship` | `/ship 이 변경사항 PR로 올려줘` |
| 작업 맥락을 저장하고 싶음 | `/context-save` | `/context-save 지금 진행 상황 저장해줘` |
| 이전 작업 맥락을 복원하고 싶음 | `/context-restore` | `/context-restore 지난번 작업 이어서 보여줘` |

Manta에서는 작은 문서 수정이나 좁은 리팩터처럼 바로 처리 가능한 작업도 많다.
그런 경우에는 굳이 큰 skill을 부르지 않아도 된다.

---

## 3. 목적별 gstack skill 묶음

현재 Codex에 등록된 gstack skills는 많기 때문에, 이름보다 목적별로 기억하는 편이 쉽다.
목록은 현재 세션 기준이며 gstack 업그레이드 후 달라질 수 있다.

### 계획과 검토

| skill | 용도 |
|---|---|
| `/autoplan` | CEO, 디자인, 엔지니어링, DX 관점의 자동 계획 리뷰 |
| `/plan-ceo-review` | 제품 방향, 범위, 우선순위 검토 |
| `/plan-eng-review` | 아키텍처, 데이터 흐름, 테스트 계획 검토 |
| `/plan-design-review` | 디자인 계획의 시각 품질 검토 |
| `/plan-devex-review` | 개발자 경험 계획 검토 |
| `/plan-tune` | gstack 질문 방식과 선호도 조정 |
| `/office-hours` | 아이디어, 제품 방향, 초기 기획 상담 |

계획 단계에서는 이 skill들이 구현을 바로 하기보다,
무엇을 만들지와 어떤 위험을 줄일지 정리하는 데 쓰인다.

### 구현 후 품질 확인

| skill | 용도 |
|---|---|
| `/investigate` | 버그나 에러의 원인 조사 |
| `/qa` | 웹 앱을 실제로 테스트하고 발견한 문제 수정 |
| `/qa-only` | 수정 없이 QA 리포트만 작성 |
| `/review` | 현재 diff의 코드 리뷰 |
| `/health` | 테스트, lint, 타입 체크 등 코드 건강 상태 점검 |
| `/benchmark` | 성능 회귀와 페이지 속도 점검 |
| `/benchmark-models` | 여러 모델의 skill 수행 결과 비교 |
| `/cso` | 보안 관점 점검 |

Manta 작업에서 가장 자주 쓰는 조합은 `/investigate`, `/qa`, `/review`다.
원인 조사가 필요한 버그는 바로 고치기보다 `/investigate`로 시작하는 편이 안전하다.

### 브라우저와 디자인

| skill | 용도 |
|---|---|
| `/browse` | 빠른 headless 브라우저 조작과 스크린샷 |
| `/open-gstack-browser` | 눈으로 볼 수 있는 gstack 브라우저 실행 |
| `/setup-browser-cookies` | 인증이 필요한 QA를 위한 쿠키 가져오기 |
| `/design-consultation` | 디자인 시스템과 방향 제안 |
| `/design-shotgun` | 여러 디자인 시안 탐색 |
| `/design-html` | 승인된 디자인을 HTML/CSS로 구체화 |
| `/design-review` | 구현된 UI의 시각 QA와 수정 |

화면이 있는 기능에서는 코드만 보는 리뷰보다 브라우저 기반 QA가 더 많은 문제를 잡는다.
레이아웃, 버튼 상태, 반응형 문제는 `/qa`나 `/design-review`가 더 적합하다.

### 릴리스와 운영

| skill | 용도 |
|---|---|
| `/ship` | 테스트, 리뷰, 버전/문서 정리, PR 생성 |
| `/land-and-deploy` | PR 머지, 배포, 배포 후 검증 |
| `/canary` | 배포 후 페이지 오류와 성능 감시 |
| `/setup-deploy` | 배포 플랫폼과 health check 설정 |
| `/landing-report` | ship 전에 버전 슬롯과 주변 작업 상태 확인 |
| `/document-release` | 릴리스 후 문서, changelog, README 정리 |

이 묶음은 부작용이 큰 편이다.
PR 생성, 머지, 배포를 원할 때는 사용자가 명확히 요청해야 한다.

### 작업 안전과 맥락 관리

| skill | 용도 |
|---|---|
| `/careful` | 위험한 명령 실행 전 안전 확인 |
| `/freeze` | 편집 가능한 디렉터리 제한 |
| `/unfreeze` | 편집 제한 해제 |
| `/guard` | 안전 확인과 편집 제한을 함께 사용 |
| `/context-save` | 현재 작업 맥락 저장 |
| `/context-restore` | 저장된 작업 맥락 복원 |
| `/learn` | 프로젝트 학습 기록 조회와 관리 |
| `/retro` | 일정 기간의 작업 회고 |

Manta처럼 여러 git repo가 같은 workspace에 있는 구조에서는 `/freeze`나 `/guard`가 특히 유용할 수 있다.
예를 들어 문서만 고치는 작업이라면 `manta-doc` 밖의 편집을 막도록 요청할 수 있다.

### 기타 도구

| skill | 용도 |
|---|---|
| `/claude` | Claude Code CLI를 통한 독립 리뷰나 상담 |
| `/devex-review` | 실제 문서와 시작 흐름을 따라가며 DX 점검 |
| `/make-pdf` | Markdown을 PDF로 변환 |
| `/pair-agent` | 다른 AI agent와 브라우저 세션 연결 |
| `/setup-gbrain` | gbrain 설정 |
| `/upgrade` | gstack 최신 버전으로 업그레이드 |

이 skill들은 매일 쓰기보다는 특정 상황에서 필요할 때 꺼내는 도구에 가깝다.

---

## 4. 좋은 요청 문장 만들기

gstack skill을 쓸 때는 “무엇을 해줘”보다 “어떤 범위에서 어떤 결과를 원한다”까지 말하는 편이 좋다.

나쁜 요청:

```text
/qa 해줘
```

좋은 요청:

```text
/qa 로컬 앱에서 task 생성, 목록 조회, 상세 보기 흐름을 테스트하고,
발견한 버그는 고친 뒤 어떤 검증을 했는지 알려줘
```

나쁜 요청:

```text
/investigate 안 돼
```

좋은 요청:

```text
/investigate `manta list --json`이 빈 배열을 출력하는 원인을 찾아줘.
최근 변경은 task 상태 폴더 이동 로직이야. 원인 확인 전에는 수정하지 마.
```

나쁜 요청:

```text
/review
```

좋은 요청:

```text
/review 현재 manta-repo diff를 머지 전 리뷰해줘.
특히 CLI 출력 계약, exit code, JSON 호환성 위주로 봐줘.
```

좋은 요청에는 보통 세 가지가 들어간다.

- 대상: 어떤 repo, 파일, URL, 명령, 사용자 흐름인지
- 기준: 무엇을 중요하게 봐야 하는지
- 결과: 수정까지 원하는지, 리포트만 원하는지

---

## 5. Manta에서의 사용 예시

### CLI 버그 원인 조사

```text
/investigate `npm test`에서 CLI 출력 계약 테스트가 실패하는 원인을 찾아줘.
stdout/stderr 분리 규칙이 깨졌는지 먼저 확인해줘.
```

이 요청은 바로 수정하기보다 실패 원인을 확인하는 데 초점을 둔다.
원인이 출력 위치인지, exit code인지, fixture 문제인지 분리할 수 있다.

### 웹 UI QA

```text
/qa 로컬 dev server를 띄워서 task 생성, 상태 변경, 상세 보기 흐름을 테스트해줘.
모바일 폭에서도 텍스트 겹침이 없는지 확인하고, 발견한 문제는 고쳐줘.
```

화면이 있는 기능은 실제 브라우저에서 클릭해 봐야 한다.
단순 테스트 통과만으로는 버튼 비활성 상태나 반응형 깨짐을 잡기 어렵다.

### 문서만 안전하게 수정

```text
/freeze manta-doc로 편집 범위를 제한해줘.
그 다음 studies 문서만 정리해줘.
```

Manta workspace에는 루트, `manta-repo`, `manta-doc`가 독립 git repo로 존재한다.
문서 작업 중 코드 repo가 건드려지면 안 되는 경우에는 편집 범위를 제한하는 것이 안전하다.

### PR 전 리뷰

```text
/review 현재 diff를 리뷰해줘.
버그, 회귀 위험, 누락된 테스트만 우선순위로 알려줘.
```

리뷰 요청은 칭찬이나 요약보다 문제 발견에 초점을 둔다.
수정까지 원한다면 리뷰 후 별도로 구현을 요청하는 것이 명확하다.

### 작업 맥락 저장

```text
/context-save 지금까지 확인한 원인, 수정한 파일, 남은 검증 작업을 저장해줘.
```

긴 작업을 중간에 멈출 때 유용하다.
다음 세션에서는 `/context-restore`로 이어갈 수 있다.

---

## 6. skill을 쓰지 않아도 되는 경우

모든 작업에 gstack skill이 필요한 것은 아니다.

예를 들어 다음은 일반 Codex 작업으로 충분할 수 있다.

- 단일 Markdown 문서의 오타 수정
- 작은 타입 이름 변경
- 특정 파일 한두 개를 읽고 요약
- 이미 원인이 명확한 한 줄 버그 수정
- 단순한 명령 출력 확인

skill은 작업을 체계화하는 장점이 있지만,
작은 작업에서는 오히려 절차가 무거울 수 있다.

판단 기준은 단순하다.

> 원인 조사, 브라우저 검증, 리뷰, 배포, 안전 제한처럼 절차가 중요한가?

그렇다면 gstack skill을 쓰는 편이 좋다.
그렇지 않다면 일반 요청으로 충분하다.

---

## 7. 기억할 기본 조합

처음에는 모든 skill을 외우지 말고 아래 조합만 기억해도 된다.

| 하고 싶은 일 | 요청 |
|---|---|
| 버그 원인 찾기 | `/investigate ...` |
| 웹 앱 실제 테스트 | `/qa ...` |
| 수정 없는 QA 리포트 | `/qa-only ...` |
| 머지 전 코드 리뷰 | `/review ...` |
| 설계 계획 검토 | `/plan-eng-review ...` |
| 제품 범위 검토 | `/plan-ceo-review ...` |
| PR 생성 | `/ship ...` |
| 배포와 검증 | `/land-and-deploy ...` |
| 작업 저장 | `/context-save ...` |
| 작업 복원 | `/context-restore ...` |
| 편집 범위 제한 | `/freeze ...` |

Manta 관점에서 gstack skills는 “AI에게 일을 맡기는 버튼 묶음”에 가깝다.
어떤 버튼을 누를지보다 중요한 것은,
작업의 목표와 성공 기준을 명확히 말하는 것이다.
