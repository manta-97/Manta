# Task-8: 에러 정의 중앙화 (`errors.ts`)

## 배경

현재 에러는 `init.ts`에서 인라인으로 생성하고, 에러 코드는 `types.ts`의 `InitResult` union에 하드코딩되어 있다.
모듈이 늘어나면 에러 정의가 여러 파일에 흩어져서 "이 에러가 어디서 정의됐지?"를 추적하기 어려워진다.

**목표**: 에러 정의를 `errors.ts` 한 파일로 모아서, 에러 코드와 메시지를 한곳에서 관리한다.

## 접근

- `packages/core/src/errors.ts`를 신규 생성한다.
- 에러 코드별 팩토리 함수를 가진 `MantaErrors` 객체를 만든다. 팩토리는 `{ error, message }` 를 반환한다.
- `MantaErrorCode` 타입을 `keyof typeof MantaErrors`로 자동 유도한다.
- `types.ts`의 하드코딩된 에러 코드 union을 `MantaErrorCode`로 교체한다.
- `init.ts`의 인라인 에러 객체를 `MantaErrors.XXX()` 호출로 교체한다.
- `index.ts`에서 re-export 한다.

## 설계 결정

### 팩토리 객체 vs enum vs class

팩토리 객체(`MantaErrors.PATH_IS_FILE(path)`)를 선택한다.

- **enum**: 코드만 정의할 수 있고, 메시지 템플릿을 담을 수 없다. 결국 별도 매핑이 필요해져서 정의가 두 곳으로 나뉜다.
- **Error class 상속**: Result 패턴(`ok: false`)을 쓰고 있어서 throw/catch 기반의 클래스 계층과 맞지 않는다.
- **팩토리 객체**: 코드 + 메시지 템플릿을 한곳에 두고, `as const`로 리터럴 타입을 보존한다. 기존 Result 패턴과 자연스럽게 합쳐진다(`{ ok: false, ...MantaErrors.XXX() }`).

### `UNKNOWN`은 `MantaErrors`에 넣지 않는다

`UNKNOWN`은 예측 불가능한 에러를 위한 catch-all이다. 팩토리로 만들 메시지 템플릿이 없고, 원본 에러의 메시지를 그대로 전달해야 하므로 `MantaErrors`에 포함하지 않는다. `types.ts`에서 `MantaErrorCode | 'UNKNOWN'`으로 유지한다.

### CLI 수정 불필요

CLI(`packages/cli/src/commands/init.ts`)는 `result.error === 'ALREADY_INITIALIZED'` 같은 문자열 비교를 쓰고 있다. 에러 코드 값 자체는 바뀌지 않으므로 CLI 쪽 변경은 없다.

## 변경 파일

| 파일 | 변경 |
|------|------|
| `packages/core/src/errors.ts` | 신규. `MantaErrors` 객체 + `MantaErrorCode` 타입 |
| `packages/core/src/types.ts` | 하드코딩 에러 코드 → `MantaErrorCode` 참조 |
| `packages/core/src/init.ts` | 인라인 에러 → `MantaErrors` 팩토리 호출 |
| `packages/core/src/index.ts` | re-export 추가 |
