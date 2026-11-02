# 테마

공통 스타일은 `packages/core/src/styles.css`에 있고 어댑터는
`@import "@moda-editor/core/styles.css"`로 재사용한다.

색상은 `--me-*` CSS 변수로 노출한다 — 새 색상을 추가할 때는 변수로
만들고 이 표에 추가한다.

| 변수                     | 기본값                          | 용도                                                                                |
| ------------------------ | ------------------------------- | ----------------------------------------------------------------------------------- |
| `--me-border-color`      | `#e2e8f0`                       | 경계선                                                                              |
| `--me-text-color`        | `#1e293b`                       | 본문 텍스트                                                                         |
| `--me-editor-bg`         | `#ffffff`                       | 에디터 배경                                                                         |
| `--me-editor-focus`      | `#3b82f6`                       | 포커스 테두리 + 캐럿색                                                              |
| `--me-frame-bg`          | `#ffffff`                       | `.me-frame` 카드 배경                                                               |
| `--me-frame-shadow`      | `0 1px 2px rgba(15,23,42,0.06)` | `.me-frame` 카드 그림자                                                             |
| `--me-pop-shadow`        | `0 8px 24px rgba(…)`            | 뜨는 UI 공통 그림자 — `.me-pop`/`.me-bubble`/`.me-linkcard`/`.me-menu`/이미지·표 바 |
| `--me-toolbar-bg`        | `#f8fafc`                       | 툴바/상태바 배경                                                                    |
| `--me-tool-hover-bg`     | `#e2e8f0`                       | 툴바 버튼 hover                                                                     |
| `--me-tool-active-bg`    | `#dbeafe`                       | 툴바 버튼 활성 배경                                                                 |
| `--me-tool-active-color` | `#1d4ed8`                       | 툴바 버튼 활성 글자                                                                 |
| `--me-status-color`      | `#64748b`                       | 상태바 글자                                                                         |
| `--me-code-bg`           | `#f1f5f9`                       | `code` 마크 배경                                                                    |
| `--me-code-border`       | `#e2e8f0`                       | 코드블록 크롬 테두리 (인라인 `code` 마크는 테두리 없이 `--me-code-bg`만 쓴다)       |
| `--me-callout-bg`        | `#eff6ff`                       | callout 노드 뷰 배경                                                                |
| `--me-callout-border`    | `#bfdbfe`                       | callout 노드 뷰 테두리                                                              |
| `--me-callout-accent`    | `#3b82f6`                       | callout 왼쪽 액센트 바 + 아이콘                                                     |
| `--me-tok-keyword`       | `#cf222e`                       | 코드블록 토큰 — 키워드                                                              |
| `--me-tok-string`        | `#0a3069`                       | 코드블록 토큰 — 문자열                                                              |
| `--me-tok-comment`       | `#6e7781`                       | 코드블록 토큰 — 주석                                                                |
| `--me-tok-number`        | `#0550ae`                       | 코드블록 토큰 — 숫자                                                                |
| `--me-tok-literal`       | `#0550ae`                       | 코드블록 토큰 — 불리언/null                                                         |
| `--me-tok-function`      | `#8250df`                       | 코드블록 토큰 — 함수 호출                                                           |
| `--me-tok-tag`           | `#116329`                       | 코드블록 토큰 — 태그/선택자                                                         |
| `--me-tok-attr`          | `#953800`                       | 코드블록 토큰 — 속성                                                                |
| `--me-code-active-bg`    | `rgba(59,130,246,.08)`          | 코드블록 활성 줄 배경 (`.me-cl.is-active`)                                          |
| `--me-gutter-bg`         | `rgba(15,23,42,.035)`           | 에디터 라인번호 거터 컬럼 배경                                                      |
| `--me-gutter-color`      | `#94a3b8`                       | 거터 라인번호 글자                                                                  |
| `--me-gutter-border`     | `var(--me-border-color)`        | 거터 구분선                                                                         |
| `--me-line-active-num`   | `#3b82f6`                       | 활성 라인의 거터 번호 글자                                                          |
| `--me-node-sel-color`    | `#3b82f6`                       | atom 노드 셀렉션 링(`.is-selected`) + 이미지 크기 핸들                              |
| `--me-sel-bg`            | `rgba(59,130,246,.22)`          | `::selection` 배경 (전역 + 코드블록)                                                |
| `--me-print-margin`      | `#e2e8f0`                       | 80열 print margin 세로선 (에디터 + 코드블록)                                        |
| `--me-font-mono`         | `ui-monospace,…`                | 코드 에디터 폰트 스택 (`.me-root`/`.me-frame` 스코프)                               |
| `--me-invisible`         | `rgba(148,163,184,.55)`         | `[data-me-invisibles]` 공백 점 (`.me-ws`)                                           |
| `--me-bracket-bg`        | `rgba(59,130,246,.28)`          | `::highlight(me-bracket)` 괄호 쌍 배경                                              |
| `--me-find-bg`           | `rgba(250,204,21,.35)`          | `::highlight(me-find)` 검색 매치 배경                                               |
| `--me-find-active-bg`    | `rgba(249,115,22,.45)`          | `::highlight(me-find-active)` 현재 매치 배경 + 외곽선                               |
| `--me-mention-bg`        | `rgba(59,130,246,.12)`          | `.me-mention` 인라인 atom 칩 배경                                                   |
| `--me-mention-color`     | `#2563eb`                       | `.me-mention` 칩 라벨 글자                                                          |
| `--me-accent`            | `#2563eb`                       | 범용 액센트 — `.me-tool` 활성 등                                                    |
| `--me-btn-bg`            | `#f1f5f9`                       | 크롬 바/오버레이 버튼 배경                                                          |
| `--me-btn-hover-bg`      | `#e2e8f0`                       | 버튼 호버 배경                                                                      |
| `--me-checkbox-border`   | `#94a3b8`                       | taskItem 마커 테두리                                                                |
| `--me-checkbox-checked`  | `#2563eb`                       | taskItem 체크 배경                                                                  |
| `--me-link-color`        | `#2563eb`                       | `a` 링크 글자                                                                       |
| `--me-placeholder-color` | `#94a3b8`                       | 플레이스홀더 글자                                                                   |
| `--me-placeholder-text`  | `"내용을 입력하세요…"`          | 플레이스홀더 문구 (content 문자열)                                                  |
| `--me-quote-color`       | `#64748b`                       | blockquote 텍스트                                                                   |
| `--me-text-muted`        | `#64748b`                       | 보조 텍스트 (타임스탐프·캡션 등)                                                    |
| `--me-danger`            | `#dc2626`                       | 위험 액션 호버 (이미지 삭제·표 행/열 삭제 버튼)                                     |
| `--me-on-accent`         | `#ffffff`                       | 액센트 배경 위 텍스트 (확인 버튼·체크 표시·원격 커서 라벨)                          |
| `--me-font-body`         | `var(--me-font-mono)`           | 본문 폰트 — 기본은 코드 에디터 폰트, 제품 산문 폰트로 교체 가능                     |
| `--me-gutter-w`          | `3.9em`                         | 에디터 라인번호 거터 폭 (패딩·그라디언트·드롭 라인이 공유)                          |
| `--me-code-gutter-w`     | `3.4em`                         | 코드블록 줄번호 거터 폭                                                             |
| `--me-editor-min-height` | `160px`                         | `.me-editor` 최소 높이                                                              |
| `--me-radius-sm`         | `4px`                           | 작은 컨트롤 (버튼·입력·칩)                                                          |
| `--me-radius-md`         | `6px`                           | 카드/패널 모서리                                                                    |
| `--me-radius-pop`        | `8px`                           | 뜨는 UI 모서리 (팝오버·버블·메뉴)                                                   |
| `--me-radius-frame`      | `10px`                          | `.me-frame`/`.me-editor` 카드 모서리                                                |
| `--me-z-toolbar`         | `20`                            | 툴바 겹침 순서                                                                      |
| `--me-z-carets`          | `35`                            | 원격 커서 레이어                                                                    |
| `--me-z-overlay`         | `40`                            | 편집 오버레이 (이미지·표·코드 바)                                                   |
| `--me-z-float`           | `50`                            | 선택 버블 메뉴                                                                      |
| `--me-z-pop`             | `60`                            | 팝오버·드롭다운 메뉴·링크 카드                                                      |

