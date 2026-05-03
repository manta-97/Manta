# 작은 Commander CLI로 출력 계약 이해하기

## 이 챕터의 목표

이 문서는 실제 파일을 만들거나 실행하지 않는다.
대신 TypeScript와 Commander로 작은 `mini-manta` CLI를 만든다고 가정하고,
코드 조각을 읽으면서 CLI 출력 계약을 이해한다.

이전 챕터가 “CLI 출력은 API다”를 설명했다면,
이번 챕터는 그 원칙이 코드에서는 어떤 모양으로 드러나는지 보는 단계다.

중요한 것은 코드를 외우는 것이 아니다.
아래 질문에 답할 수 있으면 된다.

- 어떤 출력이 stdout으로 가야 하는가?
- 어떤 출력이 stderr로 가야 하는가?
- 언제 `process.exitCode = 1`을 설정해야 하는가?
- 사람용 출력과 JSON 출력은 왜 분리되는가?
- Commander는 명령어 구조를 어떻게 표현하는가?

---

## 1. 우리가 만들 작은 CLI

학습용 CLI 이름은 `mini-manta`라고 가정한다.

지원하는 명령어는 세 개뿐이다.

```bash
mini-manta list
mini-manta list --json
mini-manta show task-1
```

그리고 실패 예시도 하나 본다.

```bash
mini-manta show task-999
```

실제 Manta보다 훨씬 작게 보는 이유는 명확하다.
처음부터 파일 시스템, frontmatter, 상태 폴더, 검색까지 모두 넣으면 CLI 계약이 아니라 구현 세부사항에 시선이 뺏긴다.

이번 챕터에서는 고정된 task 데이터가 메모리에 있다고 가정한다.

```ts
type Task = {
  id: string;
  title: string;
  status: 'todo' | 'in-progress' | 'done';
  body: string;
};

const tasks: Task[] = [
  {
    id: 'task-1',
    title: 'CLI 계약 설계',
    status: 'todo',
    body: 'stdout, stderr, exit code를 정리한다.',
  },
  {
    id: 'task-2',
    title: 'init 명령 구현',
    status: 'done',
    body: 'Manta 프로젝트 초기화 명령을 만든다.',
  },
];
```

이 코드에서 봐야 할 점:

- 데이터 저장 방식은 중요하지 않다.
- 지금은 CLI 출력 계약만 보기 위해 배열을 쓴다.
- 실제 Manta에서는 이 데이터가 `@manta/core`와 로컬 파일에서 온다.

---

## 2. Commander 프로그램의 기본 모양

Commander에서는 보통 루트 프로그램을 만들고, 그 아래에 서브커맨드를 붙인다.

```ts
import { Command } from 'commander';

const program = new Command()
  .name('mini-manta')
  .description('Small CLI for learning Manta output contracts')
  .version('0.1.0');

program.parse();
```

이 코드에서 봐야 할 점:

- `.name()`은 help 출력에서 CLI 이름으로 쓰인다.
- `.description()`은 사람이 CLI의 목적을 파악하는 첫 문장이다.
- `.version()`은 `--version` 출력의 기준이 된다.
- `program.parse()`가 실제 command line arguments를 해석한다.

이 상태에서는 아직 명령어가 없다.
그래도 Commander는 기본 help와 version 처리를 제공한다.

```bash
mini-manta --help
mini-manta --version
```

Manta 관점에서 중요한 점은 help가 단순한 덤이 아니라는 것이다.
AI가 사용할 CLI라면 help는 “사용 가능한 계약 목록”에 가깝다.

---

## 3. `list` 명령어: 사람용 출력

먼저 사람이 읽는 `list`를 만든다고 가정한다.

```ts
program
  .command('list')
  .description('List tasks')
  .action(() => {
    console.log('ID       Status        Title');

    for (const task of tasks) {
      console.log(`${task.id}   ${task.status}          ${task.title}`);
    }
  });
```

실행하면 이런 모양을 기대할 수 있다.

```text
ID       Status        Title
task-1   todo          CLI 계약 설계
task-2   done          init 명령 구현
```

이 코드에서 봐야 할 점:

