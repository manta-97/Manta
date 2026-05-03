# Task-13: 태스크 디렉토리를 폴더 기반 상태 모델로 전환

## 배경

`cli-design.md`의 초안은 이런 구조였다.

```
manta/
└── tasks/
    ├── task-1.md       ← frontmatter의 status: todo
    ├── task-2.md       ← frontmatter의 status: in-progress
    └── task-3.md       ← frontmatter의 status: done
```

파일은 한 폴더에 평평하게 쌓이고, 상태는 frontmatter의 `status` 필드가 들고 있다. 자연스러운 선택처럼 보였다. 그런데 `manta add`를 기획하려고 Phase 1의 다른 명령어(`list`, `show`, `start`, `done`, `edit`)가 이 구조 위에서 어떻게 돌아가야 하는지를 살펴보면서 이런 장면들이 연달아 떠올랐다.

- AI가 "done된 태스크 목록"을 보려면 `tasks/` 전체를 읽고, 각 파일의 YAML frontmatter를 파싱해서 `status: done`만 골라낸다.
- `manta start task-3`는 파일을 열어 `status: todo`를 찾고 `status: in-progress`로 치환한 뒤 다시 쓴다.
- 사람이 `ls tasks/`로는 어느 게 진행 중인지 알 수 없다. 파일을 하나씩 열어보거나 `grep -l` 같은 걸 쓴다.

각각은 사소한 비용이지만 합쳐놓고 보면, Manifesto의 두 원칙에 걸린다. "AI는 사용자다"인데 상태 질의가 파싱 연산이 되고, "단순함 우선"인데 상태 전환이 읽기-수정-쓰기 사이클이 된다.

반면 `manta-doc` 저장소 자체는 **폴더 기반**으로 돌아가고 있다.

```
tasks/
├── todo/task-11/impl.md
├── in-progress/
└── done/task-10/impl.md
```

상태를 `mv`로 바꾸고, `ls tasks/done/`으로 목록을 본다. 이 저장소의 개발자가 본인이 쓰기 위해 본능적으로 고른 구조가 이쪽이라는 건 설득력 있는 증거다. Dogfooding이 이미 답을 내놓은 셈이다.

그래서 한 번 돌아서기로 한다. 플랫 + frontmatter가 아니라 **폴더 = 상태**로 간다. 상태 전환은 `mv`, 상태 질의는 `ls`. frontmatter에서 `status`는 빠진다.

단, 이 전환은 `add` 이후 Phase 1 명령어 전부가 딛고 설 계약이다. 그래서 **`add` 구현에 앞서** 계약을 먼저 고정해두는 게 옳다. 이번 태스크가 하는 일이 그것이다.

**목표**: `cli-design.md`를 새 구조로 다시 쓰고, `manta init`이 세 상태 폴더를 미리 만들어두도록 조정한다. `add` 구현은 task-14로 분리한다.

## 새 구조

```
my-project/
├── .manta/                       # 프로젝트 마커 (변경 없음)
└── manta/                        # 사용자 경로 (변경 없음)
    └── tasks/
        ├── todo/
        │   ├── .gitkeep
        │   └── task-1.md
        ├── in-progress/
        │   ├── .gitkeep
        │   └── task-2.md
        └── done/
            ├── .gitkeep
            └── task-3.md
```

파일 포맷 (status 필드 제거):

```markdown
---
id: task-1
title: CLI 프로토타입 만들기
created: 2026-04-19
---

본문은 비어있거나 자유 서술.
```

상태 전환:

- `manta start task-3` → `mv tasks/todo/task-3.md tasks/in-progress/task-3.md`
- `manta done task-3`  → `mv tasks/in-progress/task-3.md tasks/done/task-3.md`

생성:

- `manta add "제목"` → 항상 `tasks/todo/task-N.md`에 쓴다.

## 설계 결정

### 왜 상태를 폴더로 옮기는가

결국 "단일 진실원천이 어디인가"의 문제다. 플랫 + frontmatter 안에서는 **폴더 위치**와 **frontmatter 필드**가 같은 정보를 두 번 말할 수 있다 (실제로 manta-doc은 이미 폴더로 상태를 말하고 있는데, 만약 frontmatter에 `status`도 있었으면 드리프트가 생겼을 것이다).

