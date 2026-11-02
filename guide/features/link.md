# 링크 (linkFeature)

`link` 마크 — 선택 텍스트에 URL을 걸거나 해제한다.
`starterFeatures()`에 포함된다.

```ts
editor.commands["setLink"]!("https://…"); // href 생략 시 prompt
editor.commands["unsetLink"]!(); // 캐럿/선택 위 링크 해제
```

단축키 `Mod-k`가 `setLink`에 바인딩돼 있다.

## 동작 모델

- `setLink`는 두 가지로 동작한다 — 선택이 있으면 범위에 `link`
  마크를 걸고, 캐럿(축소 선택)이면 `storedMarks`로 대기시켜 다음
  타이핑이 링크가 되게 한다 (GDocs 관례).
- 선택 텍스트 위에 `https?://` URL을 붙여넣으면 텍스트를 덮지 않고
  링크로 감싼다 — `insertFromPaste` 훅이 URL 패턴을 감지한다.
  코드 블록 안에서는 순수 텍스트 붙여넣기가 먼저 소비한다.
- 링크 위 캐럿의 액션 칩(열기·편집·제거)은 제품 영역 — 데모
  5개 앱이 `.me-linkcard` DOM 계약으로 구현한다. 제거는 텍스트를
  보존하고 마크만 뺀다.

## DOM 계약

```text
a[data-me-path][href] — 마크가 최외곽 요소다
.me-linkcard        — 링크 위 액션 칩 (데모 제품 영역 오버레이)
```

- `href`는 `attrs.href`에 들어간다 — 직렬화/Markdown 왕복이
  `[텍스트](href)`를 쓴다.
- 액션 칩은 오버레이라 `data-me-path` 트리를 오염시키지 않는다.
