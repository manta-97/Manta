# Task-24: Import / Export v0 (Phase 6)

## 배경

Manifesto 3.6: "작업은 언제든 다른 도구로 옮길 수 있어야 한다."
roadmap Phase 6의 후보는 Jira export, Notion import, GitHub PR summary였지만,
그 커넥터들이 공통으로 딛고 설 것이 먼저 필요하다 — task 전체를 담는
자기 서술적 번들 포맷. v0는 그 포맷과 round-trip을 고정한다.

## 설계 결정

### 포맷: `manta-tasks` JSON 번들 v1

```json
{
  "format": "manta-tasks",
  "version": 1,
  "projectId": "manta_proj_...",
  "tasks": [{ "id", "title", "created", "status", "body" }]
}
```

`format`/`version`이 안에 박혀 있어 파일만 봐도 정체를 알 수 있고,
포맷이 진화해도 import가 거부/분기할 기준이 있다.

### export의 stdout은 순수 JSON이다

`manta export > tasks.json`이 그대로 동작해야 한다. 깨진 task는 번들에서
빠지되 stderr 경고로만 알리고 exit 0을 유지한다 — 경고가 JSON을 오염시키지 않는다.

### import는 원본 id를 버리고 재채번한다

가져온 task는 대상 프로젝트의 max+1부터 새 id를 받는다. 이유:
- 대상 프로젝트의 id 단조 증가 규칙(task-13)이 깨지면 안 된다
- 기존 task와의 충돌이 원천적으로 없어야 한다

대신 `task-3 → task-12` 매핑을 출력해서 추적 가능하게 한다.
title/created/status/body는 그대로 보존된다 (status는 해당 폴더로 직행).

### 검증은 쓰기 전에 끝낸다

번들 전체(format/version/각 task의 title·status·created·body)를 먼저 검증하고,
하나라도 틀리면 `IMPORT_BUNDLE_INVALID`로 통째로 실패한다.
"3개 중 2개만 import됨" 같은 어중간한 상태를 만들지 않는다 —
context의 all-or-nothing과 같은 원칙이다.

### Jira/Notion 커넥터는 만들지 않았다

계정/API 없이 검증할 수 없고, 실사용 증거도 아직 없다. 커넥터는 이 번들
포맷의 변환기로 후속 구현한다 — "Jira/Notion/GitHub는 원본이 아니라 bridge"
원칙에 따라 어떤 커넥터도 Manta의 로컬 기록을 흐리지 않는다.

## 검증

- core 테스트: export 충실성/스킵 보고, import 재채번·상태 폴더 배치·round-trip·
  불량 번들 거부(부분 import 없음 확인)
- CLI 계약 테스트 + 수동: 프로젝트 A export → 프로젝트 B import → list 일치
