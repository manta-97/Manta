# Task-19: root SQLite v0 — `@manta/engine` + `manta index rebuild/check`

## 배경

Manifesto v2가 추가한 층이다: Markdown은 장기 원본, root SQLite(`~/.manta/manta.sqlite`)는
검색·목록·GUI를 빠르게 만드는 **로컬 운영 엔진**. 핵심 제약은 하나다 —
"root SQLite는 삭제되어도 프로젝트 파일과 anchor에서 다시 만들 수 있어야 한다."

## 설계 결정

### 별도 패키지 `@manta/engine`

`@manta/core`는 런타임 무의존 원칙을 지킨다. SQLite는 네이티브 의존성
(`better-sqlite3`)이므로 cli-design의 지침대로 adapter 경계 — 별도 패키지 — 에 둔다.
의존 방향: `engine → core`, `cli → engine`. core는 engine을 모른다.

`node:sqlite`(의존성 0)도 검토했지만 Node 22.5+ 전용이라 현재 기준(Node 20)에서
탈락. Node 최소 버전을 올리는 시점에 재검토할 가치가 있다.

### 증분 갱신이 아니라 전체 재생성

`rebuildIndex()`는 `DELETE` 후 전부 다시 넣는다. 파일이 source of truth이므로
DB가 의심스러우면 지우고 다시 만드는 것이 가장 단순한 복구 경로다.
증분 동기화는 "어느 쪽이 맞는가"라는 질문을 만든다 — 지금은 그 질문 자체를 없앤다.
수백 프로젝트 규모가 되기 전까지 성능 문제도 없다.

### 발견(discovery)은 레지스트리, 식별은 anchor

rebuild는 `~/.manta/projects.json`의 경로들을 돌며 각 프로젝트의
`.manta/project.json`을 읽는다. anchor가 없으면 그 프로젝트는 `skipped`로
보고하고 계속 진행한다 — 프로젝트 하나가 사라졌다고 인덱스 전체가 실패하면 안 된다.

### 폴더 이동 재연결은 레지스트리 upsert가 처리한다

`manta index` 실행 시 **현재 위치가 Manta 프로젝트면 먼저 재등록**한다.
`registerProject()`가 `projectId` 기준으로 upsert하므로, 이동된 프로젝트는
기존 엔트리의 경로가 새 경로로 교체되고, rebuild가 `last_seen_path`를 바로잡는다.
같은 projectId가 두 경로로 늘어나는 일이 없다.

### check의 처방은 항상 하나다

`checkIndex()`는 경로/anchor/projectId/파일 해시(sha256)의 불일치를 issue 목록으로
보고한다. 불일치는 항상 "DB가 낡았다"는 뜻이고 처방은 `manta index rebuild`다.
check가 DB를 고치려 들지 않는다 — 진단과 처방의 분리.

issue 유형: `database-missing`, `project-path-missing`, `anchor-missing`,
`project-id-mismatch`, `task-file-missing`, `task-file-changed`, `task-not-indexed`.

### 스키마 (v0)

```sql
projects(project_id PK, name, last_seen_path, task_dir, indexed_at)
tasks(project_id, id, title, status, created, path, hash, body_text, PK(project_id, id))
```

cli-design의 초기 대상 중 `sections`/`context_rank`/`ui_metadata`는 뺐다 —
context 기능(task-20)과 GUI가 실제로 요구하는 시점에 추가한다 (YAGNI).

## CLI 계약

```
manta index rebuild   # Indexed 2 project(s), 17 task(s) → ~/.manta/manta.sqlite
manta index check     # Index OK (...)  |  issue 목록 + [RUNTIME_FAILURE], exit 1
```

## 검증

- engine 테스트: rebuild 카운트/전체 교체/anchor 부재 skip/폴더 이동 재연결,
  check의 5개 불일치 시나리오
- CLI 수동: rebuild → check OK → 파일 수정 → check 실패(exit 1) → rebuild → OK,
  프로젝트 폴더 `mv` 후 rebuild로 `last_seen_path` 갱신 확인
