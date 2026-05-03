# Task-10: `manta help [command]` 구현

## 배경

지금 Manta CLI에는 `manta init`만 실제로 존재한다. Phase 1의 나머지 커맨드 — `add`, `list`, `show`, `start`, `done`, `edit` — 는 `cli-design.md`에만 명세가 있고 코드에는 없다.

이런 상태에서 AI가 Manta를 처음 만나면 무엇을 할까? `manta --help`를 친다. 그럼 Commander.js가 자동 생성한 영문 한 줄짜리 도움말이 나온다. AI는 "이 시스템으로 할 수 있는 일"을 알 방법이 없고, 결국 파일을 직접 뒤지거나 엉뚱한 워크어라운드 스크립트를 짜게 된다. Manifesto의 "AI는 사용자다" 원칙이 바로 여기서 깨진다.

`cli-design.md`가 `manta help`를 "AI가 명령어 체계를 학습하는 진입점"이라고 적은 이유는 이 지점이다. 그런데 `help`가 보여주는 것은 "앞으로 만들 계획"이 아니라 **지금 이 순간 실제로 실행 가능한 커맨드의 계약**이다. 커맨드가 추가될 때마다 help가 함께 자란다. 로드맵은 별도 문서의 일이다.

**목표**: `manta help`, `manta help <command>`, Commander의 `--help` 경로들이 모두 같은 help 렌더러를 통과하도록 만든다. 이번 태스크는 help prose/JSON 출력과 help path 계약을 정리하는 데 집중한다. CLI 전반의 unknown command / usage error / runtime error 정책 통일은 다음 태스크(`task-11`)로 분리한다.

## 동작

사람과 AI가 같은 경로로 동일한 정보를 볼 수 있어야 한다.

| 입력 | 출력 |
|---|---|
| `manta help` | 구현된 커맨드 목록 + footer |
| `manta --help` | 위와 동일 |
| `manta help init` | `init` 상세 (usage, args, examples) |
| `manta init --help` | 위와 동일 |
| `manta help help` | `help` 상세 prose |
| `manta help --help` | 위와 동일 |
| `manta help --json` | overview JSON |
| `manta help init --json` | `init` 상세 JSON |
| `manta help help --json` | `help` 상세 JSON |
| `manta help xyz` | stderr에 unknown 메시지, exit 1 |

현재 구현된 커맨드는 `init`과 `help` 두 개뿐이고, CLI help registry에도 두 개만 존재한다. 아직 구현되지 않은 커맨드는 help에 나타나지 않는다 — 보이는 것은 실행된다는 계약이다. 앞으로의 계획이 궁금하면 `docs/roadmap.md`를 본다.

(`manta init --help --json`은 이번 스코프에서 제외한다. Commander의 `configureHelp.formatHelp` 훅은 string을 반환하는 경로라 JSON 전달이 깔끔하지 않다. 필요해지면 후속 태스크로 분리한다.)

## 접근

큰 그림은 네 조각이다.

1. **CLI help registry** — 구현된 커맨드의 help 스펙을 `CommandHelpEntry[]`로 선언. `manta-repo/packages/cli/src/help/command-registry.ts`에 배열 하나. 이 레지스트리가 **source of truth**다.
2. **Commander 어댑터** — `applyHelpEntryToCommand(command, entry)`가 레지스트리의 `summary`, `args`, `options`, 필요 시 `usage`를 Commander 정의에 투영한다. 레지스트리와 Commander 정의가 따로 놀 가능성을 원천 차단한다.
3. **순수 렌더러** — `renderOverview`, `renderCommand`는 prose를, `toOverviewJson`, `toCommandJson`은 JSON 구조를 반환한다. prose 쪽은 chalk만 의존, JSON 쪽은 의존 없음. 모두 순수 함수.
4. **Commander 훅 연결** — 루트 `program.configureHelp({ formatHelp })`와 각 서브커맨드의 `.configureHelp(...)`가 prose 렌더러를 호출한다. `manta help` 커맨드 자체는 `--json` 옵션을 보고 prose/JSON을 선택한다.

이 네 조각이 모두 `manta-repo/packages/cli` 안에 있다. `@manta/core`는 건드리지 않는다.

## 설계 결정

### 왜 레지스트리를 source of truth로 두는가

레지스트리와 Commander 정의가 독립적으로 존재하면 둘 사이의 드리프트가 필연이다. `init`의 인자가 바뀌었는데 레지스트리에만 반영되면 help는 거짓말을 하고, Commander에만 반영되면 help가 뒤처진다. 매니페스토 "AI는 사용자다"가 깨지는 전형적 지점이 여기다.

