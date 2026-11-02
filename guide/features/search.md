# 검색/치환 (searchFeature)

에디터 내 find/replace — PM `prosemirror-search`·GDocs Ctrl+F 대응.
`starterFeatures()`에 포함돼 스타터 프리셋이면 바로 쓴다.

```ts
editor.commands["setSearch"]!("검색어"); // 매치 수집 + 하이라이트 (셀렉션 불변)
editor.commands["setSearch"]!("검색어", true); // 대소문자 구분
editor.commands["findNext"]!(); // 다음 매치로 이동 (순환)
editor.commands["findPrev"]!(); // 이전 매치로 이동
editor.commands["replaceNext"]!("치환어"); // 현재 매치 치환 → 다음으로
editor.commands["replaceAll"]!("치환어"); // 전체 치환 (한 트랜잭션)
editor.commands["clearSearch"]!(); // 해제 + 하이라이트 제거
```

## 동작 모델

- 검색 상태는 `pluginState["search"]`에 유지된다 — `SearchState`
  `{ query, caseSensitive, matches, active }`. `searchStateOf(state)`로
  읽어 UI(결과 카운터, n/m 표시)를 만든다.
- `setSearch`는 셀렉션을 움직이지 않는다 — 입력창 타이핑 중 캐럿이
  튀지 않도록 매치 수집과 `active`(캐럿 이후 첫 매치 — 카운터와 첫
  내비게이션 도착점)만 갱신한다. `findNext`/`findPrev`만 매치로 이동해
  선택한다 — 셀렉션이 이미 active 매치 위면 순환하고, 아니면
  캐럿 기준 이후/이전 매치로 간다.
- 포커스가 에디터 밖(찾기 바 등)에 있는 동안 렌더 커밋은 DOM 셀렉션을
  쓰지 않는다 — Blink는 contenteditable 안에 셀렉션을 쓰면 그 요소를
  focus하기 때문이다. 이 동안 현재 매치는 `me-find-active` 하이라이트와
  스크롤로 표시되고, 포커스가 돌아오면(focusin) 캐럿이 마지막 위치로
  동기화된다.
- 매치 수집은 `collectMatches(doc, query, caseSensitive)` 순수 함수 —
  텍스트 노드 단위로만 찾는다. 노드를 넘는 매치는 잡지 않는다.
- `replaceNext`는 현재 매치를 지우고 치환어를 넣은 뒤 다음 매치로
  이동한다. `replaceAll`은 뒤쪽 매치부터 한 트랜잭션으로 치환해
  undo 한 번에 되돌아간다.

## 하이라이트

DOM을 건드리지 않고 CSS Custom Highlight API로 Range를 등록한다 —
`me-bracket`(괄호 매칭)과 같은 계약이라 매핑·리렌더에 무영향이다.

| 채널                          | CSS 변수              | 용도           |
| ----------------------------- | --------------------- | -------------- |
| `::highlight(me-find)`        | `--me-find-bg`        | 모든 매치 배경 |
| `::highlight(me-find-active)` | `--me-find-active-bg` | 현재 매치 강조 |

Highlight API가 없는 환경에서는 등록을 조용히 건너뛴다 — 검색 자체는
계속 동작한다.

## UI 연결 예시

데모 앱은 ⋯ 오버플로 메뉴의 `find`(Chrome식 찾기 바 — 에디터
프레임 우상단 안쪽, 입력마다 `setSearch`, `n/total` 카운터, ↑/↓
순환, 바꾸기 줄 치환/전체, ✕ 닫기) 커맨드로 연결한다 —
`apps/dev-shared/src/data.ts`의 `runTool`/`applyFind`/`findNav`/
`findReplace` 참고.
팝오버를 쓰는 이유는 네이티브 `prompt`가 cross-origin iframe(통합
데모 셸)에서 차단되기 때문이다. 검색창 UI를 직접 만들 때는
input change → `setSearch`, 엔터/Shift-엔터 → `findNext`/`findPrev`
매핑이 표준 패턴이다.