폴더를 진실원천으로 두면 모든 연산이 파일 시스템 기본 동작으로 환원된다.

| 연산 | 플랫 + frontmatter | 폴더 = 상태 |
|---|---|---|
| "todo 목록" | tasks/ 읽고 frontmatter 파싱해서 필터 | `ls tasks/todo/` |
| "done으로 옮기기" | 파일 열기 → status 필드 치환 → 쓰기 | `mv` |
| "태스크 위치" | `grep -l "status: done" tasks/*.md` | 폴더가 곧 답 |

AI 관점에서 이 차이는 특히 크다. `mv`는 쉘/Node의 기본 연산이고 실패 모드도 단순하다(없다/권한). frontmatter 파싱·재직렬화는 YAML 라이브러리 선택·필드 순서 유지·공백 보존 같은 자잘한 이슈를 데리고 온다. 우리는 core에 YAML 의존성을 **들이지 않기 위해** 직렬화를 수제로 할 계획인데, 그 수제 코드가 상태 변경마다 다시 실행된다. 폴더 이동은 그 전체를 지워버린다.

Manifesto 3장 "AI는 사용자다"는 여기서 작동한다. AI가 `ls tasks/in-progress/`만으로 상태 질의를 완결할 수 있다면, Manta CLI를 거치지 않고도 시스템이 일관된다. CLI는 **편의 레이어**가 되지, 필수 경로가 아니다.

### 왜 frontmatter에 `status` 필드를 남기지 않는가

"그래도 frontmatter에 `status: done`이 있으면 파일 단독으로 상태를 알 수 있으니 편하지 않나?"라는 반론이 있을 수 있다. 답은 **드리프트를 만들기 때문에 안 된다**이다.

누군가(사람이든 AI든) 파일을 `mv`로 옮겼는데 frontmatter는 그대로라면, 두 출처가 다른 이야기를 한다. 그 순간부터 "어느 쪽을 믿지?"의 질문이 생기고, `manta` 명령어마다 둘을 동기화하는 로직을 갖춰야 한다. 이건 정확히 단일 진실원천 원칙이 피하고 싶은 상황이다.

그래서 `status` 필드는 없다. 파일이 어디 있는지가 상태다.

### 왜 `id` 필드는 남기는가

`id`도 파일명과 중복이지만 제거하지 않는다. 이유가 다르다.

- 파일 본문을 단독으로 읽는 순간(`cat tasks/todo/task-3.md`, 이메일 첨부, 검색 결과 스니펫)에도 "이게 task-3이다"가 자기 서술되어야 한다.
- 파일명이 실수로 바뀌어도 본문의 `id`로 원래 ID를 회복할 수 있다. 파일명이 rename돼서 "누락된 task-3"처럼 보이는 상황에서 복구 단서가 필요하다.
- 비용이 한 줄이다.

드리프트 위험은 있지만(파일명과 `id`가 달라질 수 있음), `status`와 달리 `id`는 **변하지 않는 식별자**다. 파일을 옮겨도 ID는 그대로다. 상태처럼 자주 변경되는 필드가 아니므로 실수 가능성이 훨씬 낮다.

`created`도 같은 논리로 유지: 생성 시점에 고정, 이후 불변.

### 왜 `init`에서 세 폴더를 미리 만드는가

두 가지 대안이 있다.

1. **Eager**: `init` 시점에 `todo/`, `in-progress/`, `done/` 세 개 다 만든다. 각각 `.gitkeep` 한 줄.
2. **Lazy**: 필요해지는 시점에 만든다. 처음엔 `tasks/`만, 혹은 `todo/`만.

Lazy가 겉으로는 단순해 보인다. 빈 폴더가 없다. `.gitkeep`도 필요 없다. 하지만 lazy는 **존재 여부의 모호함**을 모든 하류 명령어에 떠안긴다.

AI가 `ls tasks/in-progress/`를 쳤을 때 "No such file or directory"가 돌아오면 두 가지 해석이 가능하다.

