---
name: write-code
description: |
  이 요청에 대해 CLAUDE.md의 Code Workflow(impl.md 작성) 대신 **직접 코드를 작성**한다.
  "/write-code", "코드 작성", "직접 구현" 요청에 이 스킬을 사용한다.
  간단한 버그 수정, 리팩터, 작은 기능 추가 등 impl.md가 불필요한 경우에 사용한다.
---

**인자**: $ARGUMENTS

---

## 실행 지침

### 1. 요구사항 파악

인자에서 구현할 내용을 파악합니다.
- 태스크 ID가 있으면 `manta-doc/tasks/`에서 관련 문서를 확인한다
- impl.md가 이미 있으면 해당 내용을 기반으로 구현한다
- 불명확한 부분은 **반드시 사용자에게 질문**한다

### 2. 코드 탐색

`manta-repo/` 코드를 탐색하여:
- 변경 대상 파일과 관련 코드를 이해한다
- 기존 패턴과 컨벤션을 파악한다
- 영향 범위를 확인한다

### 3. 코드 작성

`manta-repo/` 내 코드 파일을 **직접 수정**합니다.

#### 준수사항
- `manta-repo/CLAUDE.md`의 Naming, YAGNI, Testing 규칙을 따른다
- 테스트가 필요한 변경이면 테스트도 함께 작성한다
- 빌드 확인: 코드 작성 후 `npm run build` 실행
- 테스트 확인: `npm test` 실행

### 4. 검증

```bash
cd manta-repo && npm run build && npm test
```

빌드와 테스트가 통과하는지 확인합니다. 실패하면 수정합니다.

### 5. 완료 보고

변경한 파일 목록과 변경 내용을 요약하여 보고합니다.
impl.md가 있었다면 "impl.md 기반 구현 완료"를 명시합니다.