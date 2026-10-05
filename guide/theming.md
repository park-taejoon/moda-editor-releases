# 테마

공통 스타일은 `packages/core/src/styles.css`에 있고 어댑터는
`@import "@moda-editor/core/styles.css"`로 재사용한다.

색상은 `--me-*` CSS 변수로 노출한다 — 새 색상을 추가할 때는 변수로
만들고 이 표에 추가한다.

| 변수                    | 기본값    | 용도                          |
| ----------------------- | --------- | ----------------------------- |
| `--me-border-color`    | `#e2e8f0` | 경계선                        |
| `--me-text-color`      | `#1e293b` | 본문 텍스트                    |
| `--me-editor-bg`       | `#ffffff` | 에디터 배경                    |
| `--me-editor-focus`    | `#3b82f6` | 포커스 테두리 + 캐럿색          |
| `--me-focus-ring`      | —         | `.me-frame` 포커스 링          |
| `--me-frame-shadow`    | —         | `.me-frame` 카드 그림자        |
| `--me-toolbar-bg`      | `#f8fafc` | 툴바/상태바 배경               |
| `--me-tool-hover-bg`   | `#e2e8f0` | 툴바 버튼 hover                |
| `--me-tool-active-bg`  | `#dbeafe` | 툴바 버튼 활성 배경            |
| `--me-tool-active-color` | `#1d4ed8` | 툴바 버튼 활성 글자            |
| `--me-status-color`    | `#64748b` | 상태바 글자                    |
| `--me-code-bg`         | `#f1f5f9` | `code` 마크 배경               |
| `--me-code-border`     | `#e2e8f0` | `code` 마크 테두리             |
| `--me-callout-bg`      | `#eff6ff` | callout 노드 뷰 배경           |
| `--me-callout-border`  | `#bfdbfe` | callout 노드 뷰 테두리          |
| `--me-callout-accent`  | `#3b82f6` | callout 왼쪽 액센트 바 + 아이콘  |
| `--me-tok-keyword`     | `#7c3aed` | 코드블록 토큰 — 키워드          |
| `--me-tok-string`      | `#0e7a3d` | 코드블록 토큰 — 문자열          |
| `--me-tok-comment`     | `#94a3b8` | 코드블록 토큰 — 주석            |
| `--me-tok-number`      | `#b45309` | 코드블록 토큰 — 숫자            |
| `--me-tok-literal`     | `#b45309` | 코드블록 토큰 — 불리언/null     |
| `--me-tok-function`    | `#1d4ed8` | 코드블록 토큰 — 함수 호출       |
| `--me-tok-tag`         | `#b91c1c` | 코드블록 토큰 — 태그/선택자      |
| `--me-tok-attr`        | `#7c3aed` | 코드블록 토큰 — 속성            |
| `--me-code-active-bg`  | `rgba(59,130,246,.08)` | 코드블록 활성 줄 배경 (`.me-cl.is-active`) |
| `--me-gutter-bg`       | `rgba(15,23,42,.035)` | 에디터 라인번호 거터 컬럼 배경   |
| `--me-gutter-color`    | `#94a3b8` | 거터 라인번호 글자                    |
| `--me-gutter-border`   | `var(--me-border-color)` | 거터 구분선                |
| `--me-line-active-bg`  | `rgba(59,130,246,.07)` | 문서 활성 라인 배경 (`.me-line-active`) |
| `--me-line-active-num` | `#3b82f6` | 활성 라인의 거터 번호 글자            |
| `--me-sel-bg`          | `rgba(59,130,246,.22)` | `::selection` 배경 (전역 + 코드블록) |
| `--me-print-margin`    | `#e2e8f0` | 80열 print margin 세로선 (에디터 + 코드블록) |
| `--me-font-mono`       | `ui-monospace,…` | 코드 에디터 폰트 스택 (`.me-root`/`.me-frame` 스코프) |
| `--me-invisible`       | `rgba(148,163,184,.55)` | `[data-me-invisibles]` 공백 점 (`.me-ws`) |
| `--me-bracket-bg`      | `rgba(59,130,246,.28)` | `::highlight(me-bracket)` 괄호 쌍 배경 |

`--me-tok-*` 변수는 `NodeSpec.codeHighlighter`의 토큰 스팬
(`span.me-tok.me-tok-<kind>`) 색이다 — 다크 테마에서도 재정의된다.

## Ace 디자인

에디터 서페이스는 Ace 계열 코드 에디터 크롬이다 — 모노스페이스
본문(`--me-font-mono`), 최상위 블록마다 세는 라인번호 거터,
활성 라인(`.me-line-active`), 80열 print margin. 거터 번호는
`aceFeature`가 그리는 `.me-gutter > .me-gnum` 오버레이이고,
거터 배경·구분선은 `.me-editor`의 background gradient다 — 둘 다
contenteditable 텍스트 흐름 밖이라 DOM↔doc 매핑과 Blink 캐럿
줄 계산(Home/End)에 영향이 없다. 코드블록은 자체 `.me-cl` 줄
번호를 쓰므로 문서 번호는 건너뛴다. 상태바는 블록/마크(좌) +
Ln·Col·블록 수(우)의 VSCode 관례 2분할이다.

## 다크 테마

`.me-frame`(또는 그 조상)에 `data-me-theme="dark"`를 달면 위 변수들이
Monokai 계열 팔레트로 오버라이드된다. 커스텀 테마도 같은 방식으로
추가할 수 있다 — `[data-me-theme="…"]` 셀렉터 아래에서 변수만
재정의하면 된다.

주의: `EditorView`/`WidgetView`가 렌더하는 `.me-root` 래퍼는 팔레트
기본값을 스스로 갖고 있으므로, 조상의 테마 변수를 중간에서 차단한다.
새 테마를 만들 때는 `[data-me-theme="…"] .me-root`도 함께
오버라이드해야 한다 (다크 테마 블록 참고).

DOM 클래스는 `me-` 접두어를 쓴다 (`me-root`/`me-search`/`me-list`/
`me-item`/`me-count`/`me-editor`/`me-node-view`/`me-callout`/
`me-frame`/`me-toolbar`/`me-tool`/`me-ic`/`me-sep`/`me-statusbar`/
`me-blockfmt`/`me-overflow`/`me-styles`/`me-menu`).
새 요소도 같은 접두어 + conformance 계약 갱신.