- `console.log()`는 stdout으로 출력한다.
- task 목록은 명령의 결과 데이터이므로 stdout이 맞다.
- 사람이 읽기 좋은 표 형태를 직접 만든 예시다.
- 이 출력은 사람이 보기에는 좋지만, 프로그램이 안정적으로 파싱하기에는 약하다.

여기서 중요한 질문이 생긴다.

> `mini-manta list > tasks.txt`를 실행하면 어떤 내용이 파일에 들어가야 하는가?

정답은 task 목록이다.
경고, 진행 메시지, 디버그 로그가 섞이면 안 된다.

---

## 4. `list --json`: AI와 스크립트용 출력

이번에는 같은 `list` 명령어에 `--json` 옵션을 붙인다.

```ts
program
  .command('list')
  .description('List tasks')
  .option('--json', 'emit machine-readable JSON')
  .action((options: { json?: boolean }) => {
    if (options.json) {
      console.log(
        JSON.stringify(
          {
            version: '1',
            tasks: tasks.map((task) => ({
              id: task.id,
              title: task.title,
              status: task.status,
            })),
          },
          null,
          2,
        ),
      );
      return;
    }

    console.log('ID       Status        Title');

    for (const task of tasks) {
      console.log(`${task.id}   ${task.status}          ${task.title}`);
    }
  });
```

`mini-manta list --json`의 출력은 이렇게 된다.

```json
{
  "version": "1",
  "tasks": [
    {
      "id": "task-1",
      "title": "CLI 계약 설계",
      "status": "todo"
    },
    {
      "id": "task-2",
      "title": "init 명령 구현",
      "status": "done"
    }
  ]
}
```

이 코드에서 봐야 할 점:

- `--json`은 사람용 출력과 기계용 출력을 나누는 스위치다.
- JSON도 stdout으로 나간다.
- JSON 앞뒤에 다른 문장을 출력하면 파싱이 깨진다.
- `version: '1'`은 출력 계약의 버전을 뜻한다.

나쁜 예는 이런 것이다.

```ts
console.log('Found 2 tasks');
console.log(JSON.stringify({ version: '1', tasks }));
```

사람이 보기에는 친절해 보이지만,
`mini-manta list --json | jq '.tasks'` 같은 조합에서는 깨진다.

JSON 모드에서는 stdout에 JSON만 나가야 한다.
안내 메시지가 꼭 필요하다면 stderr를 사용해야 한다.

---

## 5. `show <id>`: 성공하면 본문을 stdout으로 보낸다

이번에는 task 하나를 보여주는 명령어를 본다.

```ts
program
  .command('show')
  .description('Show a task')
  .argument('<id>', 'task id')
  .action((id: string) => {
    const task = tasks.find((item) => item.id === id);

    if (!task) {
      console.error(`Error: TASK_NOT_FOUND`);
      console.error(`Task not found: ${id}.`);
      process.exitCode = 1;
      return;
    }

    console.log(`# ${task.title}`);
    console.log('');
    console.log(`Status: ${task.status}`);
    console.log('');
    console.log(task.body);
  });
```

`mini-manta show task-1`의 출력은 이렇게 볼 수 있다.

```markdown
# CLI 계약 설계

Status: todo

