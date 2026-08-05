# task-7: `manta start` / `manta done`

## Goal
상태 전환 = 파일 이동.

## Rules
- start → `in-progress`
- done → `done`
- 이미 목표 상태면 no-op + exit 0
- 없는 task면 runtime failure
