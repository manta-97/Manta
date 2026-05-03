# AI-Safe DB Access

## 개요

이 문서는 별도 제품 아이디어를 정리한 메모다.

핵심 아이디어는 단순하다:

> **AI가 DB를 볼 수 있게 하되,**
> **일반 사용자도 안전하게 설정할 수 있게 만드는 GUI + CLI 제품**

기존에도 DB를 CLI로 접근하는 방법은 많았다.
Supabase CLI, 각종 cloud DB 도구, psql 같은 방식은 예전부터 가능했다.

문제는 접근 가능 여부가 아니라
**안전하게, 쉽게, 예쁘게, 일반인이 설정할 수 있느냐**다.

---

## 문제 정의

AI에게 DB를 연결하고 싶은 사용자는 점점 많아지고 있다.

특히 다음과 같은 흐름이 자연스럽다:

* Cursor, Claude, Codex 같은 AI로 앱을 만든다
* Supabase, Neon, Railway, PlanetScale 같은 서비스를 쓴다
* AI가 schema를 읽고, 데이터를 조회하고, 디버깅에 도움 주길 바란다

하지만 실제 설정 단계에서 바로 막힌다:

* DB read-only 계정을 따로 만들어야 한다
* role / grant / privilege 개념이 어렵다
* schema별 권한을 어떻게 나누는지 모른다
* CLI나 connection string 설정이 낯설다
* AI에게 연결을 줬다가 실수로 write가 날까 무섭다

즉 문제는 “기술적으로 가능한가”가 아니라,
**일반 사용자에게 너무 어렵고 불안하다는 것**이다.

---

## 제품 가설

이 제품은 DB 클라이언트를 새로 만드는 것이 아니다.

이 제품은:

* 사람이 GUI에서 안전한 접근 정책을 만든다
* 실제 실행은 제약된 CLI가 담당한다
* AI는 그 CLI 또는 MCP를 사용한다

즉 본질은
**AI용 DB 접근을 위한 safety layer**다.

---

## 한 줄 정의

> **Give your AI safe, read-only access to Supabase in minutes.**

한국어로 풀면:

> **Supabase 같은 DB를 AI에 안전하게 연결해주는 앱**

또는

> **SQL 권한을 몰라도 AI용 read-only DB access를 몇 분 안에 만들어주는 앱**

---

## 왜 제품이 되는가

“CLI로 DB 접근” 자체는 새로운 것이 아니다.

제품이 되는 이유는 다음에 있다:

* 복잡한 권한 설정을 추상화한다
* read-only, schema-only 같은 안전 정책을 GUI에서 만든다
* AI에게 바로 줄 수 있는 연결 프로필을 만든다
* 일반 사용자가 SQL grant 없이도 설정을 끝낼 수 있다
* 예쁜 UI와 쉬운 온보딩으로 심리적 장벽을 낮춘다

사용자가 사는 것은 CLI가 아니라
**안전성과 온보딩 단순화**다.

---

## 대상 사용자

초기 타깃은 DBA가 아니다.

오히려 다음 사용자에 가깝다:

* 바이브 코더
* 인디 해커
* AI로 빠르게 SaaS를 만드는 1인 개발자
* Supabase는 쓰지만 DB 권한 모델은 익숙하지 않은 사용자

이들은 DB 이론보다
“안전하게 연결하고 바로 쓰고 싶다”는 니즈가 더 강하다.

---

## 제품 형태

형태는 Electron 기반의 별도 앱이 자연스럽다.

구조는 다음과 같다:

* GUI: 연결, 정책, 로그, 프로필 관리
* CLI: 실제 조회 실행
* Optional MCP: AI agent 연동

정리하면:

> **Electron 앱이 정책과 연결을 관리하고,**
> **제약된 CLI가 실제 DB 작업을 수행한다**

---

## 핵심 경험

이 제품은 사용자가 DB 권한 문서를 읽게 만들면 안 된다.

이상적인 첫 경험은 다음과 같다:

1. 앱 설치
2. `Connect Supabase` 클릭
3. 프로젝트 연결 또는 connection string 입력
4. `Create Safe AI Access` 클릭
5. 앱이 안전 프로필 생성
6. 사용자는 `Copy for Claude`, `Copy for Cursor`, `Start MCP` 중 하나 선택

핵심은 사용자가 다음을 직접 하지 않게 만드는 것이다:

