# Demo: Fyne + repo-local issues (Jira dogfood)

> 상태: **active** (2026-08 합의)  
> 성격: 의도적 데모. 1차 코드·파일 계약 초안은 버려지거나 크게 바뀔 수 있다.  
> Manifesto 장기 방향(파일 원본, AI-usable 인터페이스, local-first)은 유지한다.  
> **전달 순서**만 바꾼다: 완벽한 CLI/Wails 전에, 직접 쓰며 파일 형식을 발견한다.

## 왜 하는가

- Wails + CLI를 “완벽하게” 쌓는 것이 혼자 개발의 병목이었다.
- 코어 계약(어떤 파일 형식이 좋은지)이 아직 가설이다. 문서로 고정하기 전에 **실사용으로 검증**한다.
- 1차 초점은 기능 완성이 아니라 **Jira에서 쓰는 티켓을 로컬(코드 옆)로 옮겨 쓰는 것**이다.

## 한 줄 요약

**파일이 원본인 최소 Fyne 앱으로 Jira 티켓을 코드 레포 옆에 두고 쓰면서, Markdown 이슈 형식을 dogfood로 발견한다. 기존 Wails/CLI 스케폴딩은 동결한다.**

## 결정 표

| 주제 | 결정 |
|------|------|
| 성격 | **C** — 데모. 형식·UI 코드는 폐기 가능 |
| Source of truth | **파일**. SQLite는 **재빌드 가능한 인덱스만** |
| 이슈 단위 | 이슈 = **Markdown 파일 하나** + YAML frontmatter |
| 위치 | **코드 레포 안** `issues/*.md`. 프로젝트 경계 = git 레포 |
| 프로젝트 구분 | 폴더 트리나 SQLite가 아니라 **어느 레포를 열었는지** |
| git | 커밋 정책은 **사용자 규약**(이슈/코드 커밋 분리 권장). 앱은 파일 write만, 자동 커밋 없음 |
| UI | **Fyne**, no-design(기본 위젯). MUI급 비주얼은 1차 목표 제외 |
| MVP | **(1) 이슈 CRUD (2) 목록/필터 (6) Jira 일회 import** |
| Import | Jira **REST 읽기 전용**, **특정 이슈 키**, 필드 최소. 양방향 sync 없음 |
| status | **자유 문자열** (Jira 값 그대로 수용) |
| id (1차) | **파일명 stem = id** (Jira 키 우선, 예: `PROJ-123.md`). 불변 UUID id는 나중에 |
| 목록/필터 | **텍스트 검색 + status 필터** (인덱스에 등장한 status 값) |
| 기존 Go/Wails/CLI | **동결** (`manta-repo`). 데모 코드는 형제 폴더 **`manta-pup`** |
| 종료·freeze | 구현 1차 끝 = MVP 동작 + 실사용. 계약 freeze = 같은 형식 고통이 반복될 때. **판단은 작성자** |

## Source of truth 규칙

1. 업무 엔티티(이슈 생성·수정·삭제·상태 변경)의 write 경로는 **항상 파일 먼저**.
2. SQLite(`.manta/index.sqlite` 등)는 목록·검색용 **투영**이다. 삭제 후 파일 트리에서 전부 재빌드 가능해야 한다.
3. 프로젝트 소속·이슈 본문·상태를 SQLite에만 두지 않는다.
4. UI 창 크기 같은 순수 앱 설정은 파일 계약 밖(로컬 설정)에 둘 수 있다.

## 디스크 레이아웃 (초안)

코드 레포를 열면 그 레포가 곧 워크스페이스다.

```text
<repo>/
  issues/
    PROJ-123.md
    MANTA-1.md
  .manta/
    index.sqlite          # optional, rebuildable; git 추적 비권장
```

- 전역 `~/manta/issues` 볼트는 1차 목표가 아니다. “코드와 가까움”이 우선이다.
- 상태는 **폴더 이동으로 표현하지 않는다**. `status` frontmatter만 사용한다.

## 이슈 파일 초안

```markdown
---
id: PROJ-123
title: 로그인 실패 시 메시지
status: In Progress
---

Description body (Markdown).
```

