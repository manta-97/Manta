# Task-7: Design Decisions

## 1. 전역 설정과 프로젝트 데이터, 둘 다 필요하다

처음에는 프로젝트 안에 마커 파일 하나만 두면 충분하다고 생각했다.
그런데 "이 컴퓨터에 등록된 프로젝트가 뭐가 있지?"라는 질문에 답하려면
어딘가에 프로젝트 목록이 있어야 한다.

Electron 앱을 떠올려보면 명확해진다 — 앱을 열었을 때 프로젝트 목록을 보여줘야 하는데,
매번 디스크 전체를 뒤질 수는 없다.

그래서 두 레이어로 나눴다:

| 레이어 | 위치 | 역할 |
|---|---|---|
| 전역 | `~/Library/Application Support/manta/` | 등록된 프로젝트 목록 |
| 프로젝트 | `<project>/.manta/` + `<project>/manta/tasks/` | 마커 + 실제 작업 데이터 |

---

## 2. CLI와 Electron은 같은 전역 경로를 봐야 한다

Electron을 설치하면 CLI가 함께 설치된다. 둘은 같은 프로젝트 목록, 같은 설정을 공유해야 한다.
그런데 CLI 관례(`~/.config/manta/`)와 macOS 앱 관례(`~/Library/Application Support/manta/`)가 다르다.

**결정**: macOS 표준인 `~/Library/Application Support/manta/`를 쓴다.

이유:
- Electron은 이 경로를 기본으로 사용한다
- CLI가 별도 경로를 쓰면 두 곳을 동기화해야 하는 문제가 생긴다
- `node:os`와 `node:path`로 3줄이면 구현 가능 — `env-paths` 패키지 불필요
- 크로스 플랫폼은 나중에. 지금은 macOS만 지원한다 (YAGNI)

---

## 3. `.manta/` 디렉토리를 프로젝트 마커로 쓴다

원래 설계 문서에는 "숨김 폴더를 사용하지 않는다"고 했다.
사용자가 task 파일을 직접 열고 편집할 수 있어야 하기 때문이다.

그런데 여기서 구분이 필요하다:
- **task 파일** — 사용자가 직접 보고 편집해야 한다 → 보이는 폴더 `manta/tasks/`
- **프로젝트 마커** — 사용자가 건드릴 이유가 없다 → 숨김 디렉토리 `.manta/`

`.git/`이 정확히 이 패턴이다. 사용자가 소스 코드는 직접 편집하지만 `.git/`을 직접 건드리진 않는다.

**마커 파일(`.manta`) vs 마커 디렉토리(`.manta/`)**: 디렉토리를 선택했다.
나중에 프로젝트별 설정(`config.json` 등)이 필요해지면 그 안에 넣으면 된다.
파일이었다면 디렉토리로 마이그레이션해야 하는 상황이 올 수 있다.

---

## 4. 디렉토리 이름을 강제하지 않는다

`manta init my-tasks`로 task 디렉토리 이름을 바꿀 수 있게 했다.
이러면 discovery가 복잡해진다 — 디렉토리 이름으로 찾을 수 없으니까.

해결: `.manta/` 디렉토리가 마커 역할을 하므로 `findMantaRoot`는 `.manta/`만 찾으면 된다.
task 디렉토리 이름은 전역 `projects.json`에 `taskDirName`으로 저장된다.

항상 `manta/`로 강제하는 게 더 단순하지만, Obsidian이 vault 이름을 자유롭게 지정할 수 있는 것처럼
사용자에게 선택권을 주는 게 낫다고 판단했다.

---

## 5. `manta init`은 확인 없이 바로 실행한다

`npm init`처럼 대화형으로 할지, `git init`처럼 바로 실행할지 고민했다.

**결정**: `git init` 스타일로 바로 실행한다.

이유:
- **되돌리기가 쉽다** — `.manta/`와 `manta/` 삭제하면 끝
- **물어볼 게 별로 없다** — task dir 이름 정도인데, 기본값이 대부분 맞을 것
- **AI가 사용한다** — 대화형 프롬프트는 AI 사용성을 떨어뜨린다. Manta의 핵심 원칙("AI는 사용자다")과 충돌

---

## 6. 이미 초기화된 프로젝트에 다시 init하면?

두 가지 선택지가 있었다:
- **멱등**: `created: false`로 조용히 성공 반환
- **에러**: `ALREADY_INITIALIZED`로 명시적 실패

**결정**: `ALREADY_INITIALIZED` 에러를 반환한다.

이유:
- 사용자가 실수로 다른 프로젝트에서 `manta init`을 실행하는 것을 방지
- CLI에서는 이 에러를 yellow 메시지로 친절하게 보여주면 된다 (crash가 아님)
- 멱등으로 처리하면 "이미 초기화됐는데 또 했네"라는 정보가 숨겨진다

---

## 7. Core는 런타임 의존성 0을 유지한다

Core 패키지에 `fs-extra`나 `env-paths` 같은 외부 패키지를 넣지 않는다.
`node:fs`, `node:path`, `node:os`만 사용한다.

이유:
- Core는 CLI와 Electron 모두에서 사용된다
- 의존성이 적을수록 번들 크기가 작고 충돌 가능성이 낮다
- 필요한 기능(`mkdir recursive`, `homedir`)이 Node 내장으로 충분하다
- `env-paths` v4는 ESM-only인데 현재 프로젝트는 CJS — 호환 문제도 피한다
