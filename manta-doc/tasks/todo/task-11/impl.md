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

큰 그림은 세 조각이다.

1. **오류 분류 체계 정리** — unknown command, usage error, runtime failure를 명시적인 범주로 구분한다.
2. **Commander 오류 어댑터** — Commander가 내는 parse/usage/unknown command 오류를 Manta 정책 아래로 수렴시킨다.
3. **테스트 가능한 종료 정책** — 실제 CLI 실행과 테스트 코드 양쪽에서 검증 가능한 exit/stderr 계약을 정리한다.

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

## 오류 분류 초안

아래 분류를 기준으로 정책을 정리한다.

| 분류 | 예시 |
|---|---|
| unknown command | `manta xyz`, `manta help xyz` |
| usage error | 필수 인자 누락, 허용되지 않은 옵션, 인자 개수 오류 |
| runtime failure | 파일 시스템 권한 문제, init 중 I/O 실패 등 |

초기 목표는 각 분류별로 아래 세 가지를 고정하는 것이다.

- stderr에 어떤 문구가 나가는가
- exit code가 무엇인가
- Commander 기본 출력과 어떤 관계를 가지는가

## 구현 방향

### 1. 루트 진입점에서 종료 동작 제어

`manta-repo/packages/cli/src/index.ts`에서 Commander parse 흐름을 감싸서, 테스트 중에도 종료를 가로챌 수 있는 구조를 만든다.

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

한 줄 메시지로 끝낼지, usage 힌트를 함께 붙일지, `Error:` 같은 프리픽스를 붙일지 이 태스크에서 하나로 고정한다. 출력 포맷은 루트/서브커맨드/Commander 기본 경로에서 동일해야 한다.

## 테스트 전략

### 단위/구조 테스트

- 오류 분류 함수가 있다면 입력 케이스별로 expected category를 반환하는지 확인
- stderr formatter가 분류별로 동일한 형식을 만드는지 확인

### CLI 동작 테스트

`exitOverride()`를 사용해 Commander가 테스트 프로세스를 종료시키지 않도록 한다. 테스트는 stdout, stderr, exit code를 함께 assert할 수 있어야 한다.

최소 시나리오:

1. `manta xyz`
2. `manta help xyz`
3. 필수 인자 누락
4. 허용되지 않은 옵션
5. `manta init` runtime failure
6. `manta init`의 `ALREADY_INITIALIZED`

### 수동 시나리오

1. unknown command는 루트/`help` 경로 모두 같은 정책을 따른다.
2. usage error는 서브커맨드마다 phrasing이 달라지지 않는다.
3. runtime failure는 usage error와 구분되는 stderr/exit code를 가진다.
4. 정상 help 출력은 이 태스크로 인해 바뀌지 않는다.

## 완료 기준

- `manta xyz`와 `manta help xyz`가 같은 분류 체계 아래에서 처리된다
- usage error와 runtime failure가 구분된 exit code와 stderr 형식을 가진다
- `manta init` runtime failure 처리도 새 정책 아래로 수렴한다
- 테스트 코드에서 Commander 종료를 가로채며 stdout/stderr/exit code를 검증할 수 있다
- `task-10`에서 정의한 help 출력 계약은 유지된다
