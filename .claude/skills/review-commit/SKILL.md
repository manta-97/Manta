---
name: review-commit
description: |
  staged 된 코드(git diff --cached)를 리뷰하고 커밋 메시지를 작성합니다.
  "/review-commit", "커밋 리뷰", "staged 리뷰", "커밋 메시지 써줘" 요청에 이 스킬을 사용한다.
  staged 변경사항이 있을 때 코드 품질 리뷰 + 커밋 메시지 생성을 한 번에 처리한다.
---

**인자**: $ARGUMENTS

---

## 실행 지침

staged 된 변경사항을 리뷰하고 커밋 메시지 후보를 작성하세요.

> **⚠️ 절대로 `git commit`을 실행하지 마세요.** 커밋은 사용자가 직접 합니다. 이 스킬은 메시지 후보만 생성합니다.

### 1. 정보 수집

다음 명령을 **병렬로** 실행하세요:

```bash
git -C manta-repo diff --cached          # staged 변경사항
git -C manta-repo diff --cached --stat   # 변경 파일 요약
git -C manta-repo branch --show-current  # 현재 브랜치명
```

staged 변경사항이 없으면 사용자에게 알리고 종료하세요.

### 2. 코드 리뷰

diff를 분석해 다음을 확인하세요:

- **로직 오류**: 조건 분기 실수, 경계값 처리 누락, 반전된 조건
- **타입/널 안전성**: None/null 처리 누락, 타입 불일치
- **네이밍**: 모호한 이름(`data`, `result`, `info`), 과도한 축약
- **부수 효과**: 의도치 않은 동작 변경, 다른 호출자에 영향
- **불필요한 코드**: 사용하지 않는 import, 변수, 디버그 코드

리뷰 결과가 있으면 항목별로 간결하게 보고하세요. 문제가 없으면 "리뷰 이슈 없음"으로 넘어갑니다.

### 3. 커밋 메시지 작성

#### 브랜치명에서 Jira 티켓 추출

브랜치명에서 `<prefix>-<number>` 패턴을 찾아 대문자로 변환하세요.

| 브랜치명 | 추출 결과 |
|---|---|
| `task-12-update-test` | `TASK-12` |
| `feat/qa-856-add-login` | `QA-856` |
| `main`, `develop` | (없음 — prefix 생략) |

추출 규칙:
- 영문자+숫자 패턴(`word-number`)을 브랜치명 앞부분에서 찾는다
- 대문자로 변환: `task-12` → `TASK-12`
- 패턴이 없으면 티켓 prefix 없이 커밋 메시지만 작성

#### 커밋 메시지 형식

```
TASK-12: Add validation for user input on refund form
```

- 티켓 ID가 있으면: `TICKET-ID: message`
- 티켓 ID가 없으면: `message`
- 메시지는 **영어**로 작성
- 첫 글자 대문자, 마침표 없음
- 변경의 "무엇을"보다 "왜"에 초점
- 50자 이내 권장, 필요하면 본문 추가

#### 커밋 메시지 예시

```
TASK-12: Add input validation for worker refund amount
```

```
QA-856: Fix off-by-one error in pagination offset calculation
```

```
Update TypeScript config to enable strict null checks
```

### 4. 출력 형식

커밋 메시지는 **3개 후보**를 제시하여 사용자가 선택하도록 합니다.

```markdown
## Code Review

(리뷰 결과 또는 "리뷰 이슈 없음")

## Commit Message 후보

1. `TASK-12: Add input validation for worker refund amount`
2. `TASK-12: Prevent invalid refund amounts from being submitted`
3. `TASK-12: Validate refund amount before processing request`
```