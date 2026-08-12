# Task-1 구현: manta-pup 세팅 + Fyne 최소 실행

## 개요

로드맵 현재 초점([roadmap.md](../../../../docs/roadmap.md), [demo-fyne-jira-local.md](../../../../docs/demo-fyne-jira-local.md))의 **pup 스트림 첫 작업**이다.  
`manta-pup/`에 독립 Go 모듈 + Fyne 의존성을 붙이고, **창 하나가 뜨는 최소 앱**까지 만든다.  
이슈 CRUD·Jira import는 후속 태스크다. 이번 범위는 **개발 환경과 실행 골격**만이다.

> **실행은 작성자가 한다.** 이 문서는 세팅 절차 + 적용할 코드(diff)만 담는다.  
> 코드 레포 파일은 이 문서 작성 시점에 **수정하지 않았다**.

---

## 현재 상태 (작성 시점)

`manta-pup/` 에는 이미 다음이 있다.

| 항목 | 상태 |
|------|------|
| 독립 git | 있음 (`main`, first commit) |
| `go.mod` | `module github.com/ytlee/manta-pup`, `go 1.25.1` |
| Fyne 의존성 | **없음** (`go.sum` 없음) |
| 앱 소스 (`main.go` 등) | **없음** |
| `README.md` / `CLAUDE.md` / `.gitignore` | 있음 (스캐폴드 수준) |

**이미 된 단계(1~3)는 건너뛰고, 4번부터 하면 된다.**  
처음부터 다시 잡고 싶으면 1번부터 전부 따라가면 된다.

---

## 설계 결정

1. **패키지 루트 = `main`**
   - `go build -o bin/manta-pup .` 를 `CLAUDE.md`가 이미 가정한다.
   - 데모 단계라 `cmd/manta-pup` 분리는 YAGNI. 패키지가 커지면 그때 옮긴다.

2. **최소 창만**
   - 제목 `Manta` + 짧은 라벨 하나. 목록/에디터/Jira UI는 다음 태스크.
   - no-design: Fyne 기본 위젯만.

3. **의존성 고정 방식**
   - `go get fyne.io/fyne/v2@latest` 후 빌드로 `go.mod`/`go.sum`을 확정한다.
   - 버전 핀은 `go.sum`에 맡긴다. 문서에 특정 마이너 버전을 하드코딩하지 않는다.

4. **워크스페이스 인자는 아직 없음**
   - “연 코드 레포” 개념은 이슈 파일 I/O 태스크에서 넣는다.
   - 이번엔 GUI 툴킷·빌드 파이프만 검증.

5. **테스트**
   - GUI `ShowAndRun` 은 헤드리스 CI에 부적합 → 패키지 단위 스모크 테스트만.
   - `AppName()` 같은 순수 값이 기대와 같은지 확인. TDD 강제 아님(아래 Q 참고).

6. **`manta-repo` 비접촉**
   - 동결. import/복사 없음.

---

## 세팅 절차 (작성자가 실행)

워크스페이스 루트는 `Manta/` (형제 폴더에 `manta-pup/`, `manta-doc/` 가 있는 곳)를 가정한다.

### 0. 사전 조건

| 항목 | 확인 |
|------|------|
| Go | `go version` → **1.22+** (현재 환경 예: 1.25.1) |
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
go mod init github.com/ytlee/manta-pup
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

아래 **파일 diff** 절의 내용을 `manta-pup/` 에 적용한다.

- `main.go` (신규)
- `appmeta.go` (신규) — 테스트 가능한 앱 메타
- `appmeta_test.go` (신규)
- `README.md` (실행 절 보강)

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
- 창 제목 `Manta`, 본문에 데모 안내 라벨이 보임
- 창 닫기 시 프로세스 정상 종료

### 7. (선택) 커밋 — `manta-pup` 레포에서

```bash
cd manta-pup
git add go.mod go.sum main.go appmeta.go appmeta_test.go README.md
git status   # .idea 등은 올리지 말 것
git commit -m "$(cat <<'EOF'
Add Fyne scaffold and minimal window

Wire fyne.io/fyne/v2 and a hello window so the dogfood demo can build and run.
EOF
)"
```

