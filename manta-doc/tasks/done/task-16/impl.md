# Task-16: `manta start <id>` / `manta done <id>` 구현

## 배경

폴더 = 상태(task-13)이므로 상태 전환은 `mv`가 전부다.
core의 `moveTask(tasksRoot, id, targetStatus)` 하나로 두 명령을 다 만든다.

## 설계 결정

### 두 명령은 한 팩토리의 두 인스턴스다

`start`와 `done`은 "task 파일을 목표 상태 폴더로 옮긴다"는 같은 동작에
목표 상태만 다르다. `createMoveTaskCommand(name, targetStatus)` 팩토리로
두 명령을 생성한다. 세 번째 전환 명령(예: `manta todo <id>` 같은 되돌리기)이
필요해져도 한 줄이면 된다.

### 느슨한 state machine, 최소 가드 (task-13 계약 그대로)

- 없는 task → `TASK_NOT_FOUND`, exit 1
- 이미 목표 상태 → no-op + `task-3 is already done (no-op).` + **exit 0**
- 그 외 전환은 전부 허용 — `done`에 있는 task에 `start`를 치면 리오픈된다

no-op이 성공(exit 0)인 이유: 스크립트와 AI가 "원하는 상태로 만들어라"를
멱등하게 반복 실행할 수 있어야 한다. 이미 그 상태라면 실패가 아니라 달성이다.

### 전환 결과는 from → to로 말한다

```
task-3: todo → in-progress
```

무엇이 일어났는지를 정확히 알린다. 특히 `done → in-progress` 같은 역방향 전환에서
사용자가 의도하지 않은 이동을 바로 알아챌 수 있다.

## 검증

- core: 전환/역전환/no-op(`moved: false`)/부재 테스트, 이동 후 원본 부재 확인
- CLI 통합: start → done → 반복 done(no-op, exit 0) 흐름
