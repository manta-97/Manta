---
name: write-impl
description: |
  이 요청에 대해 `manta-doc/tasks/pup/` 내 해당 태스크 폴더에 **기획 문서만** 작성한다. 코드는 작성하지 않는다.
  "/write-impl", "impl 작성", "기획서 작성" 요청에 이 스킬을 사용한다.
---

**인자**: $ARGUMENTS

---

## 실행 지침

### 1. 태스크 파악

인자에서 태스크 정보를 파악합니다:
- 태스크 ID가 있으면 (예: `task-1`): `manta-doc/tasks/pup/` 아래 해당 폴더를 찾는다
- 태스크 ID 없이 설명만 있으면: 적절한 태스크 폴더를 사용자에게 확인한다

```bash
ls manta-doc/tasks/pup/todo/ manta-doc/tasks/pup/in-progress/ 2>/dev/null
```

### 2. 요구사항 분석

- 태스크 폴더 내 기존 문서가 있으면 읽는다
- `manta-doc/docs/` 내 관련 설계 문서를 참고한다
- 구현 홈은 **`manta-pup/`** 만 본다 (다른 코드 레포 탐색·수정 금지)
- 불명확한 부분은 **반드시 사용자에게 질문**한다

### 3. impl.md 작성

`manta-doc/tasks/pup/<status>/task-N/impl.md` 경로에 기획 문서를 작성합니다.

#### 형식

```markdown
# Task-N 구현

## 개요
(무엇을 왜 변경하는지 1-2문장)

## 설계 결정
(핵심 의사결정과 그 이유)

## 파일: `path/under/manta-pup/...`

\```diff
 class Example {
-    def old_method():
+    def new_method():
\```
```

#### 핵심 원칙
- **설계 결정 중심**: 단순 코드 나열이 아닌, "왜 이 방식인가"를 설명
- diff는 변경 의도를 전달하기 위한 수단 — 완전한 코드를 작성할 필요 없음
- **코드 파일은 절대 수정하지 않는다** — impl.md만 작성
- 경로·바이너리 이름은 `manta-pup` 기준

### 4. 완료 보고

작성한 impl.md의 경로와 핵심 설계 결정을 요약하여 보고합니다.
