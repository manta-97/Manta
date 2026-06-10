# Task-18: `manta search <query>` 구현

## 배경

Phase 1 검색은 단순 텍스트 검색이다 (cli-design v0 계약). root SQLite가 있으면
빨라지겠지만(task-19), 검색의 **정확성 계약**은 파일 스캔 기준으로 먼저 고정한다.
인덱스는 최적화이지 의미론이 아니다.

## 설계 결정

### 대소문자 무시, title + body 전체 텍스트

`query.toLowerCase()` 포함 검사. title 매치는 그 자체로 충분한 신호라 snippet 없이,
body 매치는 **처음 매치된 줄**을 snippet으로 보여준다.

```
task-2 (done) — unrelated title
    the migration plan
```

### `--status`는 폴더 기준 필터

`--status done`이면 `done/` 폴더만 스캔한다. 상태가 폴더이므로 필터가
곧 스캔 범위 축소다 — 파싱으로 거르는 게 아니다.
유효하지 않은 status 값은 usage error(exit 2)로 막는다.

### 결과 없음은 성공이다

cli-design 계약 그대로: 결과가 없어도 exit 0, 메시지는 명확하게.

```
No tasks matched "oauth".
No done tasks matched "oauth".
```

"검색이 실패했다"와 "검색했는데 없었다"는 다른 사건이다. 전자만 비-0 exit를 받는다.

### 깨진 파일은 조용히 건너뛴다

list(표시 책임)와 달리 search는 "매치된 것"을 보여주는 명령이다. 파싱 불가 파일은
매치 여부를 판정할 수 없으므로 결과에서 제외한다. 깨진 파일의 가시성은 list가 책임진다.

## 검증

- core: title 매치(snippet 없음), body 매치(첫 줄 snippet), status 필터, 빈 결과
- CLI 통합: 전체 흐름에서 검색 + 빈 결과 exit 0
