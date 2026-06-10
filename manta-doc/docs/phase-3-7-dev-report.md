# Phase 3~7 개발 결과 보고 — AI Context · GUI v1 · Sections · Import/Export · Pro 기획

> 2026-06-10 · manta-repo 브랜치 `phase3-6-context-gui-export` · [Phase 1 보고서](phase-1-dev-report.md)에 이어짐

## 1. 어디서 출발했나

Phase 1~2가 끝난 시점의 Manta는 "task를 만들고 옮기고 찾는" 시스템이었다.
하지만 Manifesto가 말하는 핵심 가치 — **완료된 작업이 미래 AI 세션의 입력으로
되살아나는 것** — 은 아직 없었다. roadmap의 남은 phase들이 전부 이 한 방향을
가리킨다: context(꺼내기), sections(꺼낼 때 무엇이 중요한가), GUI(꺼내는 동작의
UX), export(다른 곳으로 꺼내기).

이번 작업은 Phase 3~6을 구현하고 Phase 7(Pro Layer)의 방향을 문서로 고정했다.

## 2. 무엇이 만들어졌나

### Phase 3 — `manta context` v0 (task-20)

```bash
manta context task-1 task-4 --max-chars 6000
```

- 모델/네트워크 없음, deterministic extractive output, 출력은 `--max-chars`를 절대 넘지 않음
- **all-or-nothing**: 하나라도 조회 실패하면 stdout 0 byte + exit 1 —
  partial context는 조용히 잘못된 입력이 되어 AI 세션을 오염시키기 때문
- GUI의 Copy AI Context와 같은 core 함수(`buildContextDocument`)를 공유

### Phase 5 — 선택적 섹션 인식 (task-22)

`## Intent / Notes / Decisions / Result` 헤딩이 있으면 context 절단 시
**Result > Decisions > Intent > 기타 > Notes** 우선순위로 살리고, 살아남은
섹션은 원문 순서로 재조립한다. 섹션이 없으면 전체가 preamble — 아무것도
강제하지 않는다. (Phase 3보다 먼저 구현 순서에 넣은 이유: 절단 정책이
섹션을 알아야 두 번 만들지 않는다.)

### Phase 4 완성 — Local Workspace v1 (task-23)

- **본문 에디터**: draft 상태에서만 편집, 명시적 Save(⌘S)로만 파일 변경.
  core `updateTaskBody()`가 frontmatter를 보존하고 깨진 파일 덮어쓰기를 거부
- **⌘K command palette**: start/done/copy context + 입력 텍스트로 즉석 add
- **Copy AI Context**: on-demand action (상시 패널 아님 — cli-design 계약 그대로)

### Phase 6 — Import / Export v0 (task-24)

```bash
manta export > tasks.json        # 순수 JSON stdout (manta-tasks 번들 v1)
manta import tasks.json          # 새 id 재채번, task-3 → task-12 매핑 출력
```

- import는 쓰기 전에 번들 전체 검증 — 부분 import 없음 (`IMPORT_BUNDLE_INVALID`)
- 원본 id를 버리고 재채번: id 단조 증가 규칙 유지 + 충돌 원천 차단
- Jira/Notion/GitHub 커넥터는 이 번들의 변환기로 후속 (bridge 원칙)

### Phase 7 — Pro Layer 방향 고정 (구현 아님)

[pro-layer-plan.md](pro-layer-plan.md)에 기획만 동결:

- 무료/Pro 경계 확정 (export는 영원히 무료 — 떠나는 길이 열려 있어야 신뢰가 생긴다)
- Sync 방향: 파일 단위 sync, E2E 암호화, CRDT 거부(이해 가능한 충돌 해소 우선),
  projectId가 기기 간 식별 키
- 시작 조건 3가지(dogfooding 2주, 계약 안정 1개월, 실제 멀티 기기 불편)를
  만족하기 전까지 동결

## 3. 핵심 설계 결정

1. **context와 GUI가 같은 함수를 쓴다.** `buildContextDocument()` 하나가 CLI와
   GUI의 context를 모두 만든다. 두 표면의 출력이 갈라질 수 없는 구조가
   "AI와 사람이 같은 규칙"의 코드 형태다.

2. **all-or-nothing을 두 곳에 적용했다.** context(부분 출력 금지)와
   import(부분 쓰기 금지). 어중간한 성공은 실패보다 위험하다 — 특히 소비자가
   AI일 때, 부분 데이터는 에러 메시지 없이 잘못된 행동을 유발한다.

3. **import는 id를 보존하지 않는다.** 보존하면 충돌 처리 정책(덮어쓰기? 스킵?
   병합?)이 필요해지고 그 순간 단순함이 깨진다. 재채번 + 매핑 출력이
   설명 가능한 가장 단순한 규칙이다.

4. **GUI 에디터는 본문만 만진다.** frontmatter 편집 UI를 만들지 않은 것은
   미완성이 아니라 결정이다 — GUI에서 파일 구조를 망가뜨릴 방법을 원천
   차단하고, raw 편집은 `$EDITOR`/직접 수정이라는 정직한 경로에 맡긴다.

5. **Phase 7은 구현하지 않는 것이 결정이다.** sync는 파일 계약이 안정된 뒤에만
   안전하다. 계약 안정성의 증거(시작 조건 3가지)를 문서에 박아 충동 구현을 막았다.

## 4. 검증

- 자동 테스트 **227개** 통과 (이전 177 + 신규 50: sections/context/export-import/
  updateTaskBody/CLI 계약)
- CLI 수동 스모크: 섹션 4개짜리 task를 `--max-chars 260`으로 절단 → Notes만
  탈락하고 원문 순서 유지 확인, all-or-nothing(stdout 0 byte) 확인,
  프로젝트 A → B export/import round-trip 확인
- desktop: typecheck + electron-forge 기동 + renderer 단독 빌드 통과
- lint 통과

## 5. 전체 상태 (roadmap 기준)

| Phase | 상태 |
|---|---|
| 1. Local Linear | ✅ done |
| 2. Root SQLite | ✅ v0 |
| 3. AI Context | ✅ v0 |
| 4. Local GUI | ✅ v1 (에디터·palette·copy context) |
| 5. Lightweight History | ✅ (섹션 인식) |
| 6. Import/Export | ✅ v0 (JSON 번들, 커넥터는 후속) |
| 7. Pro Layer | 기획 동결 (시작 조건 명시) |

CLI는 13개 명령: `init add list show start done edit search context export import index help`

## 6. 남은 일 (phase 밖)

- desktop 화면 조작 수동 QA (기동 검증까지만 완료)
- GUI title 편집, 멀티 task context 조립, palette 키보드 네비게이션
- 오류 JSON 포맷 (task-11에서 의도적 제외)
- Jira/Notion/GitHub 커넥터 (manta-tasks 번들 변환기로)
- **dogfooding**: manta-doc의 tasks/를 manta CLI로 직접 관리하기 시작하는 것 —
  Pro Layer 시작 조건 1번이기도 하다
