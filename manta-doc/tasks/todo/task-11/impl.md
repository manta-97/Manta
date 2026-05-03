# Task-11: CLI 오류/exit code 정책 통일

## 배경

`task-10`에서 `manta help`, `manta help <command>`, `--help`, `--json` 경로의 출력 계약은 정리된다. 하지만 CLI 전반의 오류 처리는 아직 한 체계가 아니다.

예를 들어:

- `manta help xyz`는 help 커맨드 내부 정책을 타게 된다
- `manta xyz`는 Commander의 루트 unknown command 처리에 기대게 된다
- 잘못된 인자 개수나 옵션은 Commander 기본 usage error 형식을 따른다
- `manta init`의 runtime 실패는 커스텀 stderr 출력을 사용한다

이렇게 경로마다 stdout/stderr/exit code가 다르면 사람도 헷갈리고, AI는 더 취약하다. Manta가 "AI는 사용자다"를 진지하게 지키려면, prose help뿐 아니라 **오류 신호도 전역적으로 일관**되어야 한다.

**목표**: Manta CLI 전체에서 unknown command, usage error, runtime failure를 어떻게 분류하고 어떤 exit code / stderr 형식으로 노출할지 통일한다. 이번 태스크는 error JSON 포맷까지 다루지 않고, 텍스트 기반 CLI 계약과 테스트 전략을 정리하는 데 집중한다.

## 다룰 범위

이번 태스크는 아래를 포함한다.

- 루트 unknown command (`manta xyz`) 처리
- `help` 내부 unknown command (`manta help xyz`) 처리
- known command의 사용법 오류 처리
- runtime failure 처리 (`manta init` 실패 포함)
- stderr 메시지 형식 일관화
- Commander 기본 오류와 Manta 커스텀 오류의 경계 정리
- `exitOverride()`를 포함한 테스트 전략 정리

이번 태스크는 아래를 포함하지 않는다.

- JSON 에러 응답 포맷
- 향후 Electron/GUI 쪽 에러 표현
- core Result 타입 자체의 구조 변경

## 접근

큰 그림은 네 조각이다.

1. **오류 분류 체계 정리** — unknown command, usage error, runtime failure를 명시적인 범주로 구분한다.
2. **CLI text error policy 모듈** — exit code, symbolic error code, stderr formatter, sanitization을 한 곳에서 소유한다.
3. **Commander 오류 어댑터** — Commander가 내는 parse/usage/unknown command 오류를 Manta 정책 아래로 수렴시킨다.
4. **테스트 가능한 종료 정책** — 실제 CLI 실행과 테스트 코드 양쪽에서 검증 가능한 exit/stdout/stderr 계약을 정리한다.

## 설계 결정

### 왜 `task-10`에서 분리하는가

help 출력 계약과 CLI 전역 오류 정책은 서로 연관은 있지만 구현 난이도와 리스크가 다르다.

- help 태스크는 렌더러와 registry 중심이다
- 오류 정책 태스크는 Commander 종료 동작, stderr 형식, runtime error까지 건드린다

둘을 한 태스크에 묶으면 스코프가 커져서 구현과 검증이 동시에 흔들린다. 그래서 help 계약은 먼저 고정하고, 전역 오류 정책은 별도 태스크로 분리한다.

### 왜 help unknown과 루트 unknown을 같은 정책 아래 두는가

사용자 입장에서는 둘 다 "요청한 명령을 실행할 수 없다"는 같은 문제다.

- `manta xyz`
- `manta help xyz`

이 둘이 서로 다른 phrasing, 서로 다른 exit code, 서로 다른 stderr 구조를 갖기 시작하면 AI 분기 로직이 복잡해진다. 그래서 같은 분류 체계 아래에서 다룬다.

### 왜 Commander 기본 메시지를 그대로 두지 않고 정책을 씌우는가

Commander 기본 메시지는 빠르게 동작하지만, CLI 전체의 톤과 구조를 보장해주지는 않는다. `help`는 Manta 스타일인데 오류만 Commander 기본 스타일이면 인터페이스가 둘로 갈라진다.