`prefers-color-scheme: dark`에서 명시 테마(`data-me-theme`)가 없는
기본 팔레트는 자동으로 다크 팔레트로 전환된다 — `applyTheme`을 쓰면
인라인 변수가 미디어 쿼리보다 우선해 자동 전환은 적용되지 않는다.
| `--me-remote-color` | `#f59e0b` | 원격 커서 색 (`.me-remote-caret`/`-label`/`.me-remote-sel`) — 바인딩이 피어별로 인라인 지정 |

`--me-tok-*` 변수는 `NodeSpec.codeHighlighter`의 토큰 스팬
(`span.me-tok.me-tok-<kind>`) 색이다 — 다크 테마에서도 재정의된다.

## Ace 디자인

에디터 서페이스는 Ace 계열 코드 에디터 크롬이다 — 모노스페이스
본문(`--me-font-mono`), 최상위 블록마다 세는 라인번호 거터,
80열 print margin. 활성 라인은 본문 배경이 아니라 거터 번호
강조(`.me-gnum.is-active`)로만 표시한다 — `.me-line-active`
클래스는 캐럿 블록에 계속 달리니 제품에서 원하면 배경을 칠할 수
있다. 거터 번호는
`aceFeature`가 그리는 `.me-gutter > .me-gnum` 오버레이이고,
거터 배경·구분선은 `.me-editor`의 background gradient다 — 둘 다
contenteditable 텍스트 흐름 밖이라 DOM↔doc 매핑과 Blink 캐럿
줄 계산(Home/End)에 영향이 없다. 코드블록은 자체 `.me-cl` 줄
번호를 쓰므로 문서 번호는 건너뛴다. 상태바는 블록/마크(좌) +
Ln·Col·블록 수(우)의 VSCode 관례 2분할이다.

