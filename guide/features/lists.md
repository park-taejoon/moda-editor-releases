# 목록 (listsFeature / taskListFeature)

글머리·번호·할일 목록 — PM `inputRules` + Tab/Backspace 관례 대응.
`starterFeatures()`에 포함된다.

```ts
editor.commands["toggleBulletList"]!(); // 현재 블록 ↔ 글머리 목록
editor.commands["toggleOrderedList"]!(); // 현재 블록 ↔ 번호 목록
editor.commands["toggleTaskList"]!(); // 현재 블록 ↔ 할일 목록
editor.commands["sinkListItem"]!(); // 한 단계 안으로 (Tab)
editor.commands["liftListItem"]!(); // 한 단계 밖으로 (Shift-Tab)
```

## 동작 모델

- `- `, `1. `, `[ ] ` 입력 규칙이 문단을 목록으로 변환한다.
- `Enter` — 새 항목. 빈 항목에서 Enter는 목록을 탈출해 문단으로
  돌아간다 (Notion/GDocs 관례).
- `Tab`/`Shift-Tab` — 항목을 들여쓰기/내어쓰기. 코드블록 안에서는
  코드 들여쓰기가 먼저 소비하고 여기로 폴스루된다.
- 항목 맨 앞 `Backspace` — 들여쓰기가 있으면 한 단계 리프트, 없으면
  목록을 풀어 문단으로 되돌린다.
- 할일 마커 클릭은 DOM 이벤트로 `checked` attr을 토글한다 —
  텍스트 편집과 같은 트랜잭션 경로라 undo에 잡힌다.

## DOM 계약

```text
ul/ol > li[data-me-path] > p[data-me-path]  — 문단이 li 안에 중첩
ul.me-tasks > li.me-task[data-checked]     — ::before 마커 클릭 영역
```

목록 상태는 쿼리가 아니라 DOM 구조로 표현된다 — `ul`/`ol`/
`ul.me-tasks` + `li` 중첩이 그대로 AST다.
