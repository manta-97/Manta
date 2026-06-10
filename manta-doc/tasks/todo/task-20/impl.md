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