그래서 레지스트리가 주(主)다. `applyHelpEntryToCommand(command, entry)`가 레지스트리의 메타를 Commander에 주입한다. 커맨드 구현자는 엔트리를 추가하고 어댑터로 Commander에 연결한 뒤 `.action()`만 직접 붙인다. 두 개의 진실이 애초에 존재하지 않는다.

이번 태스크에서 어댑터가 다루는 범위는 최소 `summary`, `args`, `options`이고, Commander 정의와 help prose/JSON 사이에서 `usage`까지 같은 계약을 유지할 수 있도록 설계한다. 새 커맨드가 추가되면 같은 패턴이 그대로 확장된다.

### 왜 레지스트리를 `packages/cli`에 두는가

`@manta/core`의 "zero runtime dependencies" 계약을 유지하기 위해서다. 레지스트리에는 `"manta init ..."` 같은 CLI 전용 문자열이 가득해서 core에 두면 성격이 맞지 않는다.

언젠가 GUI가 생기면 **shared capability(이름·시맨틱)** 와 **CLI-specific help copy(`manta init` 같은 usage 문자열, `$`-prefixed 예시)** 를 분리하게 될 것이다. 지금 레지스트리에 들어있는 것은 후자에 가깝다. 전자는 그 시점에 core로 추출된다. 지금은 CLI 단일 소비자뿐이라 그 선을 미리 긋지 않는다.

타입 이름 `CommandHelpEntry`는 그 의도를 반영한다 — CLI help 전용이지, 공유 capability 메타가 아니다.

### 왜 구현된 커맨드만 help에 보이는가

"AI는 사용자다"를 제대로 지키려면 help가 **실행 가능한 계약**이어야 한다. help에 `manta add`가 보이는데 실행하면 `unknown command`가 나면, AI는 그다음 혼란스러운 결정을 내린다 — 파일을 직접 뒤지거나, help 자체를 신뢰하지 않게 된다. 둘 다 최악이다.

그래서 레지스트리에는 현재 구현된 `init`, `help` 두 개만 있다. 커맨드가 추가되면 그 태스크에서 엔트리가 함께 들어온다. "앞으로 뭘 만들 계획인가" — 로드맵은 help의 책무가 아니라 `docs/roadmap.md`의 것이다.

### 왜 JSON을 처음부터 제공하는가

AI가 주 사용자라면 prose 포맷은 취약하다. 섹션 헤더 하나, 공백 하나 바뀌어도 AI 파서가 깨질 수 있다. 사람이 읽는 출력은 바뀔 수 있어도, AI가 분기하는 인터페이스는 더 안정적이어야 한다.

그래서 `manta help --json`, `manta help <cmd> --json` 두 경로를 처음부터 넣는다. 그리고 top-level에 `kind`와 `version`을 둬서 소비자가 응답 종류와 포맷 버전을 먼저 판별할 수 있게 한다.

### 왜 Commander builtin을 완전 교체하지 않고 `configureHelp`를 쓰는가

Commander는 `configureHelp({ formatHelp })` 훅을 표준으로 제공한다. 완전 교체하려면 `helpOption(false)` + 모든 `--help` 수동 파싱 + 경로별 분기가 필요해서 엔트리 포인트가 지저분해진다. 훅 하나만 덮으면 Commander의 인자 라우팅은 그대로 쓰면서 출력만 우리 것으로 바뀐다.

다만 주의할 점이 있다. 루트 `program.configureHelp(...)`만으로는 `manta init --help` 같은 서브커맨드 help가 커버되지 않는다. 각 서브커맨드에도 `.configureHelp(...)`를 개별 적용해야 `manta help init`과 `manta init --help`를 같은 렌더러로 맞출 수 있다.

### 왜 `effect`를 뺐는가

초안은 `CommandExample`에 `{ input, effect }`를 두었다. `effect` — "그 명령을 실행하면 어떤 일이 일어나는지"를 서술한 prose — 는 매력적이지만 **검증되지 않는다**. 코드가 바뀌면 help는 조용히 거짓말을 시작한다. AI가 그 prose를 신뢰한다면 치명적이다.

그래서 `input`만 남기고 `effect`는 뺐다. `input`은 파싱 테스트로 검증하고, 실제 실행 결과는 대표 example만 integration test로 검증한다. prose로 주장하는 대신 테스트로 증명하는 쪽이 옳다.

### 왜 `notes` 필드를 스키마에서 뺐는가

구현된 두 커맨드를 스케치해봤을 때 `notes`가 꼭 필요한 케이스가 없었다. 실제 필요가 확인된 필드만 스키마에 둔다.