- "프로젝트는 초기화됐지만 아직 in-progress인 태스크가 없음"
- "프로젝트가 제대로 초기화되지 않음"

둘은 대응이 다르다(하나는 빈 목록 반환, 하나는 init 안내). lazy 구조는 이 분기를 `list`·`show`·`start`·`done` 각각이 자기 방식으로 처리하게 만든다. Eager는 이걸 한 줄로 지운다. "in-progress/는 init 이후 언제나 존재한다"가 시스템 불변식이 된다.

빈 폴더를 git에 커밋하기 위한 `.gitkeep`은 관행적 비용이다. 세 파일, 각 0바이트. Eager 쪽이 압도적으로 이득이다.

### 왜 state machine을 느슨하게 두는가

"엄격"한 대안은 이렇게 생각해볼 수 있다.

- `start`는 `todo → in-progress`만 허용. `done/`이나 `in-progress/`에 있는 태스크에 `start`를 치면 에러.
- `done`은 `in-progress → done`만. `todo`에서 바로 `done`은 금지.

들어보면 합리적으로 들리지만 실제 사용 장면을 그려보면 곧장 불편해진다.

- done 처리한 태스크를 리오픈하고 싶다 → `manta start task-3`이 안 먹힌다. 수동 `mv`를 해야 한다.
- todo에 있는 걸 바로 done으로 처리하고 싶다 (예: "아, 이거 이미 다 된 거였네") → start를 먼저 치고 done을 치라고? 의미 없는 의례.

그래서 **느슨 + 최소 가드**로 간다. 구체적으로:

- **존재 확인**: 지정한 task-N이 세 폴더 어디에도 없으면 stderr 에러, exit 1.
- **Idempotent**: 이미 목표 폴더에 있으면 no-op + 안내 메시지, exit 0. 파일을 건드리지 않는다.
- **그 외 전환은 자유**: `done` 폴더에 있는 태스크에 `manta start`를 치면 그냥 `in-progress/`로 옮긴다.

Manifesto 5장 "복잡한 워크플로우 거부"에 정확히 맞는다. 단방향 흐름을 강제해서 얻는 "정확성"은 실수 방지보다 주로 유연성을 희생한다. 게다가 사용자는 어차피 `mv`로 우회할 수 있어서 강제는 말뿐이다.

실수 방지는 다른 수단으로 커버한다. `add`가 `todo/`에만 쓴다는 단방향 진입점 계약, 그리고 `list`가 상태별로 깔끔하게 보여주는 가시성. 이 두 개면 대부분의 "실수로 done 처리"를 예방한다.

### 왜 ID 채번은 세 폴더 전체를 스캔하는가

`add`는 항상 `todo/`에 쓰지만, **채번은 세 폴더를 모두 본다**. 이유는 단순하다.

- `todo/`에 task-1, `done/`에 task-2가 있고 `todo/`만 스캔하면 다음 번호가 task-2. 충돌.
- 삭제된 task-5의 ID를 재사용하면 과거 커밋·기억·외부 링크가 엉뚱한 태스크를 가리키게 된다.

규칙:

- 세 폴더 `todo/`, `in-progress/`, `done/`에서 `task-(\d+)\.md` 패턴 매치 → 숫자 최대값 + 1.
- 갭 재사용 없음. task-3을 삭제해도 다음은 여전히 max+1.

이 규칙은 task-14(`add`)에서 실제로 구현되지만, 계약은 여기서 고정한다. 하류 명령어가 ID 공간의 단조 증가를 가정할 수 있도록.

### 단일 사용자 CLI 전제

Manifesto 1장 "로컬-first"와 "1 repo = 1 Manta"가 말하는 전제를 그대로 유지한다. 두 사람이 동시에 `manta add`를 치는 상황은 이 시스템의 대상이 아니다. 따라서:

- 채번 시 파일 존재 체크 한 번으로 경합 제어 충분. 락 불필요.
- `mv`의 atomicity는 동일 파일 시스템 내에서 POSIX가 보장. 그 이상 필요 없음.

