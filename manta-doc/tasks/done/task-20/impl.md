# Task-20: `manta context` v0 (기획 예정)

## 한 줄 정의

완료/진행 중 task 파일들을 미래 AI 세션의 입력으로 다시 꺼내는 명령.

```bash
manta context task-1 task-4 --max-chars 6000
```

## 고정된 v0 계약 (cli-design.md)

- 모델 호출 없음, 네트워크 없음 — 로컬 task 파일만 읽는다
- deterministic extractive output
- `--max-chars`는 문자 수 기준
- 하나라도 task 조회에 실패하면 **전체 실패** (partial context 출력 금지)
  - 예: `task-999` 부재 시 exit 1, stderr `TASK_NOT_FOUND: task-999`, stdout 비움

## 기획 시 결정할 것

- 출력 Markdown 구조 (task 간 구분, frontmatter 메타 포함 여부)
- `--max-chars` 초과 시 절단 전략 (task 단위 우선순위? 본문 뒤부터?)
- `## Intent / Notes / Decisions / Result` 섹션이 있을 때의 가중치 (Phase 5 연계)
- `--for pr-review` 같은 목적 프리셋을 v0에 넣을지 (제안: 빼기 — YAGNI)
- root SQLite(`body_text`, 향후 `context_rank`)를 후보 추출에 쓸지, v0는 파일 직접 읽기로 갈지

## 토대 (이미 구현됨)

- `readTask()`가 INVALID/NOT_FOUND/DUPLICATE/UNREADABLE/MALFORMED를 구분해 돌려준다
  → "하나라도 실패하면 전체 실패" 계약을 그대로 조립할 수 있다
- CLI 오류 정책(task-11)이 exit code와 stderr 형식을 이미 소유한다

---

## 구현 결과 (2026-06-10)

v0 계약 그대로 구현 완료. 기획 시 결정하기로 했던 항목들의 결론:

- **출력 구조**: `# Manta Context` 제목 아래 task마다 `## <id> — <title>` + status/created
  메타 + 본문. task 순서는 인자로 준 순서를 그대로 따른다.
- **절단 전략**: 헤더(제목/메타)는 항상 포함. 남는 예산을 task별로 균등 배분하되
  앞 task가 안 쓴 예산은 뒤로 넘긴다(단일 패스, 결정적). 본문이 예산을 넘으면
  섹션 단위로 선별한다 — task-22(Phase 5) 참고. 최종 출력은 어떤 경우에도
  `--max-chars`를 넘지 않는다 (하드 캡).
- **섹션 가중치**: Result > Decisions > Intent > preamble/기타 > Notes 순으로 살린다.
  살아남은 섹션은 원문 순서로 재배열 — 선별은 하되 문서 흐름은 유지.
- **`--for pr-review`**: 뺐다 (YAGNI). 목적 프리셋은 실사용 증거가 생기면.
- **root SQLite 사용 여부**: v0는 파일 직접 읽기. "로컬 task 파일만 읽는다" 계약과
  정확히 일치하고, 수십 task 규모에서 성능 문제가 없다.
- **stderr 형식**: cli-design 초안의 `TASK_NOT_FOUND: task-999` 대신 task-11의
  통합 정책(`[RUNTIME_FAILURE] Runtime failure: Task not found: task-999`)을 따른다.
  all-or-nothing(실패 시 stdout 0 byte)은 계약 그대로다.

GUI의 Copy AI Context(task-23)도 같은 `buildContextDocument()`를 쓴다 —
CLI와 GUI가 같은 context를 만든다.
