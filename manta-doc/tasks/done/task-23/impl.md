# Task-23: Local Workspace v1 — 에디터 + Command Palette + Copy AI Context (Phase 4 완성)

## 배경

v0(task-21)는 read-only 미리보기까지였다. roadmap Phase 4의 나머지 —
"가운데: Markdown editor", "전역: command palette", "AI context는 on-demand action" —
를 채워서 Phase 4를 닫는다.

## 설계 결정

### 편집은 draft, 파일은 Save에서만

편집 모드는 textarea의 로컬 상태(draft)에서만 논다. 파일은 명시적 Save(버튼 또는
⌘S)에서만 변경된다 — "명시적 save action 없이는 파일을 변경하지 않는다"는
cli-design 계약의 직접 구현이다. 다른 task를 선택하면 draft는 폐기된다.

### 본문만 편집한다 (frontmatter는 보호)

에디터는 raw 파일이 아니라 **본문**을 편집한다. core의 `updateTaskBody()`가
frontmatter(id/title/created)를 보존한 채 본문만 교체하고, 쓰기 전에 기존 파일을
엄격하게 읽으므로 깨진 파일을 모르고 덮어쓰는 일이 없다. GUI에서 frontmatter를
망가뜨릴 방법 자체를 없앴다 — raw 편집이 필요하면 `manta edit`(=$EDITOR)나
파일 직접 수정이라는 더 정직한 경로가 이미 있다.

### Copy AI Context는 CLI와 같은 함수다

palette/버튼의 Copy AI Context는 main process에서 `buildContextDocument()`를
호출한다 — `manta context`와 동일한 코드 경로. GUI 전용 context 포맷을 만들지
않으므로 두 표면의 출력이 갈라질 수 없다. 클립보드 쓰기는 renderer의
`navigator.clipboard`(포커스/권한 조건이 있음) 대신 main의 Electron `clipboard`로
처리한다.

### Palette의 add는 검색어 그 자체다

⌘K palette에 별도 "new task" 폼을 만드는 대신, 입력한 텍스트가 액션과 매치되지
않으면 `Add task: "<입력>"` 액션이 동적으로 생긴다. 입력창 하나가 검색이자
생성이다 — Linear의 quick-add 감각을 최소 구현으로.

## 검증

- desktop typecheck + electron-forge 기동 + renderer 단독 vite build 통과
- updateTaskBody는 core 단위 테스트로 커버 (frontmatter 보존, malformed 거부)
- 화면 조작 수동 QA는 여전히 후속 과제 (앱 기동까지 검증)

## 남긴 것

- title 편집 (frontmatter 변경 — 별도 계약 필요)
- 멀티 task 선택 후 context 조립 (지금은 선택된 task 1개)
- palette 키보드 네비게이션 (지금은 Enter=첫 액션, 클릭)
