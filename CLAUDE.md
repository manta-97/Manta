# CLAUDE.md

This file is a guide for Claude Code when working on this repository.

## Philosophy
> 상세: [manta-doc/Manifesto.md](manta-doc/Manifesto.md)

- **작업은 파일이다**: DB가 아닌 Markdown 파일로 존재
- **AI는 사용자다**: AI가 직접 조작할 수 있는 구조화된 인터페이스 제공
- **로컬-first**: 서버/클라우드 종속 없이 사용자가 데이터를 완전히 소유
- **단순함 우선**: 복잡한 워크플로우, 과도한 설정을 거부

## Project Structure

이 워크스페이스는 3개의 독립 git 레포로 구성된다:

```
Manta/                  # 루트 — 하네스 설정 (CLAUDE.md, .claude/, .claudeignore)
├── manta-repo/         # 코드 레포 — TypeScript monorepo (@manta/core, @manta/cli)
└── manta-doc/          # 문서/태스크 레포 — 설계 문서, 태스크 추적, Manifesto
```

- **루트 git**: 하네스 설정만 관리 (manta-repo/, manta-doc/는 .gitignore로 제외)
- **manta-repo git**: 코드 변경사항
- **manta-doc git**: 문서, 태스크, 설계 결정

각 서브 레포의 상세는 해당 레포의 `CLAUDE.md`를 참고한다.

## Development Workflow

### 태스크 기반 개발 흐름

1. **태스크 확인**: `manta-doc/tasks/todo/`에서 다음 작업 확인
2. **impl.md 작성**: 태스크 폴더에 구현 기획서 작성 (`/write-impl` 또는 `/write-impl-with-code`)
3. **구현**: `manta-repo/`에서 코드 작성 (`/write-code` 또는 직접 작성)
4. **리뷰·커밋**: 코드 리뷰 후 커밋 (`/review-commit`)
5. **태스크 상태 전환**: 완료 시 `in-progress/` → `done/`으로 이동

### 빠른 수정 (impl.md 불필요)

간단한 버그 수정, 리팩터 등은 `/write-code`로 impl.md 없이 직접 코드를 수정한다.

## Git Operations

각 레포에서 개별적으로 git 작업한다:

```bash
# manta-repo에서 커밋
git -C manta-repo add .
git -C manta-repo commit -m "message"

# manta-doc에서 커밋
git -C manta-doc add .
git -C manta-doc commit -m "message"
```

### 브랜치 컨벤션
- manta-repo: `task-N-short-description` (예: `task-7-manta-init`)
- manta-doc: `main` 브랜치에서 직접 작업 (문서 특성상)
