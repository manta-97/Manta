# task-1: Go module scaffold + CLI entry

## Goal
Go 모듈과 CLI 엔트리포인트를 고정한다. TS monorepo 흔적을 남기지 않는다.

## Scope
- `go.mod`, `cmd/manta`, `internal/{core,engine,cli}` 패키지 경계
- `desktop/` 는 Wails placeholder (Electron 금지)
- `go build` / `go test ./...` 통과

## Status
Scaffold 초안은 manta-repo에 존재. help 외 명령은 미구현이 정상.
