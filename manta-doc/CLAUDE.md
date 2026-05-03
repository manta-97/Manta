# manta-doc — Documentation & Task Tracking

## Structure

```
manta-doc/
├── Manifesto.md              # 프로젝트 철학 원본 (의사결정 기준)
├── docs/                     # 설계 문서
│   ├── cli-design.md         # CLI 설계 명세
│   └── roadmap.md            # Phase 1-3 로드맵
└── tasks/                    # 태스크 추적
    ├── todo/                 # 대기 중
    │   └── task-N/
    ├── in-progress/          # 진행 중
    │   └── task-N/
    └── done/                 # 완료
        └── task-N/
```

## Task Convention

### 태스크 구조
각 태스크는 **폴더** 단위로 관리한다:
```
tasks/todo/task-8/
├── impl.md                # 구현 기획서 (필수)
└── design-decisions.md    # 설계 결정 기록 (선택)
```

### 상태 전환
태스크 폴더를 상위 디렉토리 간 이동하여 상태를 변경한다:
- `tasks/todo/task-8/` → `tasks/in-progress/task-8/` (시작)
- `tasks/in-progress/task-8/` → `tasks/done/task-8/` (완료)

### impl.md 작성 규칙
- 글로벌 `~/.claude/CLAUDE.md`의 impl.md 규칙을 따른다
- 코드 변경은 diff 형식으로 표현
- 코드 적용 완료 후 impl.md는 삭제 (커밋에 포함하지 않음)

## Manifesto

`Manifesto.md`는 Manta의 철학 원본이다. 기능이나 설계 결정 시 다음을 기준으로 판단:
- "이게 파일 기반 철학을 깨는가?"
- "이게 단순함을 해치는가?"
- "이게 AI 사용성을 떨어뜨리는가?"
