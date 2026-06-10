# Task-21: Local Workspace GUI v0 — Electron을 core에 연결

## 배경

task-9에서 Electron 환경(electron-forge + Vite + React + Tailwind)은 준비됐지만
preload가 비어 있었다 — 화면은 뜨는데 core와 연결되지 않은 상태.
이번에 IPC 계층을 만들고 cli-design Phase 4 v0 방향(목록 + read-only 미리보기)을 구현했다.

## 설계 결정

### GUI는 core 위의 동등한 adapter다 (CLI를 호출하지 않는다)

cli-design 원칙 그대로: main process가 `@manta/core`를 직접 호출한다.
CLI를 shell로 부르거나 stdout을 파싱하는 코드는 없다. CLI와 GUI는 같은 함수
(`listTasks`, `readTask`, `createTask`, `moveTask`)를 쓰므로 동작이 갈라질 수 없다.

### 실패는 예외가 아니라 값으로 renderer까지 간다

IPC 핸들러는 core의 Result(`{ ok: false, error, message }`)를 그대로 전달하고,
예외는 경계에서 `UNKNOWN` Result로 변환한다. renderer는 CLI 사용자와 똑같이
오류 코드로 분기한다 — "error code는 GUI 표시를 위한 안정 식별자"라는 계약의 실현.

### preload는 좁은 타입드 API만 노출한다

contextIsolation 아래에서 renderer가 파일에 닿는 유일한 통로는
`window.manta`(5개 메서드)다. 채널 이름·DTO 타입은 `src/shared/manta-api.ts`
한 곳이 소유하고 main/preload/renderer가 공유한다.

### renderer는 별도 source of truth를 만들지 않는다

add/move 성공 후 항상 task 목록을 다시 읽는다. 로컬 상태를 낙관적으로 고치는
대신 파일 시스템을 재조회한다 — 파일이 원본이라는 원칙을 UI 상태 관리에도 적용.

### 화면 구성 (v0)

```
┌──────────┬──────────────┬────────────────────────────┐
│ Projects │ Tasks        │ Task preview (read-only)   │
│ 레지스트리│ 상태별 그룹   │ frontmatter 메타 + 본문     │
│          │ + 새 task 입력│ + Start / Done 버튼        │
└──────────┴──────────────┴────────────────────────────┘
```

- preview는 read-only. 편집과 command palette는 다음 버전 (cli-design Phase 4 계약 유지).
- 깨진 task 파일은 CLI list와 같은 방식으로 `(malformed task file)` 표시.
- 레지스트리에 있지만 폴더가 사라진 프로젝트는 비활성으로 표시.

### 빌드 우회 한 가지: main 번들은 core TS 소스를 직접 번들

core는 CJS로 컴파일되는데 rollup이 tsc의 `Object.defineProperty(exports, ...)`
재-export에서 named export를 정적으로 읽지 못한다. vite.main.config의 alias로
`@manta/core` → `../core/src/index.ts`를 걸어 TS 소스를 직접 번들한다.
renderer는 브라우저 컨텍스트라 core에서 **타입만** 가져온다 (값은 shared에 복제).

## 검증

- `npm run typecheck --workspace packages/desktop` 통과
- `electron-forge start`로 앱 부팅 확인 (main/preload 번들 빌드 성공, 무에러 기동)
- renderer 번들 단독 vite build 성공 (Tailwind 클래스 생성 확인)
- UI 상호작용의 수동 검증은 후속으로 남음 (앱은 뜨지만 화면 조작 QA는 미수행)

## 남긴 것

- task Markdown **편집** (명시적 save action 계약 설계 필요)
- command palette (add / start / done / context)
- root SQLite를 GUI 목록/검색에 활용 (지금은 파일 직접 스캔으로 충분)
