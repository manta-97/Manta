# Task-22: 선택적 섹션 인식 (Phase 5: Lightweight History)

## 배경

Phase 5의 약속은 "섹션은 있으면 활용하고, 없어도 동작해야 한다"였다.
`manta context`(task-20)에 `--max-chars` 절단이 들어오면서 처음으로
"무엇을 버릴 것인가"라는 질문이 생겼고, 그 답이 섹션이다 — 본문에
`## Intent / Notes / Decisions / Result` 구조가 있으면 덜 중요한 것부터 버린다.

## 설계 결정

### 파서는 구조만 안다, 의미는 우선순위 테이블이 안다

`splitBodyIntoSections()`는 `## ` 헤딩으로 본문을 자를 뿐 어떤 헤딩이 중요한지
모른다. 중요도는 `sectionKeepPriority()`의 테이블 하나가 소유한다:

```
Result(0) > Decisions(1) > Intent(2) > preamble(4) > 알 수 없는 섹션(5) > Notes(9)
```

미래 AI 세션에 가장 비싼 정보는 "어떻게 결정되어 어떤 결과가 났는가"다.
Notes는 작업 중 메모이므로 가장 먼저 버린다.

### 강제하지 않는다

`manta add`는 여전히 빈 본문을 만들고, 어떤 명령도 섹션을 요구하지 않는다.
헤딩이 없는 본문은 전체가 preamble로 취급되어 그대로 동작한다.
handoff packet을 처음부터 강제하지 않는다는 roadmap 원칙 그대로다.

### 선별하되 재배열하지 않는다

예산에서 살아남은 섹션들은 **원문 순서로** 다시 조립된다. 우선순위는
무엇을 살릴지에만 쓰이고, 출력 문서의 흐름은 사용자가 쓴 순서를 유지한다.
잘려나간 것이 있으면 `… (truncated)` 마커를 남긴다.

### 레벨 2 헤딩만 경계다

`### 소제목`은 섹션을 쪼개지 않는다. 섹션 내부 구조는 섹션의 소유다.

## 검증

- task-sections 단위 테스트: 분리/preamble/레벨3 무시/빈 본문
- build-context 테스트: Notes 우선 탈락, 원문 순서 유지, 최우선 섹션 하드 절단
- 수동: 4섹션 task를 `--max-chars 260`으로 절단 → Notes만 빠지고 마커 표시 확인
