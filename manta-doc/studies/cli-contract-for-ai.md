# AI가 사용할 수 있는 CLI 계약 설계

## 이 챕터의 목표

Manta에서 CLI는 단순한 개발자용 보조 도구가 아니다.
CLI는 사람, 스크립트, AI 에이전트가 같은 규칙으로 작업을 조작하기 위한 표준 인터페이스다.

이 챕터의 목표는 `manta add`, `manta list`, `manta show` 같은 명령어를 만들기 전에,
CLI 출력과 오류를 어떤 기준으로 설계해야 하는지 이해하는 것이다.

핵심 질문은 하나다.

> AI가 이 CLI를 보고 다음 행동을 안정적으로 결정할 수 있는가?

사람은 애매한 출력도 맥락으로 해석할 수 있다.
하지만 AI와 스크립트는 출력 위치, exit code, 오류 문구, JSON 구조가 흔들리면 쉽게 잘못된 분기를 탄다.
그래서 Manta의 CLI 출력은 사용성 요소이면서 동시에 제품 API다.

---

## 1. CLI 출력은 API다

웹 API에서는 응답 body, status code, error code를 신중하게 설계한다.
CLI도 다르지 않다.

CLI에는 다음과 같은 인터페이스가 있다.

| 요소 | 역할 |
|---|---|
| 명령어 이름 | 사용자가 요청하는 동작 |
| arguments / flags | 동작에 필요한 입력 |
| stdout | 명령의 결과 데이터 |
| stderr | 진행 메시지, 경고, 오류 |
| exit code | 성공/실패의 안정 신호 |
| help | 사용 가능한 계약의 문서 |
| JSON output | AI와 스크립트를 위한 구조화된 출력 |

Manta에서는 이 계약이 특히 중요하다.
Manta가 지향하는 사용자는 사람만이 아니라 AI도 포함하기 때문이다.

예를 들어 `manta list`가 어떤 때는 표를 출력하고, 어떤 때는 경고를 stdout에 섞고, 어떤 때는 실패했는데 exit code 0을 반환하면 AI는 안정적으로 사용할 수 없다.

좋은 CLI는 다음 질문에 답할 수 있어야 한다.

- 성공했는가?
- 실패했다면 어떤 종류의 실패인가?
- 사람이 읽어야 할 메시지와 프로그램이 처리할 데이터가 분리되어 있는가?
- help만 읽어도 올바른 명령어를 만들 수 있는가?

---

## 2. Help는 첫 번째 사용자 인터페이스다

CLI에는 화면 버튼이 없다.
사용자가 무엇을 할 수 있는지 배우는 첫 번째 경로는 help다.

Manta의 help는 최소한 다음 경로에서 일관되게 동작해야 한다.

```bash
manta help
manta --help
manta help init
manta init --help
```

중요한 점은 “대충 비슷한 내용”이 아니라 같은 계약을 공유해야 한다는 것이다.
예를 들어 `manta help init`과 `manta init --help`가 서로 다른 설명을 보여주면, 어느 쪽이 진짜 계약인지 알 수 없다.

좋은 command help에는 다음이 들어간다.

| 항목 | 예시 |
|---|---|
| 명령어 설명 | `Initialize a Manta project` |
| 사용법 | `manta init [path]` |
| 인자 설명 | `path: task directory name` |
| 옵션 설명 | `--json: emit machine-readable JSON` |
| 예시 | `manta init docs` |

Manta 관점에서 help는 사람뿐 아니라 AI를 위한 학습 데이터다.
AI가 `manta help`를 보고 사용 가능한 명령어를 파악하고, `manta help add`를 보고 정확한 인자를 넣을 수 있어야 한다.

---

## 3. stdout과 stderr를 분리한다

CLI에서 가장 중요한 출력 규칙은 이것이다.

> stdout은 결과 데이터, stderr는 메시지다.

예를 들어 `manta list --json`이 task 목록 JSON을 stdout으로 출력한다면,
경고나 진행 메시지는 stdout에 섞이면 안 된다.

나쁜 예:

```text
Reading tasks...
[
  { "id": "task-1", "title": "CLI 설계" }
]
```

이 출력은 사람이 보기에는 괜찮아 보일 수 있다.
하지만 프로그램이 JSON으로 파싱하려고 하면 첫 줄 때문에 깨진다.

좋은 예:

```text
# stderr
Reading tasks...

# stdout
[
  { "id": "task-1", "title": "CLI 설계" }
]
```

