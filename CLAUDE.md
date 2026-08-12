# CLAUDE.md

This file is a guide for Claude Code when working on this repository.

## Philosophy
> 상세: [manta-doc/Manifesto.md](manta-doc/Manifesto.md)

- **작업은 파일이다**: DB가 아닌 Markdown 파일로 존재
- **AI는 사용자다**: AI가 직접 조작할 수 있는 구조화된 인터페이스 제공
- **로컬-first**: 서버/클라우드 종속 없이 사용자가 데이터를 완전히 소유
- **단순함 우선**: 복잡한 워크플로우, 과도한 설정을 거부

## 지금 무엇을 만드는가

**Manta Pup 데모**만 구현한다.

- 코드 홈: **`manta-pup/`** (독립 Go module + 독립 git)
- UI: Fyne, no-design
- 원본: 연 코드 레포의 `issues/*.md` (Markdown + YAML frontmatter)
- SQLite: 재빌드 가능한 인덱스만 (업무 데이터 원본 금지)
- MVP: 이슈 CRUD, 목록/검색 + status 필터, Jira REST 특정 키 일회 import
- 합의: [manta-doc/docs/demo-fyne-jira-local.md](manta-doc/docs/demo-fyne-jira-local.md)

에이전트는 **`manta-pup/` 과 `manta-doc/` 만** 다룬다.  
다른 형제 폴더는 이 단계의 구현 대상이 아니다. (본선 전환 시 사용자가 이 파일을 수정한다.)

## Project Structure

```
Manta/                  # 루트 — 하네스 설정 (CLAUDE.md, .claude/)
├── manta-pup/          # ★ 데모 코드 (구현 홈)
└── manta-doc/          # 문서 · pup 태스크
```

- **루트 git**: 하네스 설정. `manta-pup/` 은 독립 git → 루트 `.gitignore`
- **manta-pup git**: 데모 구현 변경·커밋
- **manta-doc**: 설계 문서, 태스크 추적, Manifesto

## Development Workflow

### 태스크

데모 태스크만 사용한다: `manta-doc/tasks/pup/{todo,in-progress,done}/`

1. **태스크 확인**: `manta-doc/tasks/pup/todo/`
2. **impl.md 작성**: 해당 태스크 폴더 (`/write-impl` 또는 `/write-impl-with-code`)
3. **구현**: **`manta-pup/`** 에서만 (`/write-code` 또는 직접 작성)
4. **리뷰·커밋**: `manta-pup` 또는 `manta-doc` 중 변경이 속한 레포
5. **상태 전환**: `pup/todo` → `pup/in-progress` → `pup/done`

### 빠른 수정

간단한 버그 수정·리팩터는 `/write-code`로 impl.md 없이 직접 수정한다.

## Commands (`manta-pup/`)

```bash
cd manta-pup
go test ./...
go vet ./...
mkdir -p bin
go build -o bin/manta-pup .
./bin/manta-pup
```

## Git

```bash
# 데모 코드
git -C manta-pup add .
git -C manta-pup commit -m "message"

# 문서·태스크
git -C manta-doc add .
git -C manta-doc commit -m "message"
```

### 브랜치
- `manta-pup`: `task-N-short-description` (예: `task-1-scaffold`)
- `manta-doc`: `main`에서 직접 작업 (문서 특성상)