* SQL grant 작성
* role 생성
* privilege 디버깅
* CLI 인증 설정
* 정책 파일 수동 편집

---

## 핵심 기능

초기 제품에서 중요한 기능은 다음과 같다:

* Supabase 연결
* read-only access profile 생성
* schema/table 범위 제한
* row limit 설정
* timeout 설정
* AI용 CLI profile 생성
* 실행 로그 / 감사 기록

가능하면 추가할 기능:

* MCP 서버 실행
* Claude/Cursor/Codex용 설정 복사
* 위험한 쿼리 차단
* 샘플 데이터 미리보기

---

## 안전성 모델

이 제품은 “읽기 전용처럼 보이게”만 하면 안 된다.
실제로 강제해야 한다.

안전성은 최소 3단으로 생각해야 한다:

1. GUI에서 위험한 액션을 기본적으로 노출하지 않는다
2. CLI/MCP에서 `SELECT only`, `LIMIT`, `timeout` 같은 정책을 강제한다
3. 가능하면 실제 DB 권한도 read-only로 분리한다

중요한 점은
일반 사용자가 DB 권한 설정을 어려워한다는 것이다.

그래서 제품은 두 가지 모드를 둘 수 있다:

* `Easy Mode`: 앱이 가능한 범위에서 쉽게 구성
* `Verified Safe Mode`: 실제 read-only role/user 생성까지 유도

---

## 디자인 방향

이 제품은 전통적인 DB admin console처럼 보이면 안 된다.
너무 무겁고 무섭게 느껴질 수 있다.

디자인은 다음 방향이 더 맞다:

* 친근하지만 가볍지 않은 보안 제품
* 연결 상태가 한눈에 보이는 카드형 UI
* `Read-only`, `Safe for AI`, `Schema limited` 같은 명확한 상태 배지
* 기술 용어보다 plain language 중심
* 위험한 액션은 경고보다 이해 가능한 설명 중심

핵심은 예쁨 자체가 아니라
**복잡성을 지워 보이게 만드는 것**이다.

---

## 초기 통합 우선순위

처음부터 모든 DB를 지원하면 제품이 흐려질 가능성이 크다.

초기 우선순위는 다음 정도가 적절하다:

* Supabase
* Neon
* 일반 Postgres

이유:

* 바이브 코더가 많이 쓴다
* AI와의 조합이 자연스럽다
* Postgres 계열로 시작하면 구현도 상대적으로 집중할 수 있다

초기 메시지는 넓게 가지 않는 편이 좋다.

예:

> **Supabase를 AI에 안전하게 연결하는 가장 쉬운 방법**

이런 식의 좁은 출발이 더 강하다.

---

## 포지셔닝

이 제품은 DBeaver 경쟁 제품이 아니다.
이 제품은 전통적인 DBA 툴도 아니다.

더 정확히는:

* AI 시대의 non-technical builder를 위한 DB safety layer
* AI용 read-only DB access manager
* Supabase를 AI에 안전하게 연결해주는 앱

즉, 사용자가 원하는 것은 DB를 잘 다루는 능력이 아니라
**AI에게 DB를 안전하게 열어주는 쉬운 방법**이다.

---

## 마케팅 메시지 초안

가능한 메시지 방향:

* AI한테 DB 보여주고 싶은데, 삭제할까 봐 무섭다면
* Supabase를 AI에 read-only로 연결하는 가장 쉬운 방법
* SQL 권한 몰라도 2분 만에 AI-safe DB access
* Claude, Cursor, Codex에 DB를 안전하게 연결하기

설명보다 짧은 데모가 더 강할 수 있다:

1. Supabase 연결
2. `Safe AI Access` 클릭
3. Claude에 붙여넣기
4. 조회 성공
5. 쓰기 시도 차단

---

## 이 아이디어의 본질

이 제품은 “DB를 CLI로 쓰게 해주는 앱”이 아니다.

이 제품은
**전문가만 하던 안전한 DB 접근 설정을, 일반 사용자도 몇 분 안에 끝내게 만드는 제품**이다.

따라서 핵심 가치는:

* 쉬움
* 안전성
* 예쁜 온보딩
* AI 연결 준비 완료 상태를 빠르게 만드는 것

여기까지가 현재 가설이다.
나중에 실제 구현 가능성과 시장 반응을 다시 검토해볼 수 있다.
