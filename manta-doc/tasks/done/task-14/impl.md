# Task-14: `manta add "제목"` 구현

## 배경

task-13에서 파일 시스템 계약(세 상태 폴더, frontmatter 3필드, 채번 규칙)이 고정됐다.
이제 그 계약 위에 첫 쓰기 명령을 올린다. `add`는 모든 task의 단방향 진입점이다 —
새 task는 **항상** `todo/`에 생긴다.

## 설계 결정

### core에 `createTask()`를 둔다

파일 생성은 CLI가 아니라 `@manta/core`(`task-repository.ts`)의 책임이다.
GUI도 같은 함수를 쓴다 (desktop의 add가 실제로 같은 경로를 탄다).

### 채번과 생성 사이의 경합은 `wx` 플래그로 막는다

`allocateNextTaskId()`(세 폴더 스캔, max+1) 직후 같은 id의 파일이 생겼다면
덮어쓰지 말고 실패해야 한다. `fs.writeFile(path, content, { flag: 'wx' })` 한 줄이
락 없이 이를 보장한다. 단일 사용자 CLI 전제(task-13)에서는 이 정도가 적정 수준이다.

### `created`는 호출자가 넘긴다

core의 `createTask(tasksRoot, title, created)`는 날짜를 만들지 않는다.
시간 의존성이 어댑터(CLI/GUI)로 밀려나야 core가 결정적이고 테스트가 단순해진다.

### 빈 제목은 CLI에서 usage error로 막는다

`manta add ""`는 Commander를 통과하므로 CLI action에서 trim 후 빈 문자열이면
`USAGE_ERROR`(exit 2)로 처리한다. core는 호출자를 신뢰한다.

## 출력 계약

```
Created task-5: Fix OAuth login (todo)
  file: manta/tasks/todo/task-5.md
```

파일 경로(프로젝트 루트 기준 상대 경로)를 함께 보여준다 — 파일이 source of truth이고,
AI와 사용자는 CLI를 거치지 않고 그 파일을 직접 열어 본문을 채울 수 있다.

## 검증

- core: 빈 프로젝트 → task-1, 세 폴더 분산 상태에서 max+1, 갭 미재사용 테스트
- CLI 통합: init → add → 파일 내용 byte 단위 확인 (`cli.test.ts`)
