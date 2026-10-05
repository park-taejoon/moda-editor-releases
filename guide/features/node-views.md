# 커스텀 노드 뷰 (nodeViews)

스키마 `toDOM`으로 표현할 수 없는 노드 — 헤더 바·버튼·셀렉트 같은
크롬이 붙는 블록 — 는 프레임워크 컴포넌트로 주입한다. 코어가
`NodeViewProps` 계약만 정하고 각 어댑터가 자기 프레임워크의
컴포넌트로 감싼다. 콜아웃과 코드블록 카드가 내장 예시다.

```tsx
<EditorView editor={editor} nodeViews={{ codeBlock: CodeBlockView }} />
```

`RenderNode.source.type === "codeBlock"`인 노드마다 컴포넌트가
마운트된다. nodeViews 미등록 시 스키마 `toDOM`이 폴백으로 렌더된다
(코드블록은 코어 `.me-codeui` 오버레이가 대신 뜬다).

## NodeViewProps

| 필드      | 타입                     | 설명                                                        |
| --------- | ------------------------ | ----------------------------------------------------------- |
| `node`    | `EditorNode`             | 원본 AST 노드 — `attrs`(언어·alt 등)는 여기서 읽는다         |
| `editor`  | `Editor`                 | 커맨드 실행은 `editor.commands.*`                             |
| `path`    | `readonly number[]`      | doc 루트에서의 자식 인덱스 경로 — 경로 기반 커맨드의 인자     |
| `content` | 프레임워크별 슬롯/스니펫 | 비-atom 노드의 편집 아울렛 — **반드시 렌더해야 한다**         |

`path`가 있으면 브라우저 캐럿과 무관하게 "이 블록"을 조작할 수 있다 —
셀렉트/버튼을 만지는 동안 캐럿이 크롬으로 이동해도 정확한 블록이
바뀐다. 경로 기반 커맨드가 없으면 `path`의 블록을 선택하는
트랜잭션을 먼저 디스패치하고 일반 커맨드를 쓴다.

## DOM 계약

```
atom 노드 — 편집 불가 섬
  .me-node-view[data-me-path][contenteditable="false"]
    └ 컴포넌트가 전부 렌더 (안쪽 좌표는 부모 경계로 환원)

콘텐츠 블록 — 편집 가능 아울렛
  .me-node-view[contenteditable="false"]
    └ 컴포넌트 크롬
        └ .me-node-content[data-me-path][contenteditable="true"]
            └ AST 자식 엘리먼트 1:1 — 여기서 타이핑/분할이 일어난다
```

- 아울렛 엘리먼트는 어댑터가 렌더한다 — 노드의 `toDOM` 태그/속성을
  따르므로 codeBlock이면 `<pre data-language class="me-node-content">`.
  컴포넌트는 `content`를 자기 크롬 안에 **위치만** 정한다. 렌더하지
  않으면 자식 블록이 화면에 나오지 않는다.
- 크롬의 클래스는 `me-` 접두어를 쓴다 — `.me-codeview`(카드),
  `.me-codehead`(헤더 바), `.me-lang`(언어 셀렉트), `.me-codebtn`이
  내장 스타일과 테스트 계약이다.
- 크롬을 클릭해도 에디터 캐럿이 빠지지 않게 헤더 컨테이너의
  `mousedown`은 `preventDefault`한다 — 단 `<select>`는 제외해야
  드롭다운이 열린다.

## 제공 함수

| 함수 / 커맨드                                | 설명                                                        |
| -------------------------------------------- | ----------------------------------------------------------- |
| `blockText(node)`                            | 노드의 텍스트 전체 (복사 버튼용)                              |
| `codeLangOptions(current)`                   | 언어 셀렉트 옵션 목록 — 현재 값이 목록에 없으면 동적 추가     |
| `editor.commands["setCodeLanguageAt"](path, l)` | path 블록의 `language` attr 변경                              |
| `editor.commands["formatCodeAt"](path)`         | path 블록 서식 정리 — JSON pretty-print·언어별 재들여쓰기·공백 정리 |
| `renderNodeHTML(rn)`                            | RenderNode → HTML 문자열 (data-me-path 유지) — 아래 IME 규칙용 |

유틸 함수(`blockText`, `codeLangOptions`, `codeTextAt` …)는 각 어댑터
패키지에서 그대로 re-export한다 — 예를 들어 `import { blockText } from
"@moda-editor/vue"`. `renderNodeHTML`은 어댑터-레벨 헬퍼라
`@moda-editor/core`에서만 export한다 — 노드 뷰 컴포넌트 작성에는
필요 없고, 새 어댑터/리프 직렬화를 만들 때 쓴다.

## IME와 리프 홀더 규칙

**텍스트 홀더(리프) 안에 프레임워크 앵커 노드를 렌더하지 않는다.**
`{#each}`/`v-for`/keyed map이 심는 빈 텍스트·주석 노드가 편집 영역에
섞이면 Chromium이 IME 조합 위치를 잘못 정규화해 한글 자모가 두 번
남는다. 리프 자식(`.me-tok` 토큰, 마크 래퍼, 빈 줄의 bogus `<br>`)은
`renderNodeHTML()`로 직렬화해 한 번에 쓴다 — Svelte 어댑터가 이
방식이다. 아울렛 자체에 `{#each}`를 쓰는 것은 괜찮다 — 앵커가
요소 사이에 있으면 문자 오프셋 매핑에 영향이 없다.

## 예시 — 코드블록 카드 (React)

```tsx
import { blockText, codeLangOptions } from "@moda-editor/react";
import type { NodeViewProps } from "@moda-editor/react";
import type { ReactNode } from "react";

function CodeBlockView({ node, editor, path, content }: NodeViewProps) {
  const lang = String(node.attrs?.["language"] ?? "");
  return (
    <div className="me-codeview">
      <div
        className="me-codehead"
        onMouseDown={(e) => {
          if (!(e.target instanceof HTMLSelectElement)) e.preventDefault();
        }}
      >
        <select
          className="me-lang me-codelang"
          value={lang}
          onChange={(e) =>
            editor.commands["setCodeLanguageAt"]?.(path, e.target.value)
          }
        >
          {codeLangOptions(lang).map((o) => (
            <option key={o.value} value={o.value}>{o.label}</option>
          ))}
        </select>
        <button
          type="button"
          className="me-codebtn"
          onClick={() => editor.commands["formatCodeAt"]?.(path)}
        >
          정리
        </button>
        <button
          type="button"
          className="me-codebtn"
          onClick={() => navigator.clipboard.writeText(blockText(node))}
        >
          ⧉
        </button>
      </div>
      {content as ReactNode}
    </div>
  );
}
```

`content`는 `unknown`으로 선언돼 있다(프레임워크별 형태가 달라서) —
React에서는 `as ReactNode`로 캐스트해 렌더한다. `commands`는
`Record<string, Command>`라 어댑터가 타입-세이프하게 감싸지 않은
커스텀 커맨드는 `commands["이름"]?.(args)`로 호출한다.

플랫폼별 형태 — Vue는 `content` 슬롯, Svelte는 `content` 스니펫,
Vanilla는 `content` HTMLElement를 받는 팩토리 `({node, editor, path,
content}) => HTMLElement`를 등록한다.

## 플랫폼 사용법

각 플랫폼 가이드 참고: [React](../platforms/react.md) ·
[Vue 3](../platforms/vue.md) · [Vue 2](../platforms/vue2.md) ·
[Svelte](../platforms/svelte.md) · [Vanilla](../platforms/vanilla.md)
