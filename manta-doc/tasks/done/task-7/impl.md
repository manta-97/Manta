# Task-7: `manta init [path]` 구현

## 개요

Manta CLI의 첫 번째 실제 명령어. `manta init`은 3가지를 한다:

1. **프로젝트 마커** — `.manta/` 디렉토리 생성 (discovery용, `.git/`과 같은 역할)
2. **작업 디렉토리** — `manta/tasks/` 생성 (보이는 폴더, 사용자가 직접 접근)
3. **전역 프로젝트 등록** — `~/Library/Application Support/manta/projects.json`에 등록

### 동작

```
manta init           → .manta/ + manta/tasks/ 생성, 전역 등록
manta init my-tasks  → .manta/ + my-tasks/tasks/ 생성, 전역 등록
```

- 이미 초기화된 경우 → "already initialized" 메시지 (멱등)
- 대상 경로가 파일이면 → 에러

### 생성되는 구조

```
~/Library/Application Support/manta/     ← 전역 (CLI + Electron 공유)
└── projects.json                         # 등록된 프로젝트 목록

my-project/                              ← 프로젝트
├── .manta/                               # 프로젝트 마커 (discovery용)
├── manta/                                # 작업 데이터 (기본 경로, 변경 가능)
│   └── tasks/
└── src/
```

### 설계 결정

- **`.manta/` 디렉토리**: `.git/`처럼 프로젝트 루트의 마커 역할. `findMantaRoot`가 이걸 찾아 올라간다.
- **`manta/tasks/`는 보이는 폴더**: 원래 설계 원칙("숨기지 않는다") 유지. 사용자가 직접 열고 편집 가능.
- **전역 경로**: `~/Library/Application Support/manta/` (macOS). `node:os`와 `node:path`만으로 구현 (env-paths 불필요). Electron과 CLI가 같은 경로를 바라본다.
- **Core는 런타임 의존성 0**: `node:fs`, `node:path`, `node:os`만 사용.