### 스키마 확장은 나중

옵션·인자 스키마의 확장(`choices`, `default`, negated 플래그 등)은 실제로 필요한 커맨드가 등장하는 시점에 다룬다. 현재 Phase 1의 `init`, `help`는 최소 스키마로 충분하다.

## CLI help registry 스키마

```ts
// manta-repo/packages/cli/src/help/types.ts
// CLI help 전용. 공유 capability 메타가 아니다.

export interface CommandArg {
  name: string;
  required: boolean;
  description: string;
}

export interface CommandOption {
  flag: string;
  description: string;
}

export interface CommandExample {
  input: string;
}

export interface CommandHelpEntry {
  name: string;
  summary: string;
  usage: string;
  args: CommandArg[];
  options: CommandOption[];
  examples: CommandExample[];
}
```

엔트리 예시:

```ts
{
  name: 'init',
  summary: 'Initialize a Manta project in the current directory',
  usage: 'manta init [path]',
  args: [{ name: 'path', required: false, description: 'task directory name (default: manta)' }],
  options: [],
  examples: [
    { input: 'manta init' },
    { input: 'manta init docs' },
  ],
}
```

레지스트리에는 현재 `init`, `help` 두 엔트리만 존재한다. 새 커맨드가 구현되는 태스크마다 엔트리가 함께 들어온다.

## 렌더러 시그니처

```ts
// manta-repo/packages/cli/src/help/render-help.ts
export function renderOverview(entries: readonly CommandHelpEntry[]): string;
export function renderCommand(entry: CommandHelpEntry): string;
export function toOverviewJson(entries: readonly CommandHelpEntry[]): OverviewJson;
export function toCommandJson(entry: CommandHelpEntry): CommandJson;
```

prose 렌더러(`renderOverview`, `renderCommand`)는 chalk만 의존하고 부작용 없음. JSON 렌더러(`toOverviewJson`, `toCommandJson`)는 의존 없는 순수 함수.

`renderOverview`는 상단 타이틀 → USAGE → COMMANDS 테이블 → footer.
`renderCommand`는 상단 타이틀 → USAGE → ARGUMENTS → OPTIONS(비어있으면 생략) → EXAMPLES.

권장 top-level JSON shape:

```ts
type OverviewJson = {
  kind: 'overview';
  version: '1';
  commands: CommandHelpEntry[];
};

type CommandJson = {
  kind: 'command';
  version: '1';
  command: CommandHelpEntry;
};
```

## 출력 모양

### `manta help`

```
Manta — File-based task management for humans and AI

USAGE
  manta <command> [args]

COMMANDS
  init          Initialize a Manta project in the current directory
  help          Show command list or details

Run `manta help <command>` for details.

Tasks live under <task-dir>/tasks/ (default: manta/tasks/).
Status model: todo → in-progress → done.
```

### `manta help init`

```
manta init — Initialize a Manta project in the current directory

USAGE
  manta init [path]

ARGUMENTS
  path          task directory name (default: manta)

EXAMPLES
  $ manta init
  $ manta init docs
```

### `manta help --json`

```json
{
  "kind": "overview",
  "version": "1",
  "commands": [
    {
      "name": "init",
      "summary": "Initialize a Manta project in the current directory",
      "usage": "manta init [path]",
      "args": [
        { "name": "path", "required": false, "description": "task directory name (default: manta)" }
      ],
      "options": [],
      "examples": [
        { "input": "manta init" },
        { "input": "manta init docs" }
      ]
    },
    { "name": "help", "...": "..." }
  ]
}
```

`manta help init --json`과 `manta help help --json`은 아래 shape를 따른다.

```json
{
  "kind": "command",
  "version": "1",
  "command": {
    "name": "init",
    "...": "..."
  }
}
```

`version: "1"`은 help JSON 포맷 버전이다. 포맷이 바뀌면 이 숫자를 올리고, AI 클라이언트가 자신이 이해하는 버전을 확인할 수 있게 한다. Manta 바이너리 버전과는 별개다.

## Exit code 메모

이번 태스크에서 확정하는 exit code 계약은 help 범위로 한정한다.

- `manta help`, `manta help init`, `manta help --json` → 0
- `manta help xyz` → 1

루트 unknown command (`manta xyz`), usage error, runtime error(`manta init` 실패 포함)를 CLI 전반에서 어떻게 통일할지는 다음 태스크(`task-11`)에서 별도로 정리한다.

## 환경 가정

