# 검색 가능한 목록 (예시 기능)

템플릿에 포함된 최소 기능 — 코어→어댑터→테스트→문서의 골든 패스를
보여준다. 실제 프로젝트에서는 도메인 기능으로 교체한다.

## 코어 옵션

```ts
new WidgetCore<Item>({
  items: Item[],                       // 전체 항목
  getItemText?: (item) => string,      // 표시/검색 문자열 (기본 String(item))
  events?: { searchChange, itemAdd },  // 이벤트 핸들러
});
```

## API

| 메서드                           | 동작                                                              |
| -------------------------------- | ----------------------------------------------------------------- |
| `setSearch(text)`                | 검색어 설정 → `filteredItems` 갱신 + `searchChange` 발행          |
| `addItem(item)`                  | 항목 추가 + `itemAdd` 발행                                        |
| `getSnapshot()`                  | `{ items, filteredItems, filteredCount, totalCount, searchText }` |
| `subscribe(fn)` / `on(type, fn)` | 변경 통지 / 이벤트 구독                                           |

## DOM 계약

모든 렌더러가 같은 클래스를 만든다:

- `.me-search` — 검색 입력 (`input` 이벤트 → `setSearch`)
- `.me-list > .me-item` — 필터된 항목
- `.me-count` — `"N / M"` 카운터

## 플랫폼 사용법

각 플랫폼 가이드 참고: [React](../platforms/react.md) ·
[Vue 3](../platforms/vue.md) · [Vue 2](../platforms/vue2.md) ·
[Svelte](../platforms/svelte.md) · [Vanilla](../platforms/vanilla.md)
