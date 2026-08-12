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

- 태스크 ID가 있으면 `manta-doc/tasks/pup/` 에서 관련 문서를 확인한다
- impl.md가 이미 있으면 해당 내용을 기반으로 구현한다
- 불명확한 부분은 **반드시 사용자에게 질문**한다

### 2. 코드 탐색

**`manta-pup/`** 만 탐색한다:
- 변경 대상 파일과 관련 코드를 이해한다
- 기존 패턴과 컨벤션을 파악한다 (`manta-pup/CLAUDE.md`)
- 다른 코드 레포는 열지 않는다

### 3. 코드 작성

`manta-pup/` 내 코드 파일을 **직접 수정**한다.

#### 준수사항
- `manta-pup/CLAUDE.md` 의 YAGNI·도메인 네이밍 규칙을 따른다
- 테스트가 필요한 변경이면 테스트도 함께 작성한다
- 빌드·테스트:

```bash
cd manta-pup
go test ./...
go vet ./...
go build -o bin/manta-pup .
```

### 4. 완료 보고

변경한 파일 목록과 변경 내용을 요약한다.
impl.md가 있었다면 "impl.md 기반 구현 완료"를 명시한다.
