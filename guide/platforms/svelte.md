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

| export                       | 역할                               |
| ---------------------------- | ---------------------------------- |
| `createWidgetStore(options)` | 코어 생성 + `readable` 스토어 래핑 |
| `toWidgetStore(widget)`      | 외부 코어를 스토어 계약으로 래핑   |
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

options는 생성 시 한 번만 읽는다 — 런타임 문서 교체(편집 대상 전환)는
반응형 블록에서 `editor.setDoc`을 호출한다:

```svelte
<script lang="ts">
  $: if (doc) editorStore.editor.setDoc(doc, { addToHistory: false });
</script>
```

`setDoc`은 같은 내용이면 no-op으로 조용히 돌아오므로 prop 동기화에도
안전하다 — 매번 호출해도 불필요한 dispatch가 발생하지 않는다.

### 커스텀 노드 뷰

```svelte
<EditorView
  editor={editorStore.editor}
  nodeViews={{ codeBlock: CodeBlockView }}
/>
```

컴포넌트는 `node`/`editor`/`path` props와 `content` 스니펫을 받는다.
계약과 헬퍼는 [커스텀 노드 뷰](../features/node-views.md) 참고.

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

**Svelte 특이점**: `options`는 생성 시 한 번만 읽는다
— 런타임 교체는 반응형 블록에서 `store.editor.setDoc(doc,
{ addToHistory: false })`로 한다. 컴포넌트 해제에 `editor.destroy()`가
호출된다(`onDestroy`; 컴포넌트 밖 생성이면 소비자가 호출).