이번 태스크에서는 Commander를 버리는 것이 아니라, Commander가 내는 오류를 Manta 정책 아래로 수렴시키는 방향을 택한다.

### 왜 JSON 에러 포맷은 이번에 제외하는가

이번 태스크의 핵심은 CLI 전반의 텍스트 오류 계약과 exit code를 먼저 고정하는 것이다. JSON 에러 포맷까지 같이 넣으면 설계 논점이 늘어난다.

AI 사용성 관점에서는 결국 JSON 에러도 필요할 수 있지만, help JSON을 먼저 고정한 다음 오류 JSON은 후속 태스크로 다루는 편이 더 안전하다.

### 왜 CLI-only error policy 모듈을 두는가

오류 출력은 CLI adapter의 책임이다. `@manta/core`는 runtime dependency가 없어야 하고, CLI prose나 stderr 문구를 알 필요가 없다.

그래서 `packages/cli/src/errors/cli-error-policy.ts`를 새로 두고 아래를 한 곳에서 소유한다.

- exit code 상수
- symbolic error code
- stderr formatter
- Commander error → Manta CLI error 변환
- 사용자 입력 sanitization

이렇게 해야 `index.ts`, `help.ts`, `init.ts`가 각자 `console.error()`와 `process.exitCode`를 직접 만들지 않는다. 새 command가 추가되어도 같은 error policy를 호출한다.

### 왜 exit code를 `0/1/2`로 나누는가

Manta CLI는 사람뿐 아니라 AI와 shell script도 사용한다. 이 소비자들은 stderr 문장을 읽기 전에 exit code로 먼저 분기한다.

이번 태스크의 exit code 계약은 아래로 고정한다.

| exit code | 의미 | 예시 |
|---:|---|---|
| `0` | 성공 또는 안전한 no-op | `manta help`, `manta init`의 `ALREADY_INITIALIZED` |
| `1` | runtime failure | 파일 시스템 권한 문제, init 중 I/O 실패 |
| `2` | invocation/usage error | unknown command, unknown option, 인자 개수 오류 |

모든 실패를 `1`로 뭉개면 task-11의 핵심인 usage/runtime 구분이 stderr 파싱으로 밀린다. 반대로 sysexits 스타일의 `64` 같은 값은 지금 Node CLI 사용성에 비해 설명 비용이 크다. `0/1/2`가 단순하면서도 AI 분기에는 충분하다.

### 왜 text stderr에 symbolic error code를 넣는가

JSON error는 이번 범위 밖이지만, text stderr도 안정 식별자를 가져야 한다.

형식은 아래처럼 고정한다.

```text
[UNKNOWN_COMMAND] Unknown command: xyz. Run `manta help` to see available commands.
[USAGE_ERROR] Usage error: unknown option '--bad'
[RUNTIME_FAILURE] Runtime failure: Permission denied: /path
```

사람은 뒤의 문장을 읽고, AI와 로그 검색은 앞의 code를 사용할 수 있다. 문구가 조금 바뀌어도 `[UNKNOWN_COMMAND]` 같은 식별자는 유지한다.

### 왜 stderr 사용자 입력을 sanitize하는가

unknown command나 bad option처럼 사용자 입력이 stderr에 echo되는 경로가 있다. 이 값에 newline, ANSI escape, 긴 문자열이 들어가면 터미널과 로그가 헷갈릴 수 있다.

이번 태스크에서는 stderr에 들어가는 사용자 제어 값에 아래 처리를 적용한다.

- ANSI escape 제거
- control character를 공백으로 치환
- whitespace를 한 줄로 접기
- 너무 긴 값은 120자 기준으로 자르기

이건 보안 기능이라기보다 CLI 출력 신뢰성을 지키는 hygiene이다.

### 왜 `parseAsync()`를 쓰는가

`manta init` action은 async 파일 I/O를 한다. 이후 추가될 `add`, `list`, `search` 같은 command도 async가 될 가능성이 높다.

