# Task-1 구현: manta-pup 세팅 + Fyne 최소 실행

## 개요

로드맵 현재 초점([roadmap.md](../../../../docs/roadmap.md), [demo-fyne-jira-local.md](../../../../docs/demo-fyne-jira-local.md))의 **pup 스트림 첫 작업**이다.  
`manta-pup/`에 독립 Go 모듈 + Fyne 의존성을 붙이고, **창 하나가 뜨는 최소 앱**까지 만든다.  
이슈 CRUD·Jira import는 후속 태스크다. 이번 범위는 **개발 환경과 실행 골격**만이다.

> **신뢰 소스:** 레포에 커밋된 코드가 우선이다. 이 문서는 그 구현을 기록한다.  
> PR: https://github.com/manta-97/manta-pup/pull/1

---

## 현재 상태 (구현 반영)

| 항목 | 상태 |
|------|------|
| 독립 git | 있음 (`TASK-1` 브랜치, PR #1) |
| `go.mod` | `module github.com/manta-97/manta-pup`, `go 1.25.1` |
| Fyne 의존성 | `fyne.io/fyne/v2 v2.8.0` + `go.sum` |
| 앱 소스 | `main.go`, `appmeta.go`, `appmeta_test.go` |
| `README.md` / `CLAUDE.md` / `.gitignore` | 있음 |

---

## 설계 결정

1. **패키지 루트 = `main`**
   - `go build -o bin/manta-pup .` 를 `CLAUDE.md`가 이미 가정한다.
   - 데모 단계라 `cmd/manta-pup` 분리는 YAGNI. 패키지가 커지면 그때 옮긴다.

2. **최소 창만**
   - 제목 `Manta Pup` + 짧은 라벨 하나. 목록/에디터/Jira UI는 다음 태스크.
   - no-design: Fyne 기본 위젯만.

3. **앱 식별자**
   - `AppID = "com.manta.pup"` (Fyne `NewWithID`용 reverse-DNS)
   - `AppDisplayName = "Manta Pup"` (창 제목)

4. **의존성 고정 방식**
   - `go get fyne.io/fyne/v2@latest` 후 빌드로 `go.mod`/`go.sum`을 확정했다.
   - 현재 핀: `v2.8.0` (`go.sum`에 상세 해시).

5. **워크스페이스 인자는 아직 없음**
   - “연 코드 레포” 개념은 이슈 파일 I/O 태스크에서 넣는다.
   - 이번엔 GUI 툴킷·빌드 파이프만 검증.

6. **테스트**
   - GUI `ShowAndRun` 은 헤드리스 CI에 부적합 → 패키지 단위 스모크 테스트만.
   - `AppDisplayName` / `AppID` 상수 값 검증.

7. **`manta-repo` 비접촉**
   - 동결. import/복사 없음.

---

## 세팅 절차 (재현용)

워크스페이스 루트는 `Manta/` (형제 폴더에 `manta-pup/`, `manta-doc/` 가 있는 곳)를 가정한다.

### 0. 사전 조건

| 항목 | 확인 |
|------|------|
| Go | `go version` → **go.mod 와 맞는 툴체인** (현재 `go 1.25.1`) |
| C 컴파일러 | Fyne는 CGO 사용. macOS: Xcode CLT (`xcode-select -p`) |
| OS | macOS / Linux / Windows 데스크톱. 이 문서는 **macOS** 기준 |

```bash
go version
xcode-select -p   # 경로가 나오면 OK. 없으면: xcode-select --install
```

Linux라면 배포판별 OpenGL/X11 개발 패키지가 추가로 필요할 수 있다.  
공식: https://docs.fyne.io/started/

### 1. (그린필드만) 디렉터리 + git

이미 `manta-pup/` 이 있으면 **스킵**.

```bash
cd /path/to/Manta
mkdir manta-pup
cd manta-pup
git init
```

루트 `Manta/.gitignore`에 `manta-pup/` 이 있으면 하네스 git은 이 폴더를 추적하지 않는다.  
데모 코드는 **`manta-pup` 자체 git** 에서 커밋한다.

### 2. (그린필드만) Go module

이미 `go.mod` 가 있으면 **스킵** (또는 내용만 확인).

```bash
cd manta-pup
go mod init github.com/manta-97/manta-pup
# go 지시문은 go 버전에 맞게 자동 기록됨
```

### 3. (그린필드만) `.gitignore` / 문서

이미 있으면 **스킵**. 최소 `.gitignore`:

```gitignore
/bin/
*.exe
.DS_Store
**/.manta/index.sqlite
**/.manta/*.sqlite
```

### 4. Fyne 의존성 추가

```bash
cd manta-pup
go get fyne.io/fyne/v2@latest
```

`go.mod`에 `require fyne.io/fyne/v2 v…` 가 생기고, 이후 빌드 시 `go.sum`이 채워진다.

### 5. 소스 파일 배치

아래 **파일** 절의 내용을 `manta-pup/` 에 둔다 (레포 실코드와 동일).

- `main.go`
- `appmeta.go` — 테스트 가능한 앱 메타
- `appmeta_test.go`
- `README.md`

### 6. 빌드 · 테스트 · 실행

```bash
cd manta-pup

go test ./...
go vet ./...
mkdir -p bin
go build -o bin/manta-pup .
./bin/manta-pup
```

**성공 기준**

- `go test ./...` 통과
- `go vet ./...` 무오류
- 창 제목 `Manta Pup`, 본문에 데모 안내 라벨이 보임
- 창 닫기 시 프로세스 정상 종료

### 7. 커밋 — `manta-pup` 레포에서

```bash
cd manta-pup
git add go.mod go.sum main.go appmeta.go appmeta_test.go README.md
git status   # .idea 등은 올리지 말 것
git commit -m "$(cat <<'EOF'
TASK-1 Add Fyne scaffold and minimal window
EOF
)"
```

---

## 완료 기준 (이 태스크)

- [x] `fyne.io/fyne/v2` 가 `go.mod` / `go.sum`에 있다
- [x] `go build -o bin/manta-pup .` 성공
- [x] `go test ./...` / `go vet ./...` 성공
- [x] 실행 시 최소 창이 뜬다
- [x] `manta-repo` 변경 없음

**범위 밖**

- 이슈 파일 CRUD, `issues/` 스캔
- SQLite 인덱스
- Jira REST
- 워크스페이스 경로 선택 UI
- CLI 표면

---

## 파일: `manta-pup/appmeta.go`

앱 ID·표시 이름을 패키지 상수로 둔다. GUI 없이도 테스트 가능.

```go
package main

const AppID = "com.manta.pup"

const AppDisplayName = "Manta Pup"
```

---

## 파일: `manta-pup/main.go`

```go
package main

import (
	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/container"
	"fyne.io/fyne/v2/widget"
)

func main() {
	application := app.NewWithID(AppID)
	window := application.NewWindow(AppDisplayName)

	statusLabel := widget.NewLabel("manta-pup demo scaffold - issues UI comes next")
	window.SetContent(container.NewVBox(statusLabel))
	window.Resize(fyne.NewSize(480, 320))

	window.ShowAndRun()
}
```

---

## 파일: `manta-pup/appmeta_test.go`

```go
package main

import "testing"

func TestAppDisplayName(t *testing.T) {
	if AppDisplayName != "Manta Pup" {
		t.Fatalf("AppDisplayName = %q, want %q", AppDisplayName, "Manta Pup")
	}
}

func TestAppID(t *testing.T) {
	if AppID != "com.manta.pup" {
		t.Fatalf("AppID = %q, want %q", AppID, "com.manta.pup")
	}
}
```

---

## 파일: `manta-pup/README.md`

```md
# manta-pup

Manta **Pup 데모** 코드

- Fyne GUI (no-design)

## 요구사항

- Go 1.22+
- CGO

## 빌드 / 실행

    go test ./...
    go vet ./...
    mkdir -p bin
    go build -o bin/manta-pup .
    ./bin/manta-pup
```

---

## 파일: `manta-pup/go.mod` (의존성)

```go
module github.com/manta-97/manta-pup

go 1.25.1

require fyne.io/fyne/v2 v2.8.0

require (
	// fyne transitive deps … go mod이 채움
)
```

`go.sum` 은 빌드/테스트로 생성되어 커밋에 포함되어 있다.

---

## 적용 순서 요약

1. `cd manta-pup`
2. `go get fyne.io/fyne/v2@latest`
3. `appmeta.go`, `main.go`, `appmeta_test.go` 생성 (위 실코드)
4. `README.md` 패치
5. `go test ./... && go vet ./... && go build -o bin/manta-pup . && ./bin/manta-pup`
6. `manta-pup` git 커밋

---

## 다음 태스크 후보 (참고만, 이번 범위 아님)

데모 MVP 순서 제안:

1. **task-1** (본 문서): 세팅 + 최소 창 ← 지금
2. 이슈 파일 모델 + `issues/*.md` 읽기/쓰기 (frontmatter)
3. 목록 UI + 텍스트 검색 + status 필터
4. 이슈 CRUD UI
5. Jira REST 일회 import (키 지정, 최소 필드)

---

## 부록: TDD / CLAUDE.md

프로젝트 `CLAUDE.md` 들(루트, `manta-doc`, `manta-pup`, `manta-repo`)과 워크스페이스 가이드에 **TDD(테스트 주도 개발) 의무 규정은 없다.**

있는 것은 대략 다음 수준이다.

- `go test ./...`, `go vet ./...` 실행을 권장하는 명령 목록
- `/write-impl-with-code` 스킬이 impl에 테스트 코드 포함을 권장하는 것 (워크플로 강제 아님)

따라서 이 태스크는 **스모크 테스트 1~2개 + 수동 창 확인**이면 충분하다.  
후속 도메인 로직(파서, 인덱스, import)부터 테스트 비중을 올리는 편이 맞다.
