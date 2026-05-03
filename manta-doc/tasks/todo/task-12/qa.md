# Task-12: Task-10 `manta help` 수동 QA

`impl.md`의 「동작」 계약과 「검증 > 수동 시나리오」를 실제 손으로 재현하는 문서다. 자동 테스트(`npm test`)가 커버하지 못하는 **두 출력 경로의 byte-level 동등성**, **TTY/NO_COLOR 동작**, **exit code** 를 직접 눈으로 확인하는 것이 이 QA의 목적이다.

## 0. 준비

### 0-1. 빌드

`@manta/cli`는 `@manta/core`에 의존하므로 **core 먼저** 빌드돼야 한다. 루트의 `npm run build`가 순서를 맞춰준다.

```bash
cd manta-repo
npm install                    # 최초 1회, 혹은 package-lock 변경 후
npm run build                  # core → cli 순서로 tsc 실행
```

빌드가 끝나면 실행 파일은 `packages/cli/dist/index.js`다. 이후 모든 QA 커맨드는 이 경로를 직접 호출한다:

```bash
MANTA="node packages/cli/dist/index.js"
```

> CLI를 `npm link`로 전역 등록해 `manta` 이름으로 쓰고 싶다면 `cd packages/cli && npm link` 후 `MANTA=manta`로 두면 된다. 다만 빌드 결과가 아닌 오래된 링크를 보게 되는 사고가 잦아서 QA에서는 `dist/index.js` 직접 호출을 권장한다.

### 0-2. 자동 테스트 선통과

수동 QA는 자동 테스트가 모두 초록인 상태에서 시작한다. 유닛·파싱·integration 테스트가 이미 빨간 상태면 수동 QA는 아무 의미가 없다.

```bash
npm test                       # 모든 패키지의 jest 테스트
npm run lint                   # eslint
```

둘 다 통과한 커밋에서 아래 시나리오를 돌린다.

### 0-3. 작업 디렉토리

`manta init` 계열 시나리오(S-9, S-10)는 **빈 임시 디렉토리**에서 돌려야 기존 `.manta/` 상태를 오염시키지 않는다.

```bash
SCRATCH=$(mktemp -d)
echo "scratch: $SCRATCH"
```

이후 `init` 관련 명령은 `cd "$SCRATCH"` 상태에서 실행한다. help 출력만 보는 시나리오(S-1 ~ S-8)는 아무 디렉토리에서 돌려도 된다.

---

## 1. 시나리오

각 시나리오는 **무엇을 보는가 → 커맨드 → 기대 → 확인 포인트** 순서다. 단순히 명령을 쳐보는 게 아니라, 어떤 불변식이 깨지면 회귀인지를 먼저 이해하고 실행한다.

### S-1. `manta help` — 구현된 커맨드만 보인다

**보는 것**: help registry가 source of truth라는 계약. 구현되지 않은 커맨드는 등장하지 않아야 한다.

```bash
$MANTA help
```

기대 출력(요지):

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

확인 포인트:
- `COMMANDS` 섹션에 **`init`, `help` 딱 두 줄만**. `add`, `list`, `show`, `start`, `done`, `edit`가 등장하면 회귀.
- 타이틀에 em-dash `—`가 그대로 렌더됨 (UTF-8 터미널 전제).
- exit code 0: `echo $?`

### S-2. `manta --help` ↔ `manta help` — 정확히 동일

**보는 것**: 루트 `--help`도 같은 렌더러를 통과한다는 계약. trailing newline 포함 byte 단위로 같아야 한다.

```bash
diff <(NO_COLOR=1 $MANTA help) <(NO_COLOR=1 $MANTA --help)
echo "exit: $?"
```

기대: `diff` 출력이 **비어 있고** exit 0.

참고로 `wc -c`로 바이트 수도 확인해두면 한 줄짜리 trailing newline 실수를 빠르게 잡을 수 있다:

