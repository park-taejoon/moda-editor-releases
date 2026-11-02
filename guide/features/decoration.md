# 데코레이션 (Plugin.decorations)

doc을 바꾸지 않는 표시 계층 — PM `Decoration` 대응.
스캐너·lint 밑줄·읽기 전용 주석처럼 "문서 내용은 아닌데 눈에 보여야
하는" 것들의 공식 경로다.

## 계약

```ts
import type { Decoration, Plugin } from "@moda-editor/core";

const lintPlugin: Plugin = {
  key: "lint",
  init: () => ({ issues: [] }),
  // pluginState가 바뀌면 재평가된다 — EditorState.decorations()가
  // 모든 플러그인의 반환을 합쳐 렌더 계층에 넘긴다
  decorations(state) {
    const issues = (state.pluginState["lint"] as { issues: Range[] }).issues;
    return issues.map((r) => ({
      type: "inline",
      from: r.anchor,
      to: r.focus,
      attrs: { class: "me-deco me-deco-lint" },
    }));
  },
};
```

```ts
type Decoration =
  | { type: "inline"; from: Point; to: Point; attrs: DecorationAttrs }
  | { type: "node"; path: readonly number[]; attrs: DecorationAttrs };
```

## 동작 모델

- **inline** — 범위 안 텍스트를 `<span class="me-deco …">`로 감싼다.
  마크 경계에서 분할해 마크·스타일과 공존한다 — DOM↔Point 매핑은
  데코 span이 "투명 래퍼"라 깨지지 않는다.
- **node** — 블록 최외곽 요소에 class/attrs를 추가한다. 마크업이
  아니라 블록 자체의 시각 상태(현재 항목, 드래그 중, 경고)에 쓴다.
- 데코는 `renderChildren`의 인자로 들어가 모든 렌더러가 같은
  트리를 소비한다 — vanilla/4어댑터 모두 deco 키를 캐시 조건에
  포함해 pluginState가 바뀌면 자동 재렌더된다.
- 범위를 벗어나는 데코는 조용히 버린다 — 스키마 경계 바깥 좌표는
  렌더 대상이 아니다.

## DOM 계약

```text
.me-deco           — 인라인 데코 span (기본은 투명)
.me-deco-<name>    — 기능이 주는 모양 (예: 데모의 .me-deco-todo)
```

데모 앱은 `TODO_DECO_EXT`로 할일 항목의 `TODO` 마커에 밑줄 데코를
단다 — `apps/dev-shared/src/data.ts` 참고.