이 가정이 깨지는 시점이 오면(팀 사용, 동기화 등 Phase 3 영역) 그때 다시 본다.

## 변경 파일

### `manta-doc/docs/cli-design.md`

가장 큰 편집이 여기다. 구체적으로:

- **디렉토리 구조 다이어그램**: 플랫 파일 목록을 세 폴더 구조로 교체.
- **작업 파일 포맷 섹션**: `status` 필드 제거, 예시 frontmatter를 id/title/created 3필드로.
- **작업 상태 모델 섹션**: "상태 = 폴더 위치"임을 명시. 상태별 경로 표 추가.
- **state machine 섹션 신설**: 전환 규칙(존재 확인, idempotent, 자유 전환), ID 채번 규칙.
- **명령어 표의 start/done 설명**: "파일을 해당 폴더로 이동" 문구로 수정.
- **설계 근거**: "숨기지 않는다", "위치 선택 가능"에 **"상태는 폴더로 드러낸다"** 추가.

cli-design.md는 Phase 1 명령어 전체의 계약이 되므로, 이번에 최대한 견고하게 써둔다. 가변 결정(정렬 규칙, 편집 방식 등)은 각 태스크로 미루고 여기서는 **구조·포맷·전환만** 고정한다.

### `manta-repo/packages/core/src/init.ts`

현재 `initializeMantaProject()`는 `.manta/`와 `<taskDir>/tasks/`를 만든다. 여기에 추가:

```ts
// 의사코드 — 실제 구현은 task-13 적용 시
const statusDirs = ['todo', 'in-progress', 'done'];
for (const statusDir of statusDirs) {
  const dirPath = path.join(tasksDirPath, statusDir);
  await fs.mkdir(dirPath, { recursive: true });
  await fs.writeFile(path.join(dirPath, '.gitkeep'), '');
}
```

- `mkdir recursive`는 이미 존재해도 실패하지 않음. 재-init 안전.
- `.gitkeep`은 0바이트 빈 파일.
- 에러는 기존 `MantaErrors` 패턴을 따른다 (권한 문제 등).

### `manta-repo/packages/core/src/init.test.ts`

assertion 추가:

- 세 서브폴더가 존재한다: `tasks/todo/`, `tasks/in-progress/`, `tasks/done/`.
- 각 폴더에 `.gitkeep`이 존재하고 크기가 0.
- 재-init 시 기존 `.gitkeep`은 그대로 유지 (덮어쓰지 않음).

마지막 항목이 중요하다. 누군가 `.gitkeep`을 다른 용도로 바꿔 썼다면 덮어쓰지 않아야 한다. `writeFile`은 기본적으로 덮어쓰므로, **파일이 이미 존재하면 건너뛰는** 로직을 따로 구현해야 한다. 이 요구사항은 impl 단계에서 `fs.access` → catch-only-create 패턴으로 처리한다.

### `manta-repo/packages/cli/src/commands/init.ts`

현재 출력:

```
Initialized Manta project at <path>
  marker:  .manta/
  tasks:   manta/tasks/
```

이후:

```
Initialized Manta project at <path>
  marker:  .manta/
  tasks:   manta/tasks/{todo,in-progress,done}/
```

`{todo,in-progress,done}` 브레이스 표기는 셸 glob 스타일로 읽혀서 사람이 바로 이해한다. 출력의 가독성 때문에 선택했지 구현이 브레이스 확장을 하는 건 아니다.

### `manta-repo/packages/cli/src/help/command-registry.ts`

`init` 엔트리의 문구 소폭 조정:

- `summary`: 그대로. "Initialize a Manta project in the current directory"는 정확하다.
- `examples`: 그대로 (`manta init`, `manta init docs`).
- 변경 없음도 가능. 바뀐 건 내부 디렉토리 구조뿐이고, help의 외부 계약(이름, 인자, 예시)은 그대로다.

루트 footer도 현재대로 두는 것을 권장한다.

```
Tasks live under <task-dir>/tasks/ (default: manta/tasks/).
Status model: todo → in-progress → done.
```

