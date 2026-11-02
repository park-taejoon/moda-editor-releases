# 표 (tableFeature)

테이블 블록 — 행/열 편집, 셀 이동, 열 너비 드래그.
`starterFeatures()`에 포함된다.

```ts
editor.commands["insertTable"]!(3, 3); // rows, cols — 캐럿 자리에 삽입
editor.commands["addTableRow"]!(); // 캐럿 아래에 행 추가
editor.commands["addTableColumn"]!(); // 캐럿 오른쪽에 열 추가
editor.commands["deleteTableRow"]!();
editor.commands["deleteTableColumn"]!();
editor.commands["deleteTable"]!();
editor.commands["setTableColWidths"]!([120, 80]); // px — 각 열 셀의 colwidth attr에 기록
editor.commands["nextCell"]!(); // 다음 셀 (Tab)
editor.commands["prevCell"]!(); // 이전 셀 (Shift-Tab)
```

## 동작 모델

- `Tab`/`Shift-Tab` — 셀 사이 이동. 마지막 셀의 Tab은 행을 추가하고
  첫 셀로 간다 (Notion 관례).
- 캐럿이 표 안에 있으면 `.me-tableui` 오버레이가 표 위에 뜬다 —
  행/열 추가·삭제 버튼과 열 너비 드래그 핸들. 셀렉션이 밖으로
  나가면 숨는다. 오버레이는 `.me-editor`의 형제라 DOM↔Point 매핑을
  침범하지 않는다.
- 열 너비는 각 행 셀의 `colwidth` attr(px 정수)로 저장된다 —
  `td[width]`/`th[width]`로 렌더돼 왕복 보존되고, 드래그 핸들이
  같은 커맨드로 쓴다.

## DOM 계약

```text
table.me-table[data-me-path] > tr > td[width]? > p[data-me-path]
```

- `td` 안에 문단이 중첩된다 — 셀 안에서도 모든 블록 편집이 동작한다.
- `.me-tableui` 오버레이는 `paintOverlay` 채널로 그린다 — 바인딩
  해제 시 `destroyOverlay`가 DOM과 ResizeObserver를 정리한다.