```bash
NO_COLOR=1 $MANTA help      | wc -c
NO_COLOR=1 $MANTA --help    | wc -c
```

두 값이 같아야 한다. 과거 회귀 기록: 1바이트 차이가 났던 시기가 있었다(console.log 자동 `\n` vs Commander `writeOut` 미부착). 지금 구현은 `formatForCommanderHook`으로 이 틈을 막는다.

### S-3. `manta help init` ↔ `manta init --help` — 정확히 동일

```bash
diff <(NO_COLOR=1 $MANTA help init) <(NO_COLOR=1 $MANTA init --help)
echo "exit: $?"
```

기대: `diff` 빈 출력, exit 0.

눈으로도 확인:

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

확인 포인트:
- `OPTIONS` 섹션이 **없음** (init은 옵션 없음 — 조건부 렌더 계약).
- 각 예시가 `$ ` 접두로 시작.

### S-4. `manta help help` ↔ `manta help --help` — 정확히 동일

```bash
diff <(NO_COLOR=1 $MANTA help help) <(NO_COLOR=1 $MANTA help --help)
echo "exit: $?"
```

기대: `diff` 빈 출력, exit 0.

확인 포인트:
- 이번엔 `OPTIONS` 섹션이 **있어야 한다** (`--json` 때문).
- `ARGUMENTS`에 `command` 항목.

### S-5. `manta help --json` — overview JSON

**보는 것**: AI 소비자용 안정 인터페이스. top-level `kind`/`version`이 먼저 오고 `commands`가 이어진다.

```bash
$MANTA help --json | jq '{kind, version, command_names: [.commands[].name]}'
```

기대:

```json
{
  "kind": "overview",
  "version": "1",
  "command_names": ["init", "help"]
}
```

확인 포인트:
- `kind === "overview"`, `version === "1"` (문자열).
- `commands` 배열에 등록된 엔트리가 모두 포함.
- exit code 0.

### S-6. `manta help init --json` — command JSON

```bash
$MANTA help init --json | jq '{kind, version, name: .command.name, args: .command.args}'
```

기대:

```json
{
  "kind": "command",
  "version": "1",
  "name": "init",
  "args": [
    { "name": "path", "required": false, "description": "task directory name (default: manta)" }
  ]
}
```

### S-7. `manta help help --json`

```bash
$MANTA help help --json | jq '{kind, options: .command.options}'
```

기대: `options` 배열에 `--json` 플래그 엔트리 존재.

```json
{
  "kind": "command",
  "options": [
    { "flag": "--json", "description": "emit machine-readable JSON instead of prose" }
  ]
}
```

### S-8. `manta help xyz` — unknown command

**보는 것**: help 범위의 exit code 계약. 이 태스크에서 확정한 유일한 "실패 exit 1" 경로다 (루트 unknown command / usage error 통일은 task-11 범위).

```bash
$MANTA help xyz
echo "exit: $?"
```

기대:
- **stderr**로 `Unknown command: xyz. Run \`manta help\` to see available commands.`
- **stdout은 비어 있음** — prose/JSON 아무것도 나오면 안 된다.
- exit code **1**.

stdout/stderr 분리 확인:

```bash
$MANTA help xyz 1>/tmp/manta-stdout 2>/tmp/manta-stderr
echo "exit: $?"
wc -c /tmp/manta-stdout   # 0이어야 함
cat /tmp/manta-stderr      # Unknown command 메시지
```

### S-9. `manta init` — 대표 example 실제 실행 (integration)

**보는 것**: help에 적힌 example이 거짓말이 아니다.

```bash
cd "$SCRATCH" && rm -rf .manta manta docs
$MANTA init
echo "exit: $?"
ls -la .manta manta/tasks 2>/dev/null
```

기대:
- exit code 0.
- `.manta/` 디렉토리 생성.
- `manta/tasks/` 디렉토리 생성.

### S-10. `manta init docs` — 경로 인자

