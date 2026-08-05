# task-2: CLI 오류 정책

## Goal
exit code와 stderr 형식을 고정한다.

## Contract
| 분류 | stderr | exit |
|---|---|---:|
| unknown command / usage | `[UNKNOWN_COMMAND]` / `[USAGE_ERROR] ...` | 2 |
| runtime failure | `[RUNTIME_FAILURE] ...` | 1 |
| success / safe no-op | (stdout only) | 0 |

- 오류 경로에서 stdout 비움
- 사용자 입력 echo는 한 줄 정규화 + 길이 상한
