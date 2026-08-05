# task-3: 폴더 상태 모델 + project anchor

## Goal
`internal/core`에 파일 계약을 구현한다.

## Contract
- status = 폴더 위치 (`todo` / `in-progress` / `done`)
- frontmatter: `id`, `title`, `created` only (no `status` field)
- project anchor: `.manta/project.json` (`projectId`, `schemaVersion`, `createdAt`, `taskDir`)
- task path: `{taskDir}/tasks/{status}/task-N.md`