터미널은 UTF-8 인코딩을 가정한다. 출력에 `—` em-dash 등 비 ASCII 문자를 사용한다. CP949 등 비 UTF-8 환경은 지원 대상이 아니다.

색상은 chalk가 `NO_COLOR` 환경변수와 TTY를 자동 감지해 비활성화한다. 테스트에서는 `chalk.level = 0`으로 강제한다.

## 변경 파일

repo root는 `manta-repo/`다. 아래 경로는 그 기준이다.

| 파일 | 변경 |
|---|---|
| `packages/cli/src/help/types.ts` | 신규. `CommandHelpEntry` 등 타입 |
| `packages/cli/src/help/command-registry.ts` | 신규. `commandHelpEntries` 배열 + `findHelpEntry(name)` |
| `packages/cli/src/help/apply-help-entry.ts` | 신규. 레지스트리 → Commander 어댑터 |
| `packages/cli/src/help/render-help.ts` | 신규. `renderOverview`, `renderCommand`, `toOverviewJson`, `toCommandJson` |
| `packages/cli/src/help/render-help.test.ts` | 신규. 렌더러·JSON 단위 테스트 |
| `packages/cli/src/commands/help.ts` | 신규. `createHelpCommand()` 팩토리. `--json` 옵션 처리 |
| `packages/cli/src/commands/init.ts` | 어댑터 연결 + 서브커맨드 `.configureHelp(...)` 연결 |
| `packages/cli/src/index.ts` | `createHelpCommand()` 등록 + 루트 `.configureHelp({ formatHelp })` |

core 패키지는 변경 없음.

## 검증

### 단위 테스트 (`render-help.test.ts`)

- `renderOverview`는 레지스트리 모든 엔트리 이름을 한 번씩 포함한다.
- `renderCommand(init)`는 `ARGUMENTS` 섹션과 `EXAMPLES` 블록을 포함하고, 각 example의 `input`을 `$ ` 접두로 출력한다.
- `findHelpEntry('init')`는 엔트리를 반환한다. `findHelpEntry('xyz')`는 `undefined`.
- `createHelpCommand()` action이 unknown 이름을 받으면 stderr 메시지 + `process.exitCode === 1`.
- `toOverviewJson(entries)`는 `{ kind: 'overview', version: '1', commands: [...] }` 구조를 반환한다.
- `toCommandJson(entry)`는 `{ kind: 'command', version: '1', command: ... }` 구조를 반환한다.

### Example 파싱 테스트

레지스트리 모든 엔트리의 모든 `example.input`을 Commander 파서로 돌려서 에러 없이 파싱되는지 확인한다. 이 테스트의 목적은 "문서에 적은 example이 parse 가능한가"를 검증하는 것이다.

테스트는 action의 실제 부작용을 실행하지 않는 방향으로 설계한다. `exitOverride()` 또는 테스트용 parser 경로를 써서 `--help`나 parse error가 테스트 프로세스를 종료시키지 않도록 한다.

### Integration 테스트

대표 example의 실제 실행 성공은 별도 integration test로 검증한다.

- 임시 디렉토리에서 `manta init` 실행 → `.manta/`와 `manta/tasks/` 생성 확인
- 임시 디렉토리에서 `manta init docs` 실행 → `.manta/`와 `docs/tasks/` 생성 확인

"모든 example은 parse 가능"과 "대표 example은 실제로 동작"을 분리해서 검증하는 구조다.

### 수동 시나리오

1. `manta help` — `init`, `help` 두 줄.
2. `manta help init`과 `manta init --help` 출력이 정확히 동일.
3. `manta --help`와 `manta help` 출력이 정확히 동일.
4. `manta help help`와 `manta help --help` 출력이 정확히 동일.
5. `manta help --json`은 `kind: "overview"`를 반환.
6. `manta help init --json`, `manta help help --json`은 `kind: "command"`를 반환.
7. `manta help xyz` — stderr에 unknown, exit code 1.
8. `NO_COLOR=1 manta help` — 컬러 없이 같은 레이아웃.

### 빌드/린트

- `npm run build` (core 먼저 빌드되는 순서 유지)
- `npm test`
- `npm run lint`

## 다음 태스크 연결

이 태스크가 끝나면 새 커맨드 추가는 단일 패턴이 된다 — 레지스트리에 엔트리 추가 → 어댑터로 Commander에 연결 → `.action()` 구현 → example 파싱/integration 테스트 추가.

CLI 전반의 오류 메시지와 exit code 정책 통일은 다음 태스크(`task-11`)에서 다룬다. 범위는 루트 unknown command, usage error, runtime failure, stderr 형식 일관화다.