stdout, stderr, exit code를 정리한다.
```

이 코드에서 봐야 할 점:

- task 본문은 명령의 결과 데이터다.
- 그래서 성공 출력은 stdout으로 보낸다.
- 사용자는 `mini-manta show task-1 > task-1.md`처럼 파일로 저장할 수 있다.
- 성공했으므로 `process.exitCode`를 건드리지 않는다.

CLI를 설계할 때는 리다이렉트를 항상 상상해야 한다.

```bash
mini-manta show task-1 > task-1.md
```

이때 `task-1.md` 안에는 Markdown 본문만 들어가야 한다.
`Loading...`, `Done`, `Found task` 같은 메시지가 들어가면 출력 계약이 망가진다.

---

## 6. 실패하면 stderr와 exit code를 사용한다

이번에는 없는 task를 조회하는 경우다.

```bash
mini-manta show task-999
```

위의 `show` 코드는 task가 없으면 이렇게 처리한다.

```ts
if (!task) {
  console.error(`Error: TASK_NOT_FOUND`);
  console.error(`Task not found: ${id}.`);
  process.exitCode = 1;
  return;
}
```

출력은 stderr로 간다.

```text
Error: TASK_NOT_FOUND
Task not found: task-999.
```

이 코드에서 봐야 할 점:

- 오류 메시지는 stdout이 아니라 stderr로 보낸다.
- `process.exitCode = 1`은 “이 명령은 실패했다”는 신호다.
- `return`으로 이후 성공 출력이 실행되지 않게 막는다.

왜 `throw new Error()`나 `process.exit(1)`을 바로 쓰지 않는지도 중요하다.

`process.exitCode = 1`은 현재 명령이 실패했다는 신호만 설정한다.
프로세스를 즉시 끊지 않기 때문에 테스트나 정리 로직과 함께 쓰기 좋다.

실제 CLI에서는 상황에 따라 `throw`를 쓰거나 상위 레이어에서 오류를 모아 처리할 수 있다.
하지만 학습 단계에서는 `console.error()`와 `process.exitCode = 1`의 역할을 분리해서 보는 것이 먼저다.

---

## 7. usage error는 Commander가 먼저 잡는다

`show` 명령어는 `<id>` 인자를 필수로 받는다.

```ts
.argument('<id>', 'task id')
```

그래서 사용자가 이렇게 입력하면:

```bash
mini-manta show
```

명령어 자체는 존재하지만 필수 입력이 빠진 상태다.
이건 `TASK_NOT_FOUND`가 아니다.
아직 어떤 task를 찾을지조차 정해지지 않았기 때문이다.

이런 오류는 usage error에 가깝다.

개념적으로는 이렇게 구분하면 된다.

| 입력 | 분류 | 이유 |
|---|---|---|
| `mini-manta xyz` | unknown command | 그런 명령어가 없음 |
| `mini-manta show` | usage error | 명령어는 있지만 필수 인자 누락 |
| `mini-manta show task-999` | runtime failure | 입력은 맞지만 실행 중 대상 없음 |

이 구분은 AI에게 중요하다.

- unknown command면 `mini-manta help`를 봐야 한다.
- usage error면 `mini-manta help show`를 봐야 한다.
- runtime failure면 task 목록이나 파일 상태를 확인해야 한다.

---

## 8. stderr가 왜 필요한지 리다이렉트로 이해하기

stdout과 stderr의 차이는 리다이렉트를 생각하면 바로 이해된다.

성공 케이스:

```bash
mini-manta list --json > tasks.json
```

기대:

- `tasks.json`에는 JSON만 들어간다.
- 터미널에는 별도 메시지가 없어도 된다.
- exit code는 0이다.

실패 케이스:

```bash
mini-manta show task-999 > task.md
```

기대:

- `task.md`에는 성공 데이터가 들어가지 않는다.
- 오류 메시지는 터미널에 보인다.
- exit code는 1이다.

stderr까지 파일로 따로 보고 싶다면 이렇게 생각할 수 있다.

```bash
mini-manta show task-999 > task.md 2> error.log
```

기대:

- `task.md`는 비어 있다.
- `error.log`에는 오류 메시지가 들어간다.

이 구조가 있어야 CLI를 다른 도구와 안전하게 연결할 수 있다.

---

## 9. exit code는 스크립트의 if 문이 된다

exit code는 사람이 눈으로 보는 출력보다 더 직접적인 성공/실패 신호다.

셸에서는 이런 식으로 쓸 수 있다.

```bash
if mini-manta show task-1 > task.md; then
  echo "task exported"
else
  echo "task export failed"
fi
```

이때 셸은 출력 문구를 읽지 않는다.
명령의 exit code만 본다.

그래서 `mini-manta show task-999`가 오류 메시지를 잘 출력하더라도,
exit code가 0이면 스크립트는 성공으로 오해한다.

Manta에서 AI가 CLI를 사용할 때도 비슷하다.
AI는 stderr 문구를 읽을 수 있지만, 안정적인 분기 기준은 exit code와 error code다.

---

## 10. 전체 구조를 한 번에 보기

위 예시를 한 파일처럼 모으면 대략 이런 구조가 된다.

```ts
import { Command } from 'commander';

type Task = {
  id: string;
  title: string;
  status: 'todo' | 'in-progress' | 'done';
  body: string;
};