루트 진입점에서 `program.parse()`를 계속 쓰면 비동기 실패를 top-level error policy가 안정적으로 기다리고 분류하기 어렵다. 따라서 `packages/cli/src/cli.ts`에서 `runMantaCli()`를 만들고, 내부에서 `program.parseAsync()`를 호출한다.

## 오류 분류 정책

아래 분류를 기준으로 정책을 고정한다.

| 분류 | symbolic code | exit code | 예시 | stdout | stderr |
|---|---|---:|---|---|---|
| unknown command | `UNKNOWN_COMMAND` | `2` | `manta xyz`, `manta help xyz` | empty | `[UNKNOWN_COMMAND] ...` |
| usage error | `USAGE_ERROR` | `2` | 필수 인자 누락, 허용되지 않은 옵션, 인자 개수 오류 | empty | `[USAGE_ERROR] ...` |
| runtime failure | `RUNTIME_FAILURE` | `1` | 파일 시스템 권한 문제, init 중 I/O 실패 | empty | `[RUNTIME_FAILURE] ...` |

정상 help와 정상 command 출력은 기존 stdout 계약을 유지한다. 오류 경로에서는 stdout이 비어 있어야 한다.

### Error/rescue registry

| 코드 경로 | 실제 입력/오류 | 분류 | rescue 위치 | 사용자에게 보이는 것 | 테스트 |
|---|---|---|---|---|---|
| root parse | `manta xyz` / `commander.unknownCommand` | `UNKNOWN_COMMAND` | `runMantaCli()` catch | `[UNKNOWN_COMMAND] Unknown command: xyz...`, exit `2` | unit + CLI contract |
| help action | `manta help xyz` | `UNKNOWN_COMMAND` | `help.ts` action | root unknown과 같은 stderr, exit `2` | unit + CLI contract |
| Commander option parse | `manta init --bad` / `commander.unknownOption` | `USAGE_ERROR` | `runMantaCli()` catch | `[USAGE_ERROR] Usage error: unknown option '--bad'`, exit `2` | unit + CLI contract |
| Commander arity parse | `manta help init extra` / `commander.excessArguments` | `USAGE_ERROR` | `runMantaCli()` catch | `[USAGE_ERROR] Usage error: too many arguments...`, exit `2` | unit + CLI contract |
| core result | `PATH_IS_FILE`, `PERMISSION_DENIED`, `UNKNOWN` | `RUNTIME_FAILURE` | command action | `[RUNTIME_FAILURE] Runtime failure: ...`, exit `1` | unit/integration |
| core result | `ALREADY_INITIALIZED` | success/no-op | `init.ts` action | 안내 메시지, exit `0` | command test |

## 구현 방향

### 1. 루트 진입점에서 종료 동작 제어

`manta-repo/packages/cli/src/index.ts`에서 직접 Commander program을 구성하지 않는다. 새 파일 `packages/cli/src/cli.ts`를 만들고 아래 역할을 맡긴다.

- `createMantaProgram()`: Commander program 생성, help hook 연결, command 등록
- `runMantaCli()`: `parseAsync()` 실행, `CommanderError`와 unknown error를 CLI error policy로 변환

직접 `process.exit()`를 흩뿌리기보다, Commander가 내는 종료/예외를 한 곳에서 분류하는 방향이 좋다. 그래야 루트 unknown command와 usage error를 한 레이어에서 통제할 수 있다.

### 2. 서브커맨드 runtime 실패와 usage 실패를 분리

`init` 같은 커맨드의 실패는 두 종류다.

- 인자를 잘못 줘서 Commander 단계에서 막히는 실패
- 실제 파일 시스템 작업 중 실패하는 runtime failure

이 둘은 사용자에게는 모두 "실패"지만, AI 분기 로직 관점에서는 성격이 다르다. 그래서 메시지와 exit code를 분리해서 다룬다.

### 3. stderr 형식 표준화

최소한 다음을 일관되게 만든다.

- unknown command 메시지
- usage error 메시지
- runtime failure 메시지

출력 포맷은 아래 한 줄 형식으로 고정한다.

```text
[CODE] Message
```

