# Vue 3

```vue
<script setup lang="ts">
import { WidgetView, useWidget } from "@moda-editor/vue";
import "@moda-editor/vue/styles.css";

const { widget, state } = useWidget({ items, getItemText });
</script>

<template>
  <WidgetView :widget="widget" :get-item-text="getItemText" />
</template>
```

| export             | 역할                                 |
| ------------------ | ------------------------------------ |
| `useWidget(options)`  | 코어 생성 + 스냅샷 `shallowRef` 구독 |
| `useWidgetState(widget)` | 외부 코어의 스냅샷만 구독            |
| `WidgetView`          | DOM 계약 렌더 SFC                    |

## 에디터

```vue
<script setup lang="ts">
import { EditorView, starterExtensions, useEditor } from "@moda-editor/vue";

const { editor } = useEditor({ doc: myDoc, extensions: starterExtensions() });
</script>

<template>
  <EditorView :editor="editor" />
</template>
```

`useEditorState(editor)`로 외부 에디터의 `EditorState`만 구독할 수도 있다.