## 테마 — `ME_THEMES` + `applyTheme`

테마는 플랫한 `{ "--me-*": 값 }` 변수 맵이다 — CSS를 고치지 않고
런타임에 적용한다. 코어가 내장 프리셋을 제공한다:

| id       | 라벨      | 비고                        |
| -------- | --------- | --------------------------- |
| `simple` | 심플      | 무채색 미니멀               |
| `modern` | 모던      | 인디고 액센트 라이트        |
| `dark`   | 다크      | Ace Monokai 계열            |
| `neon`   | 네온 다크 | 검정 + 사이언/마젠타 발광   |
| `purple` | 퍼플      | 라이트 라벤더               |
| `grape`  | 퍼플 다크 | 딥 바이올렛 + 라벤더 액센트 |

```ts
import { applyTheme, themeById, ME_THEMES } from "@moda-editor/core";

// 프레임(또는 그 안의 아무 요소) 하나면 된다 — 허브 3곳에 변수를 박는다
applyTheme(document.querySelector(".me-frame")!, themeById("neon"));
applyTheme(frame, null); // 기본으로 되돌리기 — 이전 변수 흔적을 지운다
```

`applyTheme`은 `.me-frame`(크롬)·`.me-root`·`.me-editor`(본문) 허브에
인라인 변수를 박는다 — 리프 요소는 변수를 자체 정의하지 않고 `var()`로
소비하므로 상속으로 전부 도달한다. 프레임에는 `data-me-theme="<id>"`가
표시로 남고, `scheme: "dark"` 테마는 `color-scheme`도 함께 단다
(스크롤바·폼 컨트롤이 어두워진다).

### 커스텀 테마

객체 리터럴 하나면 된다 — 정의하지 않은 변수는 기본값이 유지된다:

```ts
import { applyTheme, type MeTheme } from "@moda-editor/core";

const myTheme: MeTheme = {
  id: "brand",
  label: "브랜드",
  vars: {
    "--me-editor-bg": "#fffdf5",
    "--me-editor-focus": "#e11d48",
    "--me-tool-active-color": "#e11d48",
    "--me-link-color": "#e11d48",
    "--me-node-sel-color": "#e11d48",
  },
};
applyTheme(frame, myTheme); // 프리셋과 같은 경로 — 즉시 적용·교체 가능
```

CSS만으로 테마를 고정하고 싶다면 같은 원리로 변수만 덮으면 된다 —
허브가 `.me-frame`/`.me-root`/`.me-editor` 셋이라 셋 다 오버라이드한다:

```css
.my-theme .me-frame, /* 또는 .me-frame.my-theme */
.my-theme .me-root,
.my-theme .me-editor {
  --me-editor-bg: #fffdf5;
  /* … */
}
```

DOM 클래스는 `me-` 접두어를 쓴다 (`me-root`/`me-search`/`me-list`/
`me-item`/`me-count`/`me-editor`/`me-node-view`/`me-callout`/
`me-frame`/`me-toolbar`/`me-tool`/`me-ic`/`me-sep`/`me-statusbar`/
`me-blockfmt`/`me-overflow`/`me-styles`/`me-menu`/`me-imgui`/
`me-img-handle`/`me-img-bar`/`me-imgedit`/`me-tableui`/`me-tbl-bar`/
`me-tblbtn`/`me-col-handle`/`me-carets`/`me-remote-caret`/
`me-remote-caret-label`/`me-remote-sel`).
새 요소도 같은 접두어 + conformance 계약 갱신.
