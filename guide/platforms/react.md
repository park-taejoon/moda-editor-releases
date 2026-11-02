# React

```tsx
import { WidgetView, useWidget } from "@moda-editor/react";
import "@moda-editor/react/styles.css";

function App() {
  const { widget, snapshot } = useWidget({ items, getItemText });
  return <WidgetView widget={widget} getItemText={getItemText} />;
}
```

| export               | 역할                                             |
| -------------------- | ------------------------------------------------ |
| `useWidget(options)` | 코어 생성 + 스냅샷 구독 (`useSyncExternalStore`) |
| `WidgetView`         | 스냅샷을 DOM 계약으로 렌더하는 컴포넌트          |

이미 만든 `WidgetCore`를 공유할 때는 `WidgetView`에 `widget`만 넘기면 된다 —
스냅샷 구독은 컴포넌트가 알아서 한다. 검색 입력의 문구는
`searchPlaceholder` prop으로 바꾼다 (기본 `"검색…"`).

## 에디터

```tsx
import { EditorView, starterExtensions, useEditor } from "@moda-editor/react";

function App() {
  const { editor, state } = useEditor({
    doc: myDoc, // DocNode JSON
    extensions: starterExtensions(), // 스타터 스키마/커맨드
  });
  return <EditorView editor={editor} />;
}
```

`useEditor`는 `createEditor`를 감싼다 — 반환값의 `state`는 불변
`EditorState`라 `===` 비교로 렌더 필요 여부가 결정된다.
`editor.commands.toggleBold()` 등을 툴바 버튼에 연결하면 된다.

`doc` prop은 반응형이다 — 마운트 후 값이 바뀌면 `editor.setDoc`으로
문서를 통째 교체한다 (어드민 폼의 편집 대상 전환처럼 `key` 리마운트
없이 동작한다). 스키마 보정 후 의미 비교로 같은 내용의 재동기화는
건너뛰고, 이 반영분은 undo 스택에 쌓지 않는다. `doc` prop이 아니라
직접 교체할 때는 `editor.setDoc(doc, { addToHistory: false })`를
호출한다 — 기본은 undo 한 번으로 복원되는 일반 편집이다.

### 커스텀 노드 뷰

```tsx
<EditorView editor={editor} nodeViews={{ codeBlock: CodeBlockView }} />
```

`CodeBlockView`는 `NodeViewProps{node, editor, path, content}`를
받는 컴포넌트 — `content`(ReactNode)를 자기 크롬 안에 렌더한다.
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

**React 특이점**: `doc` prop은 반응형 — 마운트 후 값이
바뀌면 `editor.setDoc`으로 교체를 반영한다(undo에 안 쌓임). 컴포넌트
언마운트에 `editor.destroy()`가 자동 호출된다.
