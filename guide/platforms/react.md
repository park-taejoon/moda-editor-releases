# React

```tsx
import { WidgetView, useWidget } from "@moda-editor/react";
import "@moda-editor/react/styles.css";

function App() {
  const { widget, snapshot } = useWidget({ items, getItemText });
  return <WidgetView widget={widget} getItemText={getItemText} />;
}
```

| export                | 역할                                             |
| --------------------- | ------------------------------------------------ |
| `useWidget(options)` | 코어 생성 + 스냅샷 구독 (`useSyncExternalStore`) |
| `WidgetView`             | 스냅샷을 DOM 계약으로 렌더하는 컴포넌트          |

이미 만든 `WidgetCore`를 공유할 때는 `WidgetView`에 `widget`만 넘기면 된다 —
스냅샷 구독은 컴포넌트가 알아서 한다.

## 에디터

```tsx
import { EditorView, starterExtensions, useEditor } from "@moda-editor/react";

function App() {
  const { editor, state } = useEditor({
    doc: myDoc,                    // DocNode JSON
    extensions: starterExtensions(), // 스타터 스키마/커맨드
  });
  return <EditorView editor={editor} />;
}
```

`useEditor`는 `createEditor`를 감싼다 — 반환값의 `state`는 불변
`EditorState`라 `===` 비교로 렌더 필요 여부가 결정된다.
`editor.commands.toggleBold()` 등을 툴바 버튼에 연결하면 된다.
