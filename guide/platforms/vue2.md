# Vue 2.7

Vue 2.7의 내장 composition API(`<script setup>`)를 사용한다.

```vue
<script setup lang="ts">
import { WidgetView, useWidget } from "@moda-editor/vue2";
import "@moda-editor/vue2/styles.css";

const { widget, state } = useWidget({ items, getItemText });
</script>

<template>
  <WidgetView :widget="widget" :get-item-text="getItemText" />
</template>
```

주의:

- 템플릿 이벤트 핸들러는 표현식만 받는다 — 조건부·캐스트는 메서드로
  추출한다 (vite:vue2 제약).
- `v-if` 분기에 같은 컴포넌트를 쓸 때는 분기별 `key`를 명시한다 —
  Vue2는 인스턴스를 재사용해 prop 교체가 무시될 수 있다.

## 에디터

```vue
<script setup lang="ts">
import { EditorView, starterExtensions, useEditor } from "@moda-editor/vue2";

const { editor } = useEditor({ doc: myDoc, extensions: starterExtensions() });
</script>

<template>
  <EditorView :editor="editor" />
</template>
```

`EditorNode`는 SFC 자기참조 대신 render 함수(`EditorNode.ts`)로 구현돼
있다 — vite:vue2의 SFC 자기참조가 불안정하기 때문이다.