const tasks: Task[] = [
  {
    id: 'task-1',
    title: 'CLI 계약 설계',
    status: 'todo',
    body: 'stdout, stderr, exit code를 정리한다.',
  },
  {
    id: 'task-2',
    title: 'init 명령 구현',
    status: 'done',
    body: 'Manta 프로젝트 초기화 명령을 만든다.',
  },
];

const program = new Command()
  .name('mini-manta')
  .description('Small CLI for learning Manta output contracts')
  .version('0.1.0');

program
  .command('list')
  .description('List tasks')
  .option('--json', 'emit machine-readable JSON')
  .action((options: { json?: boolean }) => {
    if (options.json) {
      console.log(
        JSON.stringify(
          {
            version: '1',
            tasks: tasks.map((task) => ({
              id: task.id,
              title: task.title,
              status: task.status,
            })),
          },
          null,
          2,
        ),
      );
      return;
    }

    console.log('ID       Status        Title');

    for (const task of tasks) {
      console.log(`${task.id}   ${task.status}          ${task.title}`);
    }
  });

program
  .command('show')
  .description('Show a task')
  .argument('<id>', 'task id')
  .action((id: string) => {
    const task = tasks.find((item) => item.id === id);

    if (!task) {
      console.error(`Error: TASK_NOT_FOUND`);
      console.error(`Task not found: ${id}.`);
      process.exitCode = 1;
      return;
    }

    console.log(`# ${task.title}`);
    console.log('');
    console.log(`Status: ${task.status}`);
    console.log('');
    console.log(task.body);
  });

program.parse();
```

이 전체 코드에서 봐야 할 핵심은 네 가지다.

- Commander는 명령어와 옵션의 구조를 만든다.
- 성공 데이터는 `console.log()`로 stdout에 보낸다.
- 오류 메시지는 `console.error()`로 stderr에 보낸다.
- 실패한 명령은 `process.exitCode = 1`로 표시한다.

---

## 11. Manta 구현으로 연결하기

실제 Manta에서는 위 코드와 다른 점이 많다.

- task 데이터는 배열이 아니라 로컬 파일에서 온다.
- 파일 읽기와 쓰기는 `@manta/core`가 담당한다.
- CLI는 `@manta/core` 위의 adapter다.
- 오류 종류는 core의 Result나 error code와 연결된다.
- help 출력은 전체 명령어 계약과 맞아야 한다.

하지만 기본 구조는 같다.

```text
사용자 입력
  → Commander가 명령어/옵션/인자 파싱
  → CLI command action 실행
  → core 호출
  → 성공이면 stdout
  → 실패이면 stderr + exitCode
```

이 흐름을 이해하면 `manta add`, `manta list`, `manta show`를 만들 때 무엇을 고민해야 하는지 분명해진다.

구현 세부사항보다 먼저 정해야 하는 것은 출력 계약이다.

- `manta list`의 기본 출력은 표인가?
- `manta list --json`의 JSON 필드는 무엇인가?
- task가 없을 때는 성공인가 실패인가?
- task id가 잘못됐을 때와 task가 없을 때를 구분할 것인가?
- 오류 메시지는 어떤 error code를 포함할 것인가?

이 질문에 답한 뒤 코드를 쓰면 CLI가 훨씬 안정적이 된다.

---

## 정리

TypeScript와 Commander를 배울 때 처음부터 모든 기능을 넣을 필요는 없다.
작은 `list`, `show` 명령어만으로도 CLI 계약의 핵심을 충분히 배울 수 있다.

이번 챕터에서 기억할 것은 다음이다.

- `.command()`는 사용자가 호출할 동작을 만든다.
- `.option()`은 같은 동작의 출력 방식이나 조건을 바꾼다.
- `.argument()`는 명령이 반드시 받아야 하는 입력을 표현한다.
- `console.log()`는 stdout이다.
- `console.error()`는 stderr다.
- `process.exitCode = 1`은 실패 신호다.
- JSON 모드의 stdout에는 JSON만 있어야 한다.

Manta CLI는 이 작은 원칙들의 조합으로 커진다.
처음부터 복잡한 구현을 외우기보다, 출력 계약을 지키는 감각을 먼저 잡는 것이 중요하다.
