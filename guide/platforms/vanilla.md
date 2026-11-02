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

options은 마운트 시 한 번만 읽는다 — 런타임 문서 교체(편집 대상 전환)는
`mounted.editor.setDoc(doc)`을 호출한다. `{ addToHistory: false }`를
넘기면 undo 스택에 쌓지 않는다.

### 커스텀 노드 뷰

```ts
mountEditor(el, {
  doc,
  extensions,
  nodeViews: { codeBlock: (props: NodeViewProps) => HTMLElement },
});
```

팩토리가 `.me-node-view` 안쪽 엘리먼트를 반환한다 — `props.content`
HTMLElement를 자기 크롬 안에 append해 편집 아울렛을 배치한다.
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

**Vanilla 특이점**: `mountEditor`가 `{ editor, rootEl, binding,
destroy() }`를 반환한다 — `binding`은 `bindEditorInput`의 공개 계약
(셀렉션 동기화·`dirtyPaths`)로, 재조정 시 mountEditor가 더티 표시를
소비한다. `destroy()`는 바인딩 해제 + (내부 생성이면)
`editor.destroy()`까지 한다. `options.editor`로 외부 인스턴스를 넘기면
에디터는 해제하지 않는다. 옵션 변경은 마운트 후 반영되지 않는다 —
`setDoc`/`setEditable` 등 메서드로 조정한다.
