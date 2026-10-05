# Svelte 5

```svelte
<script lang="ts">
  import { WidgetView, createWidgetStore } from "@moda-editor/svelte";
  import "@moda-editor/svelte/styles.css";

  const store = createWidgetStore({ items, getItemText });
  // $store → 스냅샷, store.widget → 액션
</script>

<WidgetView widget={store.widget} {getItemText} />
```

| export                    | 역할                               |
| ------------------------- | ---------------------------------- |
| `createWidgetStore(options)` | 코어 생성 + `readable` 스토어 래핑 |
| `toWidgetStore(widget)`         | 외부 코어를 스토어 계약으로 래핑   |
| `WidgetView`                 | DOM 계약 렌더 SFC                  |

keyed `{#each}`는 고유 키가 필요하다 — 키 함수를 거쳐 중복/undefined를
피한다.

## 에디터

```svelte
<script lang="ts">
  import {
    EditorView,
    createEditorStore,
    starterExtensions,
  } from "@moda-editor/svelte";

  const editorStore = createEditorStore({
    doc: myDoc,
    extensions: starterExtensions(),
  });
</script>

<EditorView editor={editorStore.editor} />
```

`$editorStore` → 불변 `EditorState`, `editorStore.editor` → commands/dispatch.
`EditorNode.svelte`는 RenderNode 트리를 번역하는 자기 참조 컴포넌트다.

### 커스텀 노드 뷰

```svelte
<EditorView editor={editorStore.editor} nodeViews={{ codeBlock: CodeBlockView }} />
```

컴포넌트는 `node`/`editor`/`path` props와 `content` 스니펫을 받는다.
계약과 헬퍼는 [커스텀 노드 뷰](../features/node-views.md) 참고.
