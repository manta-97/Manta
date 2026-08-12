# manta-doc — Documentation & Task Tracking

## Stack note

- **현재 초점:** `manta-pup/` — Fyne + 레포 로컬 `issues/*.md` 데모 (Jira dogfood).
  상세: [docs/demo-fyne-jira-local.md](docs/demo-fyne-jira-local.md)
- Go. 구현은 **`manta-pup/` 만**.

## Structure

```
manta-doc/
├── Manifesto.md              # 프로젝트 철학 원본
├── docs/                     # 설계 문서
│   ├── demo-fyne-jira-local.md  # 현재 데모 합의 (우선)
│   ├── stack-go-wails.md     # 과거 스택 기록 (참고)
│   ├── cli-design.md         # CLI 설계 참고
│   └── roadmap.md            # 로드맵 (상단 = 현재 데모)
└── tasks/
    └── pup/                  # ★ 데모 태스크 (유일한 활성 큐)
        ├── todo/
        ├── in-progress/
        └── done/
```

루트의 `tasks/{todo,in-progress,done}/` 가 남아 있어도 **데모 중에는 쓰지 않는다.**  
에이전트는 **`tasks/pup/` 만** 본다.

## Task Convention (pup)

```
tasks/pup/todo/task-1/
├── task.md
├── impl.md
└── design-decisions.md    # 선택
```

상태 전환:

- `tasks/pup/todo/task-N/` → `tasks/pup/in-progress/task-N/`
- `tasks/pup/in-progress/task-N/` → `tasks/pup/done/task-N/`

### impl.md

- 코드 변경은 diff로 표현
- 적용 완료 후 impl.md는 삭제 (커밋에 포함하지 않음)
- 구현 홈: 항상 **`manta-pup/`**

## Manifesto

`Manifesto.md`는 Manta의 철학 원본이다. 기능이나 설계 결정 시:

- "이게 파일 기반 철학을 깨는가?"
- "이게 단순함을 해치는가?"
- "이게 AI 사용성을 떨어뜨리는가?"