"상태 모델"은 여전히 맞고, `<task-dir>/tasks/` 아래에 있다는 것도 맞다. 굳이 `{todo,in-progress,done}`까지 풀어 쓰면 footer가 길어진다. 세부 구조는 `manta help init` 같은 상세 경로에서 본다.

### `manta-doc/docs/roadmap.md` (선택)

Phase 1 완료 기준에 한 줄:

> - 상태 모델이 폴더 구조로 드러난다

넣을지 여부는 impl 단계에서 판단. 길어지지 않는다면 추가.

## 호환성

현재 Manta는 출시 전이고 실제 사용자 0명으로 가정한다. 이전에 `manta init`으로 생성된 플랫 구조 `.manta/` 프로젝트가 있다면 재-init을 돌리거나 세 폴더를 수동으로 만드는 것으로 대응한다. 자동 마이그레이션 경로는 제공하지 않는다 — 지금 단계에서 유지보수 비용만 부른다.

이 결정은 Phase 3 즈음에 "이전 사용자" 개념이 생기면 다시 검토한다.

## 검증

### 단위 테스트 (`init.test.ts`)

- `initializeMantaProject` 호출 후 세 서브폴더 존재 확인.
- 세 `.gitkeep` 파일 존재 + 크기 0.
- 재-init 시 기존 파일들이 덮어써지지 않음 (사용자가 `.gitkeep`에 뭘 썼을 경우를 상정한 smoke check).
- `ALREADY_INITIALIZED` 분기는 기존대로 동작.

### 수동 시나리오

```bash
cd $(mktemp -d)
node /path/to/packages/cli/dist/index.js init
tree manta/
# 기대:
# manta
# └── tasks
#     ├── done
#     │   └── .gitkeep
#     ├── in-progress
#     │   └── .gitkeep
#     └── todo
#         └── .gitkeep
```

```bash
node /path/to/packages/cli/dist/index.js init docs
ls docs/tasks/
# todo  in-progress  done
```

### task-12 QA 영향

task-12의 qa.md 시나리오 S-9, S-10은 `init` 실행 후 `.manta/`와 `tasks/` 생성만 확인한다. 이 태스크로 **디렉토리 구조 기대값이 바뀌므로** qa.md도 함께 수정해야 한다.

수정 범위:

- S-9 기대 출력: `ls -la .manta manta/tasks` → `tree manta/tasks`로 교체. `todo/`, `in-progress/`, `done/` 세 폴더와 `.gitkeep` 확인 추가.
- S-10도 동일하게 `docs/tasks` 아래 세 폴더 확인.

qa.md는 manta-doc에 있으므로 이번 impl 적용 시 함께 커밋한다. 분량은 많지 않다.

### 빌드/린트

- `npm run build` (core 먼저)
- `npm test`
- `npm run lint`

## 다음 태스크 연결

이 태스크가 끝나면 **파일 시스템 계약**이 확정된다:

- 파일 포맷: `id`, `title`, `created` (status 없음)
- 저장 위치: `<task-dir>/tasks/{todo,in-progress,done}/task-N.md`
- 상태 전환: `mv`, 느슨 + 최소 가드
- ID 채번: 세 폴더 전체 스캔, max + 1, 갭 재사용 없음

다음 태스크들이 이 위에서 순서대로 얹힌다.

- **task-14 `manta add "제목"`**: core에 `createTask()` 도입. todo/에만 쓴다. 채번 구현.
- **task-15 `manta list`**: 세 폴더 스캔 후 상태별 그룹핑 출력. 정렬 규칙은 그 태스크에서.
- **task-16 `manta show <id>`**: 단일 파일 렌더. 세 폴더 중 어디 있든 찾는다.
- **task-17 `manta start <id>` / `manta done <id>`**: `mv`. 느슨 + 최소 가드.
- **task-18 `manta edit <id>`**: `$EDITOR` 스폰 또는 `--title` 플래그 — 방식 선택은 그 태스크에서.

task-11(CLI 오류/exit code 정책)과 task-12(task-10 QA)는 이 태스크와 독립이다. task-12 qa.md는 위에서 언급한 대로 이번에 함께 갱신한다.