```bash
cd "$SCRATCH" && rm -rf .manta manta docs
$MANTA init docs
echo "exit: $?"
ls -la .manta docs/tasks 2>/dev/null
```

기대:
- exit code 0.
- `.manta/`, `docs/tasks/` 생성.
- `manta/` 디렉토리는 **생성되지 않아야** 한다 (인자로 준 이름만 쓰인다).

### S-11. `NO_COLOR=1` — 색상 비활성

**보는 것**: chalk가 환경 변수를 존중하는지. 출력 바이트 수가 색상 유/무에 따라 달라지는지.

```bash
$MANTA help                   > /tmp/manta-color.txt       # TTY 아님 → 기본적으로 색 없음
NO_COLOR=1 $MANTA help        > /tmp/manta-nocolor.txt
diff /tmp/manta-color.txt /tmp/manta-nocolor.txt
```

기대: 파이프로 리다이렉트하면 TTY가 아니므로 chalk가 이미 색을 빼고 출력한다. 두 파일은 동일해야 한다.

TTY에서 색상이 실제로 들어가는지는 터미널에서 직접 눈으로 본다:

```bash
$MANTA help                   # 터미널에서 직접 — 타이틀/섹션 헤더가 bold, 꼬리 문장이 dim
NO_COLOR=1 $MANTA help        # 위와 같은 레이아웃인데 색/굵기만 빠짐
```

확인 포인트:
- `NO_COLOR=1`일 때 ANSI escape(`\x1b[`) 문자가 출력에 없어야 한다.
  ```bash
  NO_COLOR=1 $MANTA help | grep -c $'\x1b\\['   # 0 이어야 함
  ```

### S-12. `manta help` example이 실제 parse 가능 (sanity)

레지스트리의 모든 `example.input`은 파싱 테스트로 자동 검증되지만, 의심될 때는 수동으로 한 번 돌려볼 수 있다 (단 action 부작용이 실행되므로 S-9/S-10처럼 빈 디렉토리에서):

```bash
cd "$SCRATCH" && rm -rf .manta manta docs
$MANTA help         ; echo "→ exit $?"
$MANTA help init    ; echo "→ exit $?"
$MANTA help --json  > /dev/null ; echo "→ exit $?"
$MANTA help init --json > /dev/null ; echo "→ exit $?"
```

전부 exit 0이어야 한다.

---

## 2. 빠른 회귀 체크 (one-liner)

이전에 잡혔던 회귀를 한 줄로 재확인:

```bash
# 두 출력 경로의 byte 동등성
diff <(NO_COLOR=1 $MANTA help)       <(NO_COLOR=1 $MANTA --help)      && \
diff <(NO_COLOR=1 $MANTA help init)  <(NO_COLOR=1 $MANTA init --help) && \
diff <(NO_COLOR=1 $MANTA help help)  <(NO_COLOR=1 $MANTA help --help) && \
echo "OK: all three help paths are byte-identical"

# JSON 포맷 버전
$MANTA help --json        | jq -e '.kind == "overview" and .version == "1"' > /dev/null && echo "OK: overview json"
$MANTA help init --json   | jq -e '.kind == "command"  and .version == "1"' > /dev/null && echo "OK: command json"

# unknown command exit code
$MANTA help xyz 2>/dev/null ; [ $? -eq 1 ] && echo "OK: unknown exits 1"
```

세 블록이 전부 OK면 help 계약의 핵심은 무너지지 않은 것이다.

---

## 3. 이 QA 문서가 커버하지 않는 것

- **루트 unknown command**(`manta xyz`) 같은 CLI 전반 exit code 통일 — task-11 범위.
- `manta init --help --json` 조합 — impl.md에서 이번 스코프 제외 명시.
- CP949 등 비 UTF-8 터미널 — 환경 가정에서 배제.
- Windows cmd.exe / PowerShell 경로 — `diff <(...)`는 bash 문법이므로 git bash 또는 zsh 기준.
