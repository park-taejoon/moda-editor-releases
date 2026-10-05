# 마이그레이션 매핑

다른 라이브러리에서 이 프로젝트로 옮기는 사용자를 위한 개념 매핑 표.
새 기능을 추가할 때마다 이 표에도 행을 추가한다.

| 기존 개념         | 이 프로젝트                                      | 비고               |
| ----------------- | ------------------------------------------------ | ------------------ |
| (예) props → 상태 | `WidgetCore` 옵션 → 스냅샷                          | 코어가 상태를 소유 |
| (예) render prop  | `getItemText`                                    | 표시 문자열 해석   |
| (예) onChange     | `events.searchChange` / `widget.on("searchChange")` | 이벤트 맵          |

## 에디터 (ProseMirror/Tiptap 대비)

| PM/Tiptap 개념            | 이 프로젝트                                   | 비고                          |
| ------------------------- | --------------------------------------------- | ----------------------------- |
| `Schema` nodes/marks      | `Extension.nodes` / `Extension.marks` 스펙    | toDOM/parseDOM 규칙 포함      |
| `Extension` (Tiptap)      | `Feature` (`defineFeature`) + `Extension` 인터페이스 | `features/<name>.ts` 한 파일 = 기능 하나 |
| `toggleMark` 커맨드       | `toggleMark(type)` / `commands.toggleBold` 등 | 스타터 프리셋 제공            |
| `isMarkActive` (Tiptap)   | `isMarkActive(state, type)`                   | 코어 헬퍼 — 툴바 active 판정 |
| `undo`/`redo` (history)   | `commands.undo/redo` + `canUndo`/`canRedo`    | Step 역변환 기반              |
| `Plugin` (state + view)   | `Extension.plugins` `Plugin.apply(prev,next)` | 상태 플러그인만, 뷰 플러그인 없음 |
| `keymap` 플러그인         | `Extension.keymap` (`"Mod-b"` 등)             | input.ts keydown이 순서 매칭  |
| `handleTextInput`/`handleKeyDown` props | `Extension.input` (insertText/insertParagraph/deleteBackward) | true=소비, false=폴스루 — 같은 모델 |
| `markInputRule`/`textblockTypeInputRule` | `Extension.inputRules` + `markRule`/`blockRule` | 같은 tr 안에서 변환 — 하나의 undo 단위 |
| `NodeView` (atom 섬)      | `EditorView nodeViews` prop                 | `atom: true` 노드 — ce=false 섬 |
| `NodeView` + `NodeViewContent` | `nodeViews` + `NodeViewProps.content`/`path` | 비-atom 노드 — `.me-node-content`에 자식 블록 렌더, `path`로 경로 기반 커맨드 |
| `editor.isActive("heading")` | `isBlockActive(state, "heading")`          | 최상위 블록 기준              |
| `bulletList`/`orderedList`/`listItem` | 스타터 노드 + `tr.splitItem`/`liftItem`/`sinkItem` | `-`/`1.` 입력 규칙, Enter/Backspace/Tab 키맵 내장 |
| `liftListItem`/`sinkListItem` 커맨드 | `commands.liftItem`/`sinkItem` | 중첩 아이템은 바깥 리스트로 들어올림(PM 관례) |
| `blockquote`/`codeBlock`/`horizontalRule`/`hardBreak` | 스타터 노드 | `>`/```` ``` ````/`---` 입력 규칙, `Shift-Enter`/`Mod-Enter` 키맵 |
| `link` 마크 + `setLink` | `link` 마크 + `commands.link` | `[텍스트](url)` 입력 규칙 + `Mod-K` 토글 |
| `placeholder` 확장 | CSS-only — `.me-editor > p:only-child:has(br)` `::before` | `--me-placeholder-text` 변수로 문구 교체 |
| 버블 메뉴 확장      | 제품 영역 — `.me-bubble` DOM 계약 + 셀렉션 구독 | 데모 5개 앱에 참조 구현 |
| `setTextAlign` (Tiptap) | `commands.setAlign(align)` — `attrs.align` | paragraph/heading 공용, `text-align` 스타일 |
| `setColor`/`setHighlight`/`setFontSize` | `textColor`/`highlight`/`fontSize` 마크 + `commands.setMarkAttr` | attrs → 인라인 style |
| `superscript`/`subscript` 마크 | 동명 마크 + `commands.toggleSuperscript`/`toggleSubscript` | 상호 배타 토글 |
| `unsetAllMarks` (Tiptap) | `commands.clearFormat` | 범위 전 마크 제거 / collapsed면 storedMarks 초기화 |
| `table`/`tableRow`/`tableCell` | 동명 스타터 노드 + `commands.insertTable`/`nextCell`/`addTableRow`/`addTableColumn`/`deleteTable` | Tab=다음 셀, 끝 셀 Tab=행 추가 |
| `image` 노드 | `image` atom 리프 + `commands.insertImage(src)` | attrs.src/alt/width → `<img>` |
| 이미지 선택/편집 UI (Tiptap/TinyMCE 이미지 툴바) | 클릭 → NodeSelection + `.me-imgui` 오버레이 (코어 내장) | 편집(src/alt)·파일 교체·삭제·드래그 리사이즈 → `updateImage`/`deleteImage` |
| `taskList`/`taskItem` | 동명 스타터 노드 + `commands.toggleTaskItem` | `[ ]`/`[x]` 입력 규칙, `.me-check` 마커 클릭 토글 |
| 링크 다이얼로그 (Tiptap `setLink`) | `.me-pop` 팝오버 + `commands.setLink(href)`/`unsetLink` + `linkHrefAt(state)` | Mod-K capture 핸들러가 코어 prompt 폴백을 대체 |
| 이미지 업로드 확장 | `.me-pop` file input → FileReader → `commands.insertImage(src, alt)` | 제품에서는 업로드 API로 교체 |
| 코드블록 `language` (hljs 등) | `attrs.language` → `pre[data-language]` + `commands.setCodeLanguage`/`setCodeLanguageAt` | 언어 별칭 매핑 — bash/java/sql/rust 등, 미지원은 generic 폴백. 멀티블록 선택은 하나의 codeBlock으로 병합 |
| 코드블록 크롬 (언어/복사 버튼) | `nodeViews.codeBlock` 컴포넌트 — `.me-codeview`/`.me-codehead` 카드 헤더 + `formatCodeAt`/`blockText` | nodeViews 미등록 시 코어 `.me-codeui` 오버레이 폴백 |
| 코드블록 lowlight/hljs 데코레이션 (Tiptap) | `NodeSpec.codeHighlighter` + `highlightCode` 토크나이저 | 뷰 레벨 토큰 스팬 — doc 불변 |
| `enableTabIndentation` (Tiptap) | `indentCode`/`dedentCode` — codeBlock 키맵 내장 | Tab=`\t`, Shift-Tab=탭/스페이스4 제거 |
| `exitOnArrowDown`/`exitOnArrowUp` (Tiptap) | `exitCodeArrow` — codeBlock 키맵 내장 | 이웃 없으면 빈 문단 생성 |
| Notion 코드블록 Copy 버튼 | `codeTextAt(state)` + 데모 ⧉ 버튼 | 클립보드에 블록 전체 텍스트 |
| GDocs 링크 칩 (클릭→열기/복사/수정/제거) | `linkRangeAt(state)` + `.me-linkcard` DOM 계약 | collapsed 캐럿의 연속 링크 범위 |
| GDocs 표시텍스트 편집 | 데모 `applyLink` (data.ts) | 팝오버 text 필드 → replaceText + 마크 재적용 |
| 선택 위 URL 붙여넣기→링크 (GDocs/Notion) | input.ts `insertFromPaste` 내장 | 텍스트 보존, 마크만 부여 |
| Summernote `onImageUpload` | `EditorOptions.onImageUpload(file)` | paste/drop 파일→src; 없으면 FileReader data URL |
| Ace behaviours (자동 괄호 쌍/타입오버) | input.ts 코드블록 입력 경로 내장 | `([{"'` 쌍 삽입·감싸기, 닫기 타입오버, 빈 쌍 Backspace 삭제 |
| Ace `toggleCommentLines` (`Ctrl-/`) | `codeComment` 커맨드 — `Mod-/` 키맵 내장 | 언어별 `//`·`#`·`/* */`·`<!-- -->`, 들여쓰기 보존 |
| Ace `movelines` (`Alt-↑/↓`) | `moveCodeLines` 커맨드 — codeBlock 키맵 내장 | 선택 줄 통째 이동 |
| Ace `duplicateSelection` (`Ctrl-Shift-D`) | `duplicateCodeLines` 커맨드 — `Mod-Shift-D` | 선택 줄 바로 아래 복제 |
| Ace bracket matching | `codeBracketAt(state)` + `::highlight(me-bracket)` | Custom Highlight API — DOM 무영향 |
| Ace `showInvisibles` | `.me-ws`/`.me-ws-tab` 스팬 + `[data-me-invisibles]` | 글리프가 아닌 배경이라 캐럿 좌표 불변 |
| Ace `wrap` | `.me-frame[data-me-wrap]` → `pre-wrap` | 뷰 토글 — doc 불변 |
| Ace `gutter`/`highlightActiveLine`/`showPrintMargin` | `.me-cl` CSS 카운터 + `is-active` + `--me-print-margin` | 줄번호는 DOM 노드 아님 |

<!-- 새 기능 행을 여기에 추가 -->
