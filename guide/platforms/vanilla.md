# Vanilla DOM

어댑터 없이 `mountWidget`으로 직접 마운트한다 — CDN/순수 JS 환경용.

```ts
import { mountWidget } from "@moda-editor/core";
import "@moda-editor/core/styles.css";

const { widget, rootEl, destroy } = mountWidget(el, {
  items,
  getItemText: (i) => i.label,
});

widget.setSearch("사과");
destroy(); // 리스너·DOM 정리
```

`MountWidgetOptions`는 `WidgetCoreOptions`를 확장한다 — 코어 옵션에 렌더
옵션(`searchPlaceholder` 등)이 더해진다.

## 에디터

```ts
import { mountEditor, starterExtensions } from "@moda-editor/core";

const { editor, rootEl, destroy } = mountEditor(el, {
  doc: myDoc,
  extensions: starterExtensions(),
});

editor.commands.toggleBold();
destroy(); // 구독·입력 바인딩·DOM 정리
```

`MountEditorOptions`에 `editor`를 넘기면 외부 에디터를 재사용한다 —
그 경우 `destroy()`는 에디터를 파괴하지 않고 DOM만 정리한다.

### 커스텀 노드 뷰

```ts
mountEditor(el, {
  doc, extensions,
  nodeViews: { codeBlock: (props: NodeViewProps) => HTMLElement },
});
```

팩토리가 `.me-node-view` 안쪽 엘리먼트를 반환한다 — `props.content`
HTMLElement를 자기 크롬 안에 append해 편집 아울렛을 배치한다.
계약과 헬퍼는 [커스텀 노드 뷰](../features/node-views.md) 참고.
