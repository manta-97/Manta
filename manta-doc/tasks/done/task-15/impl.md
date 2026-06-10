# Task-15: `manta list` / `manta show <id>` 구현

## 배경

상태가 폴더로 드러나니(task-13) 목록 조회는 원리상 `ls` 세 번이다.
하지만 사람이 매일 쓰는 화면이려면 title이 보여야 하고, 그러려면 frontmatter를 읽어야 한다.
여기서 첫 번째 설계 질문이 나온다: **깨진 파일 하나가 목록 전체를 죽여도 되는가?**

## 설계 결정

### list는 깨진 파일에 관대하다 — 단, 숨기지 않는다

`listTasks()`는 frontmatter 파싱에 실패한 파일을 빼버리지도, 전체를 실패시키지도 않는다.
`malformed: true` 플래그와 함께 목록에 남긴다. CLI는 `(malformed task file)`로 표시한다.

- 빼버리면: 사용자는 task가 사라졌다고 생각한다. 복구 단서도 사라진다.
- 전체 실패하면: 파일 하나 때문에 시스템이 멈춘다.
- 표시하면: "여기 깨진 게 있다"는 신호 자체가 복구 행동을 유도한다.

반면 `show`는 단일 task를 정확히 요구하는 명령이므로 엄격하다 —
`TASK_FILE_MALFORMED`(파싱 실패 사유 포함)로 실패한다.

### 정렬은 id 숫자 오름차순

파일 시스템의 사전순은 task-10이 task-2보다 앞에 온다. `listTaskRefs()`가
숫자 기준으로 정렬해서 돌려주고, 모든 소비자(CLI/GUI/엔진)가 같은 순서를 본다.

### show는 파일 경로를 출력한다

```
task-2 — build cli

  status:   in-progress
  created:  2026-06-10
  file:     manta/tasks/in-progress/task-2.md

<본문>
```

`file:` 줄이 핵심이다. AI가 본문을 고치고 싶으면 `show`로 경로를 얻고 파일을 직접
편집하면 된다. CLI는 편의 레이어이지 필수 경로가 아니다 (Manifesto 3.5).

### 빈 상태도 헤더를 보여준다

`todo (0)`처럼 카운트와 함께 세 섹션을 항상 출력한다. "섹션이 없음"과
"task가 없음"을 구분할 필요가 없어지고, 출력 구조가 예측 가능해진다 (AI 파싱 친화).

## 검증

- core: 상태 교차 정렬, `.gitkeep` 무시, malformed 플래그 테스트
- CLI 통합: add → list → show 흐름, `status:   todo` 표기 확인