Manta에서는 다음 기준을 기본값으로 둔다.

| 출력 | 보낼 곳 |
|---|---|
| task 목록 데이터 | stdout |
| task 상세 Markdown | stdout |
| JSON 응답 | stdout |
| 경고 | stderr |
| 오류 | stderr |
| 진행 상태 | stderr |
| 사용자가 다음에 할 수 있는 안내 | 주로 stderr |

이 규칙을 지키면 사용자는 다음처럼 안전하게 조합할 수 있다.

```bash
manta list --json > tasks.json
manta show task-1 > task-1.md
manta search auth --json | jq '.tasks[].id'
```

---

## 4. exit code는 AI의 분기 신호다

사람은 오류 문구를 읽고 상황을 이해할 수 있다.
AI와 스크립트는 exit code를 먼저 본다.

기본 원칙은 단순하다.

| exit code | 의미 |
|---|---|
| `0` | 명령이 성공했다 |
| `1` | 명령이 실패했다 |

처음부터 많은 exit code를 만들 필요는 없다.
하지만 성공과 실패는 반드시 구분되어야 한다.

예를 들어 검색 결과가 없는 상황은 실패가 아닐 수 있다.

```bash
manta search "oauth"
```

검색이 정상적으로 실행됐고 결과만 없다면 exit code는 0이 자연스럽다.
반대로 task id 형식이 잘못됐거나 파일을 읽을 수 없다면 실패다.

```bash
manta show abc
manta show task-999
```

이런 경우는 exit code 1이어야 한다.

Manta에서 중요한 판단 기준은 이것이다.

> 명령이 요청한 동작을 정상적으로 수행했는가?

결과가 비어 있는 것과 명령이 실패한 것은 다르다.

---

## 5. 오류는 종류를 나눈다

CLI 오류를 모두 “에러가 났다”로 뭉개면 AI가 복구 행동을 선택하기 어렵다.
Manta에서는 최소한 다음 세 가지를 구분하는 것이 좋다.

| 분류 | 예시 | 의미 |
|---|---|---|
| unknown command | `manta xyz` | 그런 명령어가 없다 |
| usage error | `manta show` | 명령어는 있지만 입력이 잘못됐다 |
| runtime failure | `manta show task-1` 중 파일 읽기 실패 | 입력은 맞지만 실행 중 실패했다 |

각 오류는 사용자가 다음 행동을 알 수 있어야 한다.

unknown command 예시:

```text
Error: UNKNOWN_COMMAND
Unknown command: xyz.
Run `manta help` to see available commands.
```

usage error 예시:

```text
Error: USAGE_ERROR
Missing required argument: id.
Run `manta help show` for usage.
```

runtime failure 예시:

```text
Error: TASK_NOT_FOUND
Task not found: task-999.
Run `manta list` to see available tasks.
```

여기서 중요한 것은 문구의 화려함이 아니다.
오류의 종류가 안정적으로 드러나야 한다는 점이다.

---

## 6. 사람용 출력과 AI용 JSON을 나눈다

사람에게 좋은 출력과 프로그램에게 좋은 출력은 다르다.

사람에게는 이런 출력이 읽기 좋다.

```text
ID       Status        Title
task-1   todo          CLI 계약 설계
task-2   done          init 명령 구현
```

AI와 스크립트에게는 JSON이 좋다.

```json
{
  "tasks": [
    { "id": "task-1", "status": "todo", "title": "CLI 계약 설계" },
    { "id": "task-2", "status": "done", "title": "init 명령 구현" }
  ]
}
```

그래서 Manta 명령어는 가능하면 두 출력을 분리해서 생각한다.

```bash
manta list
manta list --json
```

주의할 점은 JSON 출력도 제품 계약이라는 것이다.
한 번 AI와 스크립트가 의존하기 시작하면 필드 이름을 쉽게 바꿀 수 없다.

따라서 JSON에는 처음부터 너무 많은 것을 넣기보다,
실제로 필요한 안정 필드부터 작게 시작하는 편이 낫다.

예를 들어 `manta list --json` v0는 다음 정도면 충분할 수 있다.

```json
{
  "version": "1",
  "tasks": [
    {
      "id": "task-1",
      "title": "CLI 계약 설계",
      "status": "todo",
      "path": "tasks/todo/task-1.md"
    }
  ]
}
```

`version`을 두면 나중에 출력 계약을 바꿔야 할 때 마이그레이션 경로를 만들 수 있다.

