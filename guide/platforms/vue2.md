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

`doc`이 반응형이면 변경 시 `editor.setDoc`으로 문서를 통째 교체한다
(Vue 3과 같은 계약 — 같은 내용 재동기화는 건너뛰고 undo에 쌓지
않는다). 비반응형 options는 생성 시 한 번만 읽는다 — 런타임 교체는
`editor.setDoc(doc, { addToHistory: false })`를 직접 호출한다.

### 커스텀 노드 뷰

```vue
<EditorView :editor="editor" :node-views="{ codeBlock: CodeBlockView }" />
```

Vue 3과 같은 계약 — `node`/`editor`/`path` props + `content` 슬롯.
render 함수 컴포넌트에서는 `this.$scopedSlots.content`로 아울렛을
꺼낸다. 계약과 헬퍼는 [커스텀 노드 뷰](../features/node-views.md) 참고.

## 옵션과 런타임 API

`EditorOptions`와 `editor` 인스턴스 API 요약 — 상세는
[architecture.md](../architecture.md)의 표를 본다.

| 옵션               | 용도                                                                                      |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `doc`              | 초기 문서 (JSON AST)                                                                      |
| `initialHtml`      | 초기 문서를 HTML로 (doc과 함께면 무시 + devWarn)                                          |
| `extensions`       | 기능 번들 — `starterExtensions()` + 커스텀                                                |
| `schema`           | 확장 조립 대신 스키마 직접 지정                                                           |
| `editable`         | `false`면 읽기 전용 — 런타임 전환은 `setEditable()`                                       |
| `ariaLabel`        | `.me-editor`의 `aria-label`                                                               |
| `placeholder`      | 빈 문서 플레이스홀더 (`data-me-placeholder`)                                              |
| `autofocus`        | 마운트 직후 에디터에 포커스                                                               |
| `initialSelection` | 초기 셀렉션 (기본: 문서 시작 캐럿)                                                        |
| `onImageUpload`    | 이미지 File → src 훅                                                                      |
| `events`           | `create`/`update`/`selectionUpdate`/`transaction`/`focus`/`blur`/`paste`/`destroy` 리스너 |

| 메서드                                      | 용도                                         |
| ------------------------------------------- | -------------------------------------------- |
| `setDoc(doc, {addToHistory?})`              | 문서 전체 교체 (동일 문서는 no-op)           |
| `setHtml(html)`                             | HTML로 문서 교체                             |
| `getHTML()` / `getText()` / `getMarkdown()` | 직렬화                                       |
| `isEmpty()`                                 | 빈 문서 판정                                 |
| `setEditable(v)`                            | 편집/읽기 전용 런타임 전환                   |
| `commands.<name>(...)`                      | 커맨드 호출 — 스타터명·인자는 타입으로 안내  |
| `dispatch(tr)`                              | 트랜잭션 적용 (빈 tr은 무시)                 |
| `destroy()`                                 | 수명 종료 — `"destroy"` 이벤트로 바인딩 정리 |

**Vue 2.7 특이점**: `options.doc`이 ref면 변경 시
`setDoc`으로 교체를 반영한다. 컴포넌트 스코프 안에서 쓰면 언마운트에
`editor.destroy()`가 호출된다. `EditorView`는 원시 SFC로 배포되므로
컴파일러 없이도 쓸 수 있다.