루트/서브커맨드/Commander 기본 경로가 모두 이 형식을 따라야 한다. Commander 기본 stderr는 `configureOutput()`으로 숨기고, catch block에서 Manta formatter로 한 번만 출력한다.

### 4. 코드 적용 가이드 분리

실제 적용할 diff는 [implementation-code.md](./implementation-code.md)에 둔다. `impl.md`는 정책, 경계, 테스트, 완료 기준의 source of truth다.

## 테스트 전략

### 단위/구조 테스트

- 오류 분류 함수가 입력 케이스별로 expected category를 반환하는지 확인
- stderr formatter가 분류별로 동일한 형식을 만드는지 확인
- 사용자 입력 sanitization이 newline/ANSI/긴 문자열을 안전하게 처리하는지 확인
- CommanderError code가 `UNKNOWN_COMMAND`/`USAGE_ERROR`로 매핑되는지 확인

### CLI 동작 테스트

`exitOverride()`를 사용해 Commander가 테스트 프로세스를 종료시키지 않도록 한다. 테스트는 stdout, stderr, exit code를 함께 assert할 수 있어야 한다.

단위 테스트만으로는 부족하다. CLI 오류 정책은 process-level contract라서 실제 entrypoint에 가까운 `runMantaCli()` 호출 또는 subprocess 테스트로 stdout/stderr/exit code를 검증한다.

최소 시나리오:

1. `manta xyz`
2. `manta help xyz`
3. 필수 인자 누락
4. 허용되지 않은 옵션
5. `manta init` runtime failure
6. `manta init`의 `ALREADY_INITIALIZED`

각 시나리오의 기대값:

| 시나리오 | stdout | stderr | exit |
|---|---|---|---:|
| `manta xyz` | empty | `[UNKNOWN_COMMAND] ...` | `2` |
| `manta help xyz` | empty | `[UNKNOWN_COMMAND] ...` | `2` |
| bad option | empty | `[USAGE_ERROR] ...` | `2` |
| too many args | empty | `[USAGE_ERROR] ...` | `2` |
| `manta init` runtime failure | empty | `[RUNTIME_FAILURE] ...` | `1` |
| `manta init` already initialized | 안내 stdout | empty | `0` |

### 수동 시나리오

1. unknown command는 루트/`help` 경로 모두 같은 정책을 따른다.
2. usage error는 서브커맨드마다 phrasing이 달라지지 않는다.
3. runtime failure는 usage error와 구분되는 stderr/exit code를 가진다.
4. 정상 help 출력은 이 태스크로 인해 바뀌지 않는다.

## 완료 기준

- `manta xyz`와 `manta help xyz`가 같은 분류 체계 아래에서 처리된다
- usage error와 runtime failure가 구분된 exit code와 stderr 형식을 가진다
- unknown/usage/runtime failure가 stable symbolic error code를 가진다
- stderr에 echo되는 사용자 입력은 한 줄로 sanitize된다
- 루트 entrypoint가 `parseAsync()` 기반으로 async command failure를 기다린다
- `manta init` runtime failure 처리도 새 정책 아래로 수렴한다
- 테스트 코드에서 Commander 종료를 가로채며 stdout/stderr/exit code를 검증할 수 있다
- 실제 CLI contract에 가까운 테스트가 stdout/stderr/exit code를 함께 검증한다
- `task-10`에서 정의한 help 출력 계약은 유지된다

## 구현 코드 문서

구현에 필요한 실제 diff와 주석은 [implementation-code.md](./implementation-code.md)에 분리했다. `impl.md`는 정책과 완료 기준을 유지하고, 코드는 별도 문서에서 직접 따라 적용한다.

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 1 | issues_open | HOLD_SCOPE, 0 critical gaps, 1 deferred follow-up |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | - | not run |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 0 | - | eng review required |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | - | skipped, no UI scope |
| DX Review | `/plan-devex-review` | Developer experience gaps | 0 | - | not run |

- **UNRESOLVED:** 0
- **VERDICT:** CEO review found no critical gaps, but eng review is still required before implementation.