---

## 7. Manta에 적용해보기

아래는 Manta Phase 1 명령어를 설계할 때 사용할 수 있는 판단 기준이다.

### `manta add "제목"`

성공 시:

- task 파일을 `tasks/todo/`에 만든다.
- 생성된 task id를 stdout에 출력한다.
- exit code 0을 반환한다.

예시:

```text
Created task-3
```

AI가 후속 명령을 이어가기 좋게 하려면 JSON 출력도 고려할 수 있다.

```json
{
  "version": "1",
  "task": {
    "id": "task-3",
    "title": "제목",
    "status": "todo",
    "path": "tasks/todo/task-3.md"
  }
}
```

### `manta list`

성공 시:

- task 목록을 stdout에 출력한다.
- task가 하나도 없어도 실패가 아니다.
- 결과 없음 메시지는 stdout에 출력해도 된다. 단, `--json`에서는 빈 배열을 출력한다.

예시:

```text
No tasks found.
```

JSON 예시:

```json
{
  "version": "1",
  "tasks": []
}
```

### `manta show task-1`

성공 시:

- task Markdown 본문을 stdout에 출력한다.
- 파일을 다른 명령어로 넘길 수 있어야 한다.

예시:

```bash
manta show task-1 > task-1.md
```

실패 시:

- task가 없으면 stderr에 오류를 출력한다.
- exit code 1을 반환한다.

### `manta start task-1` / `manta done task-1`

성공 시:

- 파일을 목표 상태 폴더로 이동한다.
- 이미 목표 상태라면 no-op으로 처리하고 exit code 0을 반환한다.
- 사용자가 이해할 수 있는 짧은 메시지를 출력한다.

실패 시:

- task가 없으면 `TASK_NOT_FOUND`로 실패한다.
- 같은 id가 여러 상태 폴더에 있으면 `DUPLICATE_TASK_ID`로 실패한다.

---

## 8. 학습 과제

아래 질문에 답하면서 CLI 계약을 직접 설계해본다.

### 과제 1. stdout/stderr 분리

다음 상황에서 무엇을 stdout에 보내고 무엇을 stderr에 보낼지 정한다.

- `manta list --json`
- `manta search "auth"`
- `manta show task-1`
- `manta done task-999`

판단 기준:

- 다른 프로그램이 파싱해야 하는 데이터인가?
- 사용자에게 보여주는 안내인가?
- 리다이렉트했을 때 파일에 들어가도 되는 내용인가?

### 과제 2. 오류 분류

다음 케이스를 unknown command, usage error, runtime failure 중 하나로 분류한다.

```bash
manta xyz
manta show
manta show abc
manta show task-999
manta init /root/no-permission
```

추가로 각 케이스의 exit code를 정한다.

### 과제 3. JSON 계약 만들기

`manta list --json`의 v0 응답을 직접 설계한다.

필수로 생각해볼 항목:

- top-level `version`을 둘 것인가?
- task의 `status`는 문자열로 둘 것인가?
- `path`를 포함할 것인가?
- 결과가 없을 때 `tasks: []`로 둘 것인가, 메시지를 둘 것인가?

### 과제 4. help 계약 점검

다음 네 명령어의 출력이 어떤 관계여야 하는지 정한다.

```bash
manta help
manta --help
manta help init
manta init --help
```

Manta에서는 `manta help init`과 `manta init --help`가 같은 내용을 보여주는 것이 좋다.
두 출력이 달라지면 사람이든 AI든 어느 쪽을 기준으로 삼아야 할지 헷갈리기 때문이다.

---

## 정리

Manta CLI를 만들 때 중요한 것은 명령어가 “동작한다”는 사실만이 아니다.
그 명령어를 사람, 스크립트, AI가 같은 방식으로 해석할 수 있어야 한다.

이 챕터의 핵심 원칙은 네 가지다.

- help는 CLI의 첫 번째 UI다.
- stdout은 데이터, stderr는 메시지다.
- exit code는 AI와 스크립트의 분기 신호다.
- JSON 출력은 작고 안정적인 제품 계약이다.

이 원칙이 잡히면 Electron GUI를 만들 때도 구조가 단순해진다.
GUI는 CLI stdout을 파싱하지 않고 `@manta/core`를 직접 호출하겠지만,
CLI에서 정리한 성공/실패 모델과 데이터 계약은 그대로 GUI 설계의 기준이 된다.
