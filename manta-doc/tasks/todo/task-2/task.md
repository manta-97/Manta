# task-2: CLI 종료·출력 계약 (정석)

## Goal
Unix/12-factor 관례에 맞는 exit code와 stdout/stderr 규칙을 고정한다.

## Contract

| exit | 의미 |
|---:|---|
| 0 | 성공, 빈 검색 결과, idempotent no-op |
| 1 | 실행 실패 (없음, I/O, 잘못된 데이터 등) |
| 2 | 사용법 오류 (unknown command, 잘못된 옵션/인자 개수) |

- stdout: 성공 시 결과 데이터만
- stderr: 사람용 `Error: ...` 메시지 (대괄호 `[CODE]` 프로토콜 없음)
- 실패 시 결과 데이터를 stdout에 쓰지 않음
- `--help` / `--version` 경로 확보

상세: [docs/cli-design.md](../../../docs/cli-design.md) — “CLI 출력·종료 계약”
