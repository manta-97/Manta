---
name: write-impl-with-code
description: |
  이 요청에 대해 `manta-doc/tasks/` 내 해당 태스크 폴더에 **기획 내용과 구현 코드를 모두** 작성한다. 코드 파일은 직접 수정하지 않는다.
  "/write-impl-with-code", "impl 코드 포함", "상세 기획" 요청에 이 스킬을 사용한다.
---

**인자**: $ARGUMENTS

---

## 실행 지침

### 1. 태스크 파악

인자에서 태스크 정보를 파악합니다:
- 태스크 ID가 있으면 (예: `task-8`): 해당 태스크 폴더를 찾는다
- 태스크 ID 없이 설명만 있으면: 적절한 태스크 폴더를 사용자에게 확인한다

```bash
ls manta-doc/tasks/todo/ manta-doc/tasks/in-progress/ 2>/dev/null
```

### 2. 요구사항 분석

구현할 내용을 파악하기 위해:
- 태스크 폴더 내 기존 문서가 있으면 읽는다
- `manta-doc/docs/` 내 관련 설계 문서를 참고한다
- `manta-repo/` 코드를 **깊이 탐색**하여 현재 구조, 패턴, 관련 코드를 이해한다
- 불명확한 부분은 **반드시 사용자에게 질문**한다

### 3. impl.md 작성

`manta-doc/tasks/<status>/task-N/impl.md` 경로에 기획 + 구현 코드를 작성합니다.

#### 형식

```markdown
# Task-N 구현

## 개요
(무엇을 왜 변경하는지 1-2문장)

## 설계 결정
(핵심 의사결정과 그 이유. 트레이드오프 포함)

## 파일: `packages/core/src/example.ts`

\```diff
 import { Something } from './something'

 class Example {
-    oldMethod(data: unknown): void {
-        const result = process(data)
-        return result
-    }
+    parseAndValidateTask(rawTaskContent: string): ParsedTask {
+        const parsedFrontmatter = parseYamlFrontmatter(rawTaskContent)
+        validateRequiredFields(parsedFrontmatter)
+        return {
+            id: parsedFrontmatter.id,
+            title: parsedFrontmatter.title,
+            status: parsedFrontmatter.status,
+            body: extractMarkdownBody(rawTaskContent),
+        }
+    }
\```

## 파일: `packages/core/src/new-file.ts` (신규)

\```diff
+import { TaskStatus } from './types'
+
+export interface ParsedTask {
+    id: string
+    title: string
+    status: TaskStatus
+    body: string
+}
\```

## 파일: `packages/core/src/example.test.ts`

\```diff
+describe('parseAndValidateTask', () => {
+    it('should parse valid frontmatter and return task object', () => {
+        // Given
+        const rawContent = '---\nid: task-1\ntitle: Test\nstatus: todo\n---\nBody'
+
+        // When
+        const result = parseAndValidateTask(rawContent)
+
+        // Then
+        expect(result.id).toBe('task-1')
+    })
+})
\```
```

#### 핵심 원칙
- **설계 결정 + 완전한 구현 코드**: `/write-impl`과 달리 실제 적용 가능한 수준의 diff 작성
- 테스트 코드도 포함
- 네이밍은 `manta-repo/CLAUDE.md`의 Naming 규칙을 엄격히 따른다
- **코드 파일은 절대 수정하지 않는다** — impl.md 안의 diff로만 표현

### 4. 완료 보고

작성한 impl.md의 경로, 변경 파일 목록, 핵심 설계 결정을 요약하여 보고합니다.