- frontmatter 키는 최소만: `id`, `title`, `status` (+ 필요 시 점진 추가).
- 로컬 신규 이슈도 1차는 같은 id 규칙(레포 short key + 시퀀스, 예: `MANTA-1`). 발급 위치는 구현 시 정한다.
- 장기적으로 불변 id + `jira_key` 분리(B안)가 더 나을 수 있으나, **아픔이 생기기 전에는 A안 유지**. 반쯤 B를 섞지 않는다.

## MVP 상세

### 1. 이슈 CRUD
- 생성·수정·상태 변경·닫기( status 변경으로 충분할 수 있음).
- 저장 = 해당 `issues/*.md` write.

### 2. 목록 / 필터
- 전체 목록.
- 제목·본문·id 텍스트 검색.
- status 필터 (자유 문자열; UI는 등장 값 기준).

### 6. Jira 일회 import
- 입력: base URL, 토큰, **이슈 키 하나 또는 소수**.
- 출력: `issues/<KEY>.md` 한 장.
- 가져올 필드(초최소): key→id/파일명, summary→title, description→body, status.
- 댓글·첨부·하위이슈·커스텀필드·스프린트: 없음. 나중에 추가.
- 같은 키 재import: **파일 덮어쓰기** (머지 없음).
- **양방향 동기화 없음.**

## 1차 범위 밖

- 칸반 보드, 드래그 전이
- Jira 양방향 sync
- 코멘트 스레드, 첨부, 시간 추적
- 고정 상태 머신 / 허용 status 목록
- CLI 필수 표면, Wails UI, MUI 디자인
- root 전역 SQLite (`~/.manta/manta.sqlite`) 제품화
- 프로젝트 소속을 DB에 저장

## 기존 스택과의 관계

| 경로 | 상태 |
|------|------|
| `manta-repo/` (`cmd/manta`, Wails `desktop/` …) | **동결** — 데모 중 확장하지 않음 |
| roadmap Phase 1~7 (CLI-first Local Linear …) | **일시 정지** — 데모 종료 후 재평가 |
| **`manta-pup/`** | **데모 구현 홈** — 독립 Go module + 독립 git. 폐기 가능 |

`manta-pup`을 본선과 분리하는 이유: 동결 골격과 섞이지 않게 하고, 데모 코드를 버리기 쉽게 한다.  
성공 시 계약·코드만 `manta-repo`로 이식하거나 lab을 승격한다. 실패 시 디렉터리 삭제.

TS/Electron 폐기는 유지. Go 언어 선택도 유지. 바뀌는 것은 **1차 전달 수단(Fyne + 레포 로컬 issues)** 이다.

## 종료 조건

1. **구현 1차 완료**: CRUD + 목록/필터 + 특정 키 Jira import가 동작하고, 실제 티켓을 가져와 갱신하며 쓸 수 있다.
2. **파일 계약 freeze**: 서두르지 않는다. 같은 형식 불만(필드 부족, status 난립, id 체계 등)이 반복될 때만 개정한다.
3. 캘린더 강제 종료 없음. **진행·freeze 판단은 작성자.**

데모 이후 선택지(미리 고르지 않음):

- 계약을 고정하고 CLI/AI 표면을 다시 얹기
- UI만 Wails 등으로 교체 (파일이 원본이면 UI 교체 비용이 상대적으로 작음)
- 데모 코드 폐기 후 계약만 이식

## 구현 직전 열어 둔 것

- Jira 토큰 보관 (환경변수 vs 로컬 설정 파일)
- 로컬 신규 id prefix/시퀀스 저장 위치
- frontmatter 키 최종 표기 (`title` vs `summary` 등)
- 바이너리 이름 (당분간 `manta-pup` 또는 `manta`)

## 관련 문서

- 철학: [../Manifesto.md](../Manifesto.md) (장기 신념; 이 데모는 전달 순서를 먼저 검증한다)
- 이전 스택 전환: [stack-go-wails.md](stack-go-wails.md) (Wails 경로; 데모 중 동결)
- 로드맵: [roadmap.md](roadmap.md) (데모 섹션이 현재 초점)
- 데모 코드: `../../manta-pup/` (`README.md`, `CLAUDE.md`)
- 데모 태스크 큐: `../tasks/pup/` (본선 `tasks/{todo,…}` 와 분리; pup = 새끼/치어 단계 dogfood)