---

## 완료 기준 (이 태스크)

- [ ] `fyne.io/fyne/v2` 가 `go.mod` / `go.sum`에 있다
- [ ] `go build -o bin/manta-pup .` 성공
- [ ] `go test ./...` / `go vet ./...` 성공
- [ ] 실행 시 최소 창이 뜬다
- [ ] `manta-repo` 변경 없음

**범위 밖**

- 이슈 파일 CRUD, `issues/` 스캔
- SQLite 인덱스
- Jira REST
- 워크스페이스 경로 선택 UI
- CLI 표면

---

## 파일: `manta-pup/appmeta.go` (신규)

앱 ID·표시 이름을 패키지 상수로 둔다. GUI 없이도 테스트 가능.

```diff
+package main
+
+// AppID is the Fyne application unique ID (reverse-DNS style).
+const AppID = "com.manta.fyne"
+
+// AppDisplayName is the window title and human-facing product name for this demo binary.
+const AppDisplayName = "Manta"
```

---

## 파일: `manta-pup/main.go` (신규)

파일 전체:

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

	statusLabel := widget.NewLabel("manta-pup demo scaffold — issues UI comes next")
	window.SetContent(container.NewVBox(statusLabel))
	window.Resize(fyne.NewSize(480, 320))

	window.ShowAndRun()
}
```

---

## 파일: `manta-pup/appmeta_test.go` (신규)

```diff
+package main
+
+import "testing"
+
+func TestAppDisplayName(t *testing.T) {
+	if AppDisplayName != "Manta" {
+		t.Fatalf("AppDisplayName = %q, want %q", AppDisplayName, "Manta")
+	}
+}
+
+func TestAppID(t *testing.T) {
+	if AppID == "" {
+		t.Fatal("AppID must not be empty")
+	}
+}
```

---

## 파일: `manta-pup/README.md` (갱신)

기존 소개 단락은 유지하고, **구현은 아직 비어 있다** 문장을 실행 가이드로 교체한다.

```diff
 # manta-pup
 
 Manta **dogfood 데모** 코드 홈.
 
 - Fyne GUI (no-design)
 - 연 코드 레포의 `issues/*.md` 가 source of truth
 - Jira REST: 특정 티켓 일회 import
 - `manta-repo` (Wails/CLI) 와 분리 — 폐기·이식 쉽게
 
 합의 문서: [`../manta-doc/docs/demo-fyne-jira-local.md`](../manta-doc/docs/demo-fyne-jira-local.md)
 
-구현은 아직 비어 있다. 앱 코드는 이 디렉터리에만 추가한다.
+## 요구 사항
+
+- Go 1.22+
+- CGO + 시스템 GUI 툴체인 (macOS: Xcode Command Line Tools)
+
+## 빌드 / 실행
+
+```bash
+go test ./...
+go vet ./...
+mkdir -p bin
+go build -o bin/manta-pup .
+./bin/manta-pup
+```
+
+앱 코드는 이 디렉터리에만 추가한다. `manta-repo` 는 동결이다.
```

---

## 파일: `manta-pup/go.mod` (의존성 — 명령으로 생성)

직접 버전을 박지 말고 4번 절차의 `go get` / `go build` 결과물을 사용한다.  
적용 후 대략 이런 형태가 된다 (버전 숫자는 실행 시점 latest).

```diff
 module github.com/ytlee/manta-pup
 
 go 1.25.1
+
+require fyne.io/fyne/v2 v2.x.x
+
+require (
+	// fyne transitive deps … go mod이 채움
+)
```

`go.sum` 은 `go build` / `go test` 가 생성한다. 커밋에 포함한다.

---

## 적용 순서 요약

1. `cd manta-pup`
2. `go get fyne.io/fyne/v2@latest`
3. `appmeta.go`, `main.go`, `appmeta_test.go` 생성 (위 확정본)
4. `README.md` 패치
5. `go test ./... && go vet ./... && go build -o bin/manta-pup . && ./bin/manta-pup`
6. 문제 없으면 `manta-pup` git 커밋

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
