# Task-17: `manta edit <id>` 구현

## 배경

task 본문은 자유 Markdown이고 파일이 source of truth다. 그렇다면 "편집"이란
무엇을 만들어야 하는가? 자체 에디터? 필드별 플래그(`--title`)? 답은 더 단순했다:
**사용자의 에디터를 열어주고 비켜서기.**

## 설계 결정

### `$VISUAL` → `$EDITOR` 순으로 스폰한다

Unix 관행 그대로다. `spawnSync(editor, [args..., filePath], { stdio: 'inherit' })`로
터미널을 에디터에 넘긴다. `EDITOR="code --wait"`처럼 인자가 포함된 값을 지원하기 위해
공백으로 분리해 첫 토큰을 바이너리로 쓴다.

### 에디터가 없으면 실패하되, 파일 경로를 알려준다

```
[RUNTIME_FAILURE] Runtime failure: No editor configured. Set $EDITOR (or $VISUAL),
or edit the file directly: /path/to/manta/tasks/todo/task-3.md
```

AI 에이전트는 보통 `$EDITOR`가 없는 환경에서 돈다. 이 에러 메시지가 곧 우회로다 —
경로를 받았으니 파일을 직접 수정하면 된다. (이 메시지 때문에 stderr 절단 한도를
120→300자로 올렸다. task-11 결과 기록 참고.)

### 편집 후 검증은 하지 않는다

에디터 종료 후 frontmatter를 재검증해 경고하는 안을 검토했지만 뺐다.
파일은 사용자 소유물이고, 깨졌다면 `list`가 `(malformed task file)`로 드러내며,
git으로 되돌릴 수 있다. 저장을 막거나 잔소리하는 건 단순함을 해친다.

### 자체 출력은 없다

에디터가 정상 종료(exit 0)하면 manta도 조용히 exit 0. Unix 침묵 관례를 따른다.
에디터가 비정상 종료하면 그 종료 코드를 `RUNTIME_FAILURE`로 보고한다.

## 검증

- CLI 수동: `EDITOR` 미설정 → exit 1 + 경로 안내, `EDITOR=true` → exit 0
- 자동 테스트는 에디터 스폰 특성상 존재 확인/에러 경로 중심
