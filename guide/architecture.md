# 아키텍처

## 헤드리스 코어 + 얇은 어댑터

```
옵션 → WidgetCore(상태) → getSnapshot()(불변 스냅샷) → subscribe 리스너
                                                     ↓
                          React useSyncExternalStore / Vue shallowRef /
                          Svelte readable / vanilla 직접 렌더
```

- **코어(`@moda-editor/core`)** — 모든 로직. 프레임워크·DOM에 무관하다
  (`mountWidget`만 DOM을 만진다). 상태를 바꾸면 내부에서 스냅샷을 통째로
  다시 계산하고 구독자에게 통지한다.
- **어댑터(`@moda-editor/*`)** — 코어 스냅샷을 각 프레임워크의 반응성
  시스템으로 연결하고 DOM 계약에 맞게 렌더한다. 로직을 두지 않는다.

## 핵심 계약

1. `getSnapshot()`은 상태가 바뀌기 전까지 **같은 참조**를 반환한다 —
   React `useSyncExternalStore`·Svelte `$derived`가 이에 의존한다.
2. setter는 **동일 값 재호출에 멱등** — 어댑터의 반응형 동기화 루프를
   깨지 않는다.
3. 모든 렌더러는 **같은 DOM 계약**(`me-*` 클래스)을 만든다 —
   `runWidgetConformance`가 5개 렌더러를 같은 스펙으로 검증한다.
4. 이벤트는 `events` 옵션 + `widget.on(type, fn)` — 어댑터는 이벤트를
   프레임워크 emit/prop 콜백으로 변환한다.

## 파일 배치 규칙

| 위치                               | 내용                                    |
| ---------------------------------- | --------------------------------------- |
| `packages/core/src/features/`      | 기능 모듈 — Feature 계약 + 기능별 테스트 |
| `packages/core/src/commands.ts`    | 기능 제작 툴킷 — 커맨드·입력 규칙 팩토리 |
| `packages/core/src/presets/`       | 스타터 조립 + 구문 하이라이트           |
| `packages/core/src/types.ts`       | 옵션·스냅샷·이벤트 타입                 |
| `packages/core/src/core.ts`        | 상태 로직 + 스냅샷 계산                 |
| `packages/core/src/mount.ts`       | vanilla DOM 렌더러                      |
| `packages/core/src/conformance.ts` | 5렌더러 공용 DOM 계약 스펙              |
| `packages/*/src`                   | 어댑터 — 바인딩만, 로직 없음            |
| `apps/dev-*`                       | 데모 — 기능 체크리스트 + 실제 옵션 사용 |
| `e2e/`                             | Playwright — helpers.ts의 apps 루프     |
| `docs/guide/`                      | 기능·플랫폼 가이드                      |

## 에디터 엔진

위젯 골든 패스 위에 실제 에디터 코어가 올라가 있다 — 구조는 같다:
`createEditor()(불변 EditorState = 스냅샷) → subscribe → 어댑터`.

```
트랜잭션 = Step[](순수 JSON) → EditorState.apply() → 새 상태(구조 공유)
   ↓                                       ↓
filterTransaction ──→ appendTransaction → notify → 어댑터 재렌더
```

- **문서** — `EditorNode` JSON AST. `Point{path,offset}` 좌표 —
  독립 직렬화 가능 (CRDT 매핑 전제)
- **뷰 계층** (`core/src/view/`) — 프레임워크 비의존:
  - `render.ts` — doc → `RenderNode` 트리 (스키마 `toDOM` 반영)
  - `dom.ts` — `data-me-path` 기반 DOM↔Point 변환
  - `input.ts` — `beforeinput`→트랜잭션, `selectionchange`→doc 동기화
  - `mount.ts` — `mountEditor` vanilla 렌더러
- **어댑터** — `useEditor`(React/Vue3/Vue2), `createEditorStore`(Svelte) +
  `EditorView`가 RenderNode 트리를 번역하고 `bindEditorInput`을 붙인다

컨포먼스: `runEditorConformance`가 5개 렌더러의 에디터 DOM 계약을 검증.

## 기능 모듈 (`features/`)

에디터 기능은 `features/` 아래 파일 하나 = 기능 하나다. 각 모듈은
`Feature` 인터페이스(`defineFeature`)를 만족하는 자기완결 번들로,
스키마(nodes/marks) + 커맨드 + 키맵 + 입력 규칙 + 뷰 입력 훅을
같은 파일에 둔다:

```ts
export const myFeature = defineFeature({
  name: "myFeature",
  description: "한 줄 설명 — 기능 목록·가이드 표에 쓴다",
  nodes: { ... }, commands: { ... }, keymap: { ... },
  inputRules: [ ... ], input: { insertText(ctx, data, type) { ... } },
});
```

- `features/feature.ts` — Feature 계약. Extension의 하위 타입으로
  `name`+`description`을 필수로 해 기능 인벤토리·문서·테스트의
  경계를 맞춘다.
- `commands.ts` — 기능 제작 툴킷 (toggleMark·caretBlockPath·
  markRule·blockRule·wrapBlock·blockTextOffset/Point 등). 두 개
  이상의 기능이 쓰는 공용 헬퍼와 뷰 계층이 필요로 하는 커맨드는
  여기 둔다 — **의존 방향 계약**: 코어 계층(model/state/view/
  editor)은 `features/`를 참조하지 않고, 기능만 코어를 참조한다.
- `Extension.input` (InputHooks) — 기능이 뷰 입력 단계에 끼어드는
  공식 훅: `insertText`/`insertParagraph`/`deleteBackward`/
  `insertFromPaste`(RangeCtx + 클립보드 데이터)/`files`(FileDropCtx —
  paste/drop 이벤트의 File 목록) 인터셉터 (true=소비, false=폴스루 —
  keymap과 같은 모델)와 `paintOverlay` DOM 오버레이. 코드블록의
  자동 쌍·타입오버·자동 들여쓰기·순수 텍스트 붙여넣기·괄호
  하이라이트, 링크의 선택 위 URL 감싸기, 이미지의 파일 업로드,
  ace의 활성 라인이 이 계약으로 구현돼 input.ts는 기능을 모른다.
- `features/index.ts` — `starterFeatures()`가 배열 순서로 조립한다.
  **순서가 계약**: keymap은 앞쪽 확장부터 매칭해 false면 다음으로
  폴스루하고(Tab: codeBlock → lists → table), inputRules·input
  훅도 같은 순서로 실행된다.
- `features/<name>.test.ts` — 기능별 테스트. 공용 에디터/타이핑
  헬퍼는 `features/testkit.ts`가 제공한다.
- `presets/starter.ts` — 기능들 + historyExtension을 묶는 얇은
  조립 셈. 기존 임포트 경로는 전부 재수출로 유지한다.

새 기능 추가 = 파일 하나 + testkit 기반 테스트 + index.ts 등록.
`docs/guide/extending.md`의 워크플로를 따른다.

## 붙여넣기 파이프라인

```
clipboard text/html → sanitizeHTML(위험 요소/속성 제거)
  → parseHTML(스키마 parseDOM 규칙 매칭, 미매칭 태그 언랩)
  → tr.insertSlice (인라인/블록 분기) → dispatch
```

- **sanitize**는 "위험한가"만 본다 — 스키마 밖 태그의 생존은 파서가 결정
- **스타일 규칙** — `font-weight=bold|[6-9]00` 형태로 인라인 스타일을
  마크로 인식 (구글독스/워드 붙여넣기)
- **`insertSlice`** — 단일 텍스트블록은 인라인으로 풀어삽입(PM 관례),
  블록 조각은 현재 블록을 나눠 사이에 삽입
- `serializeToHTML` — doc → HTML. data-me-path는 내부 계약이라 제외

## 히스토리 (undo/redo)

Step이 순수 JSON이라 (step, docBefore) → 역스텝 역변환이 가능하다 —
`history.ts`의 `invertStep`/`invertSteps`가 이를 구현한다.

```
docChanged tr → plugin.apply가 {steps, inverse, selBefore/After} 엔트리를
done 스택에 push → undo 커맨드는 inverse 스텝을 역순으로 디스패치
```

- **그룹핑** — `tr.meta["history.group"]`이 같으면 한 엔트리로 병합.
  input.ts는 타이핑을 "type", 삭제를 "delete"로 묶는다. 단 같은
  그룹이라도 마지막 편집과 `GROUP_DELAY`(500ms — PM의
  `newGroupDelay` 관례)를 넘으면 별도 엔트리로 끊는다 — 시간 창이
  없으면 한 세션의 모든 타이핑이 하나로 뭉쳐 undo 한 번에
  문서 전체가 날아간다
- **redo** — undo가 이동시킨 엔트리의 원 스텝을 재적용. 새 편집은
  undone 스택을 비운다
- **키맵** — input.ts의 keydown 핸들러가 `editor.keymaps`를 순서대로
  매칭, 첫 true 커맨드가 이기고 false는 폴스루 (PM 관례)
- 한계: removeMark 역변환은 마크 attrs 미복원, join 역변환은
  next 노드 attrs 미복원 (join 자체가 유실시키므로)

## IME 조합 입력

한글/일본어 IME는 beforeinput을 막으면 조합이 깨지므로, composition
중에는 브라우저가 DOM을 직접 바꾸게 두고 compositionend에서 흡수한다
(`view/input.ts`).

```
compositionstart → composing=true (beforeinput/keydown/selectionchange 무시)
  브라우저가 텍스트 홀더 DOM을 직접 변경
compositionend → setTimeout으로 composing 해제 지연
  → DOM 선택이 가리키는 텍스트 홀더의 textContent와 doc 텍스트를
    공통 접두/접미로 diff → 차이만큼 delete+insert 트랜잭션
  → 리렌더로 DOM이 doc 상태로 정규화
```

- **지연 해제가 핵심** — Safari는 compositionend 뒤 같은 태스크에 최종
  beforeinput(insertCompositionText)을, Chromium은 조합 확정 Enter의
  keydown을 같은 태스크에 보낸다. 타이머까지 composing을 유지해야
  이중 삽입과 Enter 오커밋을 둘 다 막는다.
- **커밋-Enter의 구조 입력** — Chromium은 조합 커밋 Enter에
  compositionend 직후 insertParagraph beforeinput을 같은 창에
  보낸다. 이걸 composing 가드가 그냥 리턴하면 브라우저가 네이티브
  split으로 `data-me-path`가 복제된 블록을 만들어 doc과 영구히
  어긋난다 ("Enter를 두 번 쳐야 내려간다" 회귀). compositionend에서
  `pendingAbsorb`를 세워 두고, 그 창에 도착한 구조 입력
  (insertParagraph/LineBreak/delete*/insertFromPaste/Drop)은
  `flushComposition`으로 흡수를 앞당긴 뒤 정상 처리한다. 텍스트
  계열 입력은 계속 브라우저에 맡긴다 — Safari 최종 커밋 텍스트를
  막으면 글자가 사라진다. DOM 셀렉션은 플러시 전에 읽어둔다 —
  커밋 끝점 좌표라 흡수된 텍스트를 포함해 정확하고, 리렌더가
  셀렉션을 조합 시작점으로 되돌리기 전의 값이다.
- 흡수된 텍스트는 `history.group: "type"`으로 일반 타이핑과 같은
  undo 그룹에 들어간다.
- **외부 DOM 변경 → 리마운트 계약** — 조합 중 브라우저가 바꾼 DOM은
  vnode와 어긋난다 (빈 블록의 `<br>` 자리표시자를 지우고 텍스트를 넣는
  식). 그대로 diff하면 프레임워크 리컨실이 크래시한다 (실측:
  React `removeChild` 예외→프리즈). 바인딩이 흡수한 블록의 경로를
  `binding.dirtyPaths`에 기록하고, 어댑터는 `DirtyKeyMap`으로 그 블록의
  렌더 키를 버전업해 통째 리마운트한다 — 리마운트는 부모에서 블록
  엘리먼트만 detach하므로 항상 안전하다. 새 어댑터도 같은 계약을 구현해야
  한다.
- **무효 DOM 셀렉션 폴백** — 리렌더로 캐럿 노드가 detach된 채 입력이
  오면 DOM 셀렉션이 읽히지 않는다. 이때 입력을 흘려보내면 브라우저가
  DOM만 바꿔 doc과 영구히 어긋나므로, `beforeinput`은 doc 셀렉션으로
  폴백해 트랜잭션을 만든다.
- 제약: 조합이 하나의 텍스트 홀더 안에 머문다고 가정 — 조합 시작 시
  여러 블록에 걸친 선택 삭제는 미지원.

## beforeinput 커버리지

브라우저가 DOM을 직접 바꾸는 모든 입력은 트랜잭션으로 환원해야
doc이 어긋나지 않는다 — 처리하지 않는 inputType도
`preventDefault`로 막아 네이티브 변형을 원천 차단한다.

- 텍스트 — insertText/insertCompositionText(쌍·타입오버·규칙),
  insertParagraph/LineBreak(구조 분기), insertReplacementText·
  insertFromYank·insertFromPasteAsQuotation(교체 범위는
  `getTargetRanges`)
- 붙여넣기/드롭 — insertFromPaste는 `input.insertFromPaste` 훅에
  먼저 넘긴다(링크의 선택 위 URL 감싸기, 코드블록 순수 텍스트),
  insertFromDrop은 텍스트 드롭을 트랜잭션으로. 파일은 paste/drop
  DOM 이벤트가 `input.files` 훅에 넘긴다(이미지 업로드) —
  어느 기능도 소비하지 않으면 beforeinput/기본 동작으로 폴스루
- 삭제 — deleteContentBackward/Forward(캐럿·쌍 삭제), deleteWord*·
  deleteSoftLine*·deleteHardLine*·deleteEntireSoftLine·deleteByDrag·
  deleteByCut은 `getTargetRanges`의 브라우저 계산 범위로 지운다
  (macOS Option-Delete·Ctrl-K·Cmd-X). 캐럿 Backspace/Delete도
  targetRange를 우선 써서 이모지 서로게이트 페어·결합문자 같은
  그래피임 클러스터가 통째로 지워진다. 잘라내기의 클립보드 쓰기는
  `cut` 이벤트의 디폴트라 beforeinput을 막아도 채워진다.
  **타겟 레인지 신뢰 규칙** — Chrome은 블록 경계에서 ① 캐럿 "뒤"
  글자 범위(반대 방향)나 ② (P,0) 같은 요소 레벨 끝점을 주는 버그가
  있다. 방향에 맞는 끝점이 캐럿과 일치하고 양 끝점이 텍스트 리프
  포인트일 때만 신뢰하고, 아니면 `pointBefore/pointAfter` 폴백으로
  블록 머지한다 (`view/input.test.ts`가 계약을 고정한다)
- 서식 — formatBold/Italic/Underline/StrikeThrough/Superscript/
  Subscript → toggleMark 매핑 (모바일·컨텍스트 메뉴 경로),
  formatRemove → clearFormat, formatJustify* → setAlign,
  formatIndent/Outdent → 리스트 싱크/리프트,
  insertOrderedList/UnorderedList → 리스트 토글,
  insertHorizontalRule → hr 삽입
- 히스토리 — historyUndo/historyRedo → 우리 undo/redo
- 미지원(insertTranspose·formatFontColor 등) → preventDefault no-op.
  네이티브 변형이 항상 doc과 어긋나므로 방치보다 차단이 안전하다

## 성능 — 렌더 재조정과 메모이즈

불변 doc + 구조 공유의 파생 이점: `prev.content[i] === next.content[i]`이면
그 서브트리 DOM을 건드릴 필요가 없다.

- **vanilla mountEditor** — 블록 단위 재조정. doc 참조가 같으면
  (selection-only 변경) 렌더를 통째 스킵하고, 아니면 바뀐 블록만
  `replaceWith` 한다. happy-dom 기준 keystroke 1.76ms → 0.26ms.
- **React** — `useMemo([state.doc])`로 엘리먼트 트리 메모이즈.
  같은 엘리먼트 배열 참조면 리액트가 서브트리 재조정을 스킵.
- **Vue 3 / Vue 2** — computed로 RenderNode 배열 메모이즈.
  같은 배열 + 같은 `:node` prop 참조 → 자식 컴포넌트 재렌더 스킵.
- **Svelte** — `$derived.by`로 배열 메모이즈. 같은 배열이면
  `{#each}` 블록 자체가 스킵된다.
- 컨포먼스 계약: "한 블록만 바뀌면 나머지 DOM 노드 보존",
  "selection-only 디스패치는 DOM 노드 보존" — 5개 렌더러 전부
  `toBe(동일 엘리먼트)`를 통과한다.
- 데모앱 `?big` 쿼리로 300블록(1만 자+) 문서를 마운트할 수 있다.

## 커스텀 노드 뷰 (nodeViews)

스키마의 `toDOM`으로 표현할 수 없는 노드는 프레임워크 컴포넌트로
주입한다. 코어는 `RenderNode.source`(원본 AST 노드)와
`NodeViewProps{node, editor}` 계약만 제공하고, 각 어댑터가 자기
프레임워크의 컴포넌트 타입으로 번역한다.

```
EditorView nodeViews={{ callout: CalloutView }}
  → RenderNode.source.type === "callout" 매칭
  → .me-node-view 래퍼(contenteditable=false) 안에 컴포넌트
```

- 스키마 `toDOM`은 nodeViews 미등록/직렬화 시의 폴백으로 남는다.
- vanilla는 `(props) => HTMLElement` 팩토리를 등록한다.

노드 뷰는 스키마 `atom` 여부에 따라 두 가지 DOM 계약으로 렌더된다:

```
atom: true (편집 불가 섬)
  .me-node-view[data-me-path][contenteditable="false"]
    └ 컴포넌트 크롬 (내부 좌표는 부모 경계로 환원)

콘텐츠 블록 (content: "block+" 등) — 편집 가능 아울렛
  .me-node-view[contenteditable="false"]
    └ 컴포넌트 크롬
        └ .me-node-content[data-me-path][contenteditable="true"]
            └ AST 자식 엘리먼트 1:1 — 여기서 타이핑/분할이 일어난다
```

- **atom** — 래퍼가 data-me-path를 갖고 안쪽은 통째 비편집 섬.
  DOM 셀렉션이 안쪽에 놓여도 부모 경계 포인트로 환원한다.
- **콘텐츠 블록** — NodeViewProps의 `content` 아울렛을 컴포넌트가 자기
  크롬 안에 배치한다 (React: `content` ReactNode, Vue: `content` 슬롯,
  Svelte: `content` 스니펫, vanilla: `content` HTMLElement). 아울렛
  엘리먼트 자체가 data-me-path + ce=true 계약을 가지므로 컴포넌트는
  위치만 정하면 된다. data-me-path가 래퍼가 아닌 아울렛에 있어야
  컨테이너 포인트가 AST 자식과 1:1로 매핑된다. 아울렛을 렌더하지
  않으면 자식 블록이 화면에 나오지 않는다.
- 아울렛 안의 편집(타이핑·Enter 분할·삭제)은 일반 블록과 같은
  트랜잭션 경로를 탄다 — 트랜잭션 계층이 중첩 path를 그대로 처리한다.
- **중첩 editable과 셀렉션** — ce=false 래퍼 안에 ce=true 아울렛이
  있으면 Chromium의 네이티브 select-all이 셀렉션을 펼치지 못하고
  문서 시작으로 collapse한다 (실측 재현). 두 겹으로 우회한다:
  - `keydown`의 `Mod-a`를 가로채 doc 전체 선택 트랜잭션을 직접
    디스패치한다.
  - `pointToDOM`은 컨테이너 경계 포인트(루트 포함)를 리프 텍스트
    위치로 내려 DOM 셀렉션을 만든다 — contenteditable 요소의
    `(요소, 자식인덱스)` 경계 셀렉션은 중첩 editable을 건널 때
    Chromium이 조용히 collapse시키기 때문이다. 루트 포인트는
    `data-me-path=""`가 root 자신에 있어 `querySelector`가 매칭
    못 하므로 path=[]를 root로 직접 해석한다. 자식이 ce=false
    크롬이면 이전 형제의 마지막 리프로 돌아간다.

## 트랜잭션 좌표 의미

스텝은 순차 적용되므로 빌더 메서드의 위치 인자는 **"직전 스텝까지
적용된 문서" 좌표**다(PM 관례). `TransactionBuilder`는 내부적으로
누적 스텝을 `applyStep`으로 시뮬레이션한 `docNow` 기준으로 위치를
해석한다 — `tr.delete(sel)` 뒤 `insertText`가 삭제된 노드를 가리키는
일이 없다.

브라우저 DOM 셀렉션은 컨테이너 경계·atom 노드 안 등 문서 모델이
허용하지 않는 좌표를 만들 수 있으므로, 경계 해석이 빌더 안에 있다:

- `resolveInsert` — 인라인 컨테이너 경계는 가장 가까운 텍스트 위치로,
  atom 리프 안은 부모 경계로, 블록 컨테이너(doc 루트 등) 경계는
  새 텍스트 블록 삽입으로 정규화한다. 전체 삭제 후 타이핑이
  루트 bare text가 아닌 paragraph 안 텍스트가 되는 이유다.
- `resolveBoundary` + `docOrderCmp` — 삭제 범위 정규화. path 사전순
  비교(pointCmp)는 컨테이너 경계 포인트의 문서 순서를 뒤집을 수
  있어 삭제는 전용 비교자로 정렬한 뒤 atom 경계를 부모 경계로 올린다.
- `deleteOrdered` — 같은 노드/같은 부모/일반 LCA 세 단계로 분해해
  교차-블록·교차-컨테이너 범위를 역순 제거로 처리한다.

## 입력 규칙 (마크다운 변환)

`Extension.inputRules`는 타이핑으로 텍스트가 삽입된 직후 실행되는
변환 규칙이다 — `Extension` 스캐폴딩에 이미 필드가 있었고, 실행 경로를
`tr.applyInputRules`로 배선했다.

```
beforeinput(insertText) → tr.insertText(문자) → tr.applyInputRules(rules)
  → 블록 시작~커서 텍스트를 만들어 각 규칙의 pattern에 매칭
  → 커서에서 끝나는 첫 매치가 같은 tr 위에 변환 스텝을 쌓음
  → dispatch (삽입+변환 = 하나의 트랜잭션 = 하나의 undo 단위)
```

- **핸들러 계약** — `(ctx, match, range, doc)`. `range`는 누적 스텝
  적용 후 좌표의 매치 범위(focus=캐럿), `doc`은 누적 스텝 적용 후
  문서. docNow 좌표계라 핸들러가 쌓는 스텝도 직전 스텝 기준으로
  해석된다. `ctx.dispatch`는 no-op — 호출부가 tr 전체를 디스패치한다.
- **`markRule(pattern, markType, open, close)`** — fence 위치를
  매치 끝에서 문자열 길이로 역산해 닫힘→열림 순으로 삭제하고 안쪽에
  addMark. 패턴이 앞쪽 컨텍스트를 포함해도 안전하다.
- **`blockRule(pattern, fn)`** — 블록 시작 패턴이면 `fn`이 돌려준
  노드 조각으로 블록을 교체하고 접두사를 지운다 (`# `~`###### `).
- **fence 모호성** — `*`/`_` 규칙은 `(?:^|[^*])` 계열 접두 가드가
  필요하다. 없으면 `**x*` 입력 중간 상태에서 `*x*`가 italic으로
  먼저 변환돼 `**x**` 완성이 불가능하다 (실제 브라우저 회귀로 잡음).
- 스타터 규칙: `#`~`######`→heading, `**`→bold, `*`/`_`→italic,
  `~~`→strike, `` ` ``→code, `-`/`1.`→리스트, `>`→blockquote,
  ```` ``` ````/`~~~`+스페이스→codeBlock(` ```js `처럼 언어 접미사 —
  공백이 트리거라 접미사를 칠 시간이 있다, Tiptap
  `backtickInputRegex` 관례), `---`→horizontalRule,
  `[텍스트](url)`→link 마크, `[ ]`/`[x]`→할일 항목. 데모의 callout은
  확장 노드 예시로 남아 있다.

## 리치텍스트 블록 (스타터)

리스트·인용구·코드블록은 블록 스키마 + 트랜잭션 빌더 연산 +
키맵 조합으로 동작한다 — 전부 코어에 있고 어댑터는 모른다.

- **리스트** — `bulletList`/`orderedList`(`content: "listItem+"`)와
  `listItem`(`content: "block+"`, 중첩 리스트 포함). 아이템 연산은
  `tr` 메서드:
  - `splitItem(itemPath, caret)` — Enter: 아이템을 둘로 쪼개 캐럿 뒤
    내용을 새 아이템으로.
  - `liftItem(itemPath)` — 최상위 아이템은 블록들을 리스트 밖으로
    꺼내고(유일/머리/꼬리/중간 — 리스트 분할 포함), **중첩 아이템은
    바깥 리스트의 형제 li로 들어올린다** — 뒤따르는 형제들은 들어올린
    아이템의 서브리스트로 딸려 나간다 (PM `liftListItem` 관례).
  - `sinkItem(itemPath)` — Tab 인덴트: 이전 형제의 서브리스트 끝으로,
    없으면 같은 타입 리스트를 새로 만들어 넣는다.
  - `liftOnItemBackspace` — 아이템 시작 Backspace = 들어올림.
    빈 아이템 Enter 탈출도 같은 `liftItem` 경로다.
- **인용구** — `blockquote`는 `content: "block+"` 컨테이너.
  `> ` 입력 규칙과 `wrapBlock` 커맨드가 캐럿 블록을 감싼다.
- **코드블록** — `codeBlock`은 `code: true` + `marks: ""` +
  `attrs.language` 스펙. Enter는 input.ts에서 `\n` 삽입으로 처리하고,
  빈 블록이거나 캐럿 끝이 빈 줄이면 꼬리 개행을 지우고 블록 뒤 본문으로
  탈출한다. `Mod-Enter`는 어느 위치에서든 탈출 (`exitBlock` 커맨드).
  `toggleCodeBlock`은 선택이 여러 최상위 텍스트블록을 걸치면 전부
  `\n`으로 잇는 하나의 코드블록으로 병합한다 — 선택 텍스트를
  코드로 옮기는 정상 경로다. `language`는 `data-language` attr로
  렌더돼 CSS 라벨이 되고, `commands.setCodeLanguage`로 바꾼다.
  - **Tab/Shift-Tab** — `indentCode`/`dedentCode` (Tiptap
    `enableTabIndentation`·VS Code 관례). collapsed면 캐럿에 `\t`,
    범위 선택이면 걸친 줄 전부의 줄 앞에 `\t` 삽입/제거(탭 하나 또는
    스페이스 최대 4개). 리스트/표 Tab 바인딩보다 먼저 시도하고
    코드블록 밖이면 폴스루한다.
  - **ArrowUp/ArrowDown 탈출** — `exitCodeArrow` (Tiptap
    `exitOnArrowUp/Down` 관례). 첫 줄의 ArrowUp은 블록 앞으로,
    마지막 줄의 ArrowDown은 블록 뒤로 나간다 — 이웃 블록이 없으면
    빈 문단을 만들어 캐럿이 갈 곳을 보장한다.
  - **Enter** — 자동 들여쓰기 (Ace auto-indent 관례): 현재 줄의
    선행 공백을 새 줄에 복사한다. 공백뿐인 꼬리 줄에서 Enter를
    누르면 꼬리를 지우고 코드블록을 빠져나간다 — 빈 줄 Enter
    탈출과 같은 규칙의 들여쓰기 변형이다.
  - **Ln:Col** — `codeCaret(state)`는 캐럿의 코드블록 내
    `{line, col}`(1-base)를 돌려준다 — 데모 상태바가 Ace의
    `Ln:Col` 표시 관례 그대로 쓴다.
  - **구문 하이라이트** — `NodeSpec.codeHighlighter(code, lang)`
    훅 + `presets/highlight.ts`의 내장 정규식 토크나이저
    (`highlightCode`). render 계층이 코드 텍스트를
    `span.me-tok.me-tok-<kind>` 토큰 스팬으로 쪼갠다 — Tiptap의
    lowlight 데코레이션처럼 **뷰 레벨**이라 doc/AST는 순수 텍스트
    그대로다. 토큰 스팬에 `data-me-path`를 붙이지 않는 것이 계약 —
    `domToPoint`는 텍스트 누적이라 분할이 안전하다. 무거운 엔진은
    같은 `CodeToken[]` 형태로 주입하면 된다.
  - **줄 번호 거터 + 활성 줄** — render 계층이 토큰을 `\n` 경계로
    `span.me-cl` 줄 래퍼로 묶는다(Ace 거터 관례). 줄 번호는
    `.me-cl::before` CSS 카운터가 그린다 — DOM 노드가 아니라
    매핑/텍스트 추출에 무영향. 개행은 각 줄 끝의 실제 텍스트 노드로
    남아 `textContent`와 DOM↔doc 오프셋이 원문 그대로다.
    활성 줄은 바인딩의 `restoreSelection`이 DOM 셀렉션 앵커가 있는
    `.me-cl`에 `is-active`를 단다(Ace `highlightActiveLine` 관례) —
    어댑터별 구현 없이 공용 경로에서 처리된다.
  - **복사** — `codeTextAt(state)`로 블록 전체 텍스트를 읽어 ⧉
    버튼이 클립보드에 넣는다 (Notion Copy 관례).
  - **자동 괄호 쌍** (Ace behaviours 관례) — `([{"'` 입력은 쌍을
    삽입하고 캐럿을 안쪽에 둔다. 선택이 있으면 내용을 감싸고,
    닫는 문자가 바로 뒤에 있으면 입력 대신 넘어간다(타입오버).
    인용부호는 단어 안쪽/앞에선 아포스트로피로 보고 쌍을 만들지
    않는다. 빈 쌍 사이의 Backspace는 쌍을 통째로 지운다.
    `codeStringMask`가 `codeHighlighter` 토큰으로 문자열/주석
    안쪽을 판정해 그 안에선 리터럴 삽입한다 (Ace `isCodeLike`) —
    라인 주석·닫히지 않은 문자열은 끝 경계도 안쪽으로 친다.
    `{|}`/`(|)`/`[|]` 사이의 Enter는 들여쓴 가운데 줄로 확장한다
    (VS Code/Ace bracket expansion).
  - **주석 토글** — `Mod-/`는 `codeComment(state)` 커맨드로
    언어별 주석(`//`, `#`, `/* */`, `<!-- -->`)을 선택 줄 전부에
    걸어 토글한다 — 들여쓰기는 보존한다 (Ace toggleCommentLines).
  - **줄 이동/복제** — `Alt-↑/↓`는 `moveCodeLines`로 선택 줄을
    통째로 위아래로 옮기고(Ace movelines), `Mod-Shift-D`는
    `duplicateCodeLines`로 바로 아래에 복제한다
    (Ace duplicateSelection).
  - **괄호 매칭** — `codeBracketAt(state)`가 캐럿 인접 `()[]{}`의
    짝을 스캔해 범위를 돌려주고, 바인딩이 CSS Custom Highlight
    API(`::highlight(me-bracket)`)로 칠한다 — DOM 노드가 아니라
    매핑 무영향 (Ace bracket matching 관례). 문자열/주석 토큰 안의
    괄호는 `codeStringMask`로 스캔에서 제외한다.
  - **보이지 않는 문자/줄바꿈** — render 계층이 공백 런을
    `.me-ws`, 탭을 `.me-ws-tab` 스팬으로 분리해 두고,
    `.me-frame[data-me-invisibles]`에서만 점/화살표 배경을
    그린다 (Ace showInvisibles). `.me-frame[data-me-wrap]`는
    코드 `white-space: pre-wrap`으로 줄바꿈을 켠다 (Ace wrap).
    둘 다 뷰 속성이라 doc은 순수 텍스트 그대로다.
- **구분선/하드브레이크** — `horizontalRule`은 atom leaf,
  `hardBreak`는 인라인 atom leaf. 둘 다 자식이 없어 렌더러는
  void 태그(`hr`/`br`)를 자식 없이 출력해야 한다 — React는
  `children` prop 자체를 생략, Svelte는 `svelte:element`를
  자식 없는 정적 분기로 처리한다 (동적 태그 + 내용 = 컴파일 경고).
- **링크** — `link` 마크(`attrs.href` → `<a href>`).
  `[텍스트](url)` 입력 규칙과 `Mod-K`/툴바 토글.
  `setLink(href)`는 인자로 받는다 — UI 없이 호출하면
  `globalThis.prompt` 폴백이지만, 데모처럼 팝오버를 두는 것이
  관례다(`linkHrefAt(state)`로 기존 href를 프리필).
  - **링크 카드** — `linkRangeAt(state)`는 collapsed 캐럿이 링크
    위에 있을 때 같은 href의 연속 텍스트 노드를 이어 하나의
    `{range, href, text}`로 돌려준다 — 데모의 `.me-linkcard`
    칩(URL + 복사/수정/제거, GDocs 관례)이 이걸 쓴다.
    `unsetLink`도 collapsed면 이 범위 전체를 벗긴다 — 텍스트는
    보존하고 마크만 제거.
  - **표시 텍스트 편집** — 팝오버의 텍스트 필드는 `applyLink`
    (데모 `data.ts`)가 링크 범위/선택 텍스트를 교체한 뒤 마크를
    다시 건다 (GDocs 링크 대화상자의 Text 필드 관례).
  - **URL 붙여넣기 래핑** — 비-collapsed 선택 위에 `http(s)` URL을
    붙여넣으면 텍스트를 덮지 않고 링크 마크로 감싼다
    (input.ts의 `insertFromPaste` — GDocs/Notion 관례).
- **플레이스홀더** — CSS-only. 문서가 빈 문단 하나뿐이면 bogus
  `<br>` 구조 덕분에 `.me-editor > p:only-child:has(...br)`가
  매칭돼 `::before`로 표시. 문구는 `--me-placeholder-text`,
  색은 `--me-placeholder-color` 변수.

## 블록 서식과 스타일 마크

- **정렬** — `paragraph`/`heading`의 `attrs.align`
  (`left`/`center`/`right`). `toDOM`이 `style="text-align:…"`을
  출력하고 `parseDOM.style`로 역파싱한다. 커맨드는
  `commands.setAlign(align)` — 인자를 받는 커맨드는 레지스트리에서
  `unknown[]` 인자를 캐스팅하는 래퍼로 감싸 등록한다.
- **스타일 마크** — `textColor{color}`/`highlight{color}`/
  `fontSize{size}`는 attrs를 인라인 `style`로 출력하는 마크.
  `commands.setMarkAttr(type, attrs)`로 선택 범위에 싣고,
  같은 type의 마크는 `addMark`가 attrs까지 교체한다(covered 구간만).
- **첨자** — `superscript`/`subscript` 마크는 상호 배타 —
  `toggleExclusive(type, other)` 커맨드가 켤 때 상대를 벗긴다.
- **서식 지우기** — `commands.clearFormat` — 범위 선택이면 스키마의
  모든 마크 type을 `removeMark`, collapsed면 `setStoredMarks([])`.

## 표·이미지·할일 목록

- **표** — `table > tableRow > tableCell > block+` 트리로 일반
  노드다(노드 뷰 불필요). 셀 안 타이핑은 일반 트랜잭션 경로.
  커맨드: `insertTable(rows, cols)`(빈 블록 교체 또는 뒤 삽입 +
  첫 셀 캐럿), `nextCell`/`prevCell`(Tab/Shift-Tab — 마지막 셀에서
  Tab은 새 행), `addTableRow`/`addTableColumn`/`deleteTable`.
- **이미지** — `image`는 `atom: true` 리프(`attrs.src/alt`→`<img>`).
  `commands.insertImage(src)`는 빈 블록을 교체하거나 뒤에 삽입하고
  본문 문단을 남긴다.
  - **파일 붙여넣기/드롭** — input.ts가 `paste`/`drop`의 이미지
    File을 잡아 `EditorOptions.onImageUpload(file) → src` 훅으로
    변환해 `insertImage`를 디스패치한다 (Summernote `onImageUpload`/
    Quill imageUploader 관례). 훅이 없으면 `FileReader` data URL이
    기본값 — 실제 제품은 여기에 업로드 API를 연결한다. 드롭은
    `caretPointFromXY`로 떨어진 지점에 셀렉션을 옮겨 삽입한다.
- **할일 목록** — `taskList > taskItem > block+`, `taskItem`은
  `checked` attr + `checkbox: true` 스펙. 렌더는 `li` 앞에
  `.me-check` 마커(`contenteditable="false"`)를 붙이고, input.ts의
  클릭 위임이 마커 클릭을 `commands.toggleTaskItem`(setAttr
  트랜잭션)으로 연결한다 — 어댑터는 몰라도 된다. Enter/Backspace/
  Tab은 listItem과 같은 경로를 탄다(`splitItem`/`liftItem`/
  `sinkItem`이 아이템 타입을 보존).

## 마크 스텝의 좌표 재해석

`addMark`/`removeMark` 스텝은 기록 시점의 범위를 들고 다니는데,
같은 트랜잭션 안에서 앞선 마크 스텝이 텍스트를 분할하면 범위의
엔드포인트가 노드 길이를 넘어선다(예: `[2,0]:2..3` → 분할 후
`[2,0]`은 "H2"로 길이 2). `applyMark`는 적용 시점에 엔드포인트를
현재 doc으로 재해석한다 — 오프셋 초과분을 뒤따르는 텍스트 형제로
넘기는 `resolveMarkPoint`. 이게 없으면 `toggleExclusive`의
`removeMark(상대)+addMark(자신)` 시퀀스와 `clearFormat`의 연속
`removeMark`, 그리고 부분 범위 마크의 undo(역스텝도 같은 범위를
분할된 doc에 들고 감)가 조용히 무동작한다.

## 선택 버블 메뉴

`.me-bubble` — 텍스트 셀렉션(비-collapsed) 중 떠는 인라인 서식 바.
코어 기능이 아니라 **제품 영역**이지만 5개 데모가 같은 DOM 계약을
공유한다: 셀렉션 rect로 위치를 잡고 `mousedown`을 `preventDefault`해
버튼 클릭이 에디터 셀렉션을 뺏지 않게 한다. selectionchange →
dispatch 동기화 덕분에 드래그만 해도 doc 셀렉션이 갱신되어
버블이 자동으로 뜬다.

## 팝오버 (링크/이미지)

`.me-pop[data-me-pop]` — 버블과 같은 자리지만 **input을 품는**
패널이라 규칙이 다르다: `mousedown`을 막지 않아 input에 실제
포커스가 간다 — doc 셀렉션은 state에 남아 커맨드가 그대로 동작한다.
데모의 공통 동작(각 앱 data.ts의 `applyPop`/`applyLink`/
`insertImageFile`/`installLinkHotkey`/`focusEditor` 헬퍼):

- 툴바 🔗/🖼와 Mod-K(document capture 단계 — 에디터 keydown보다
  먼저 가로채 코어의 prompt 폴백을 대체)로 연다.
- 셀렉션/캐럿 rect 아래에 띄우고, 열 때 input에 포커스를 직접
  준다 — 안 하면 Esc/Enter가 에디터로 간다.
- 링크 팝오버는 두 필드다: `input[data-pop-field='text']`(표시
  텍스트) + `input[data-pop-field='url']` — GDocs 링크 대화상자의
  Text/Link 필드 관례. 이미지 팝오버는 URL 필드 하나.
- 확인은 `applyLink`/`insertImage`, 제거는 `unsetLink`를 디스패치하고
  닫을 때 `.me-editor`로 포커스를 되돌린다.
- 이미지 업로드는 `<input type="file">` → `FileReader.readAsDataURL`
  → `insertImage(dataURL, 파일명)` — 서버 없이도 동작하는 데모
  관례이고, 실제 제품은 여기서 업로드 API를 호출해 URL로 바꾼다
  (paste/drop 경로는 `EditorOptions.onImageUpload` 훅 사용).

## 링크 카드

`.me-linkcard` — collapsed 캐럿이 링크 위에 있을 때 링크 아래에 뜨는
액션 칩 (GDocs/Notion 관례): `.me-linkcard-url`(앵커 — 새 탭 열기) +
`[data-card-act]` 버튼들(copy/edit/remove). `linkRangeAt(state)`로
감지하고 DOM 캐럿 rect 아래에 띄운다. `mousedown`은
`preventDefault`해 셀렉션을 유지하되, 앵커의 click은 남겨 새 탭
네비게이션이 동작한다. 팝오버가 열려 있으면 숨긴다(수정 경로).

## 툴바 / 상태 UI

에디터 UI 자체는 제품 영역이지만, 공통 판정 로직은 코어에 둔다:

- `isMarkActive(state, type)` — collapsed면 storedMarks → 커서 텍스트,
  범위면 전체 커버 시에만 true
- `isBlockActive(state, type)` — 캐럿 최상위 블록 타입 판정
- `canUndo` / `canRedo` — 히스토리 스택 깊이
- `toggleMark` / `setBlockType` / `splitBlock` — 스타터 커맨드

데모 5개 앱은 같은 DOM 계약을 공유한다 — `.me-toolbar >
button.me-tool[data-me-cmd][.is-active]` + `.me-sep` 구분선 +
`.me-statusbar[data-me-status]`. 툴바 컨테이너의 `mousedown`을
`preventDefault`해 클릭 시 에디터 셀렉션이 유지되도록 한다.

툴바는 한 줄 원칙이다 (Ace/CodeMirror의 최소 크롬 관례):

- **블록 서식 드롭다운** `.me-blockfmt` — ¶/H1~H3 버튼 열 대신
  현재 서식을 보여주는 select 하나 (GDocs paragraph-style 관례)
- **⋯ 오버플로** `details.me-overflow > summary.me-tool +
  .me-menu` — 덜 쓰는 커맨드 패널 (GDocs ⋮ 관례). `details`라
  프레임워크 상태 없이 토글되고, `ToolItem.menu` 표시로 올라간다
- **컨텍스트 버튼** — `ToolItem.ctx`("table"/"code") 항목은 해당
  블록 안에서만 ⋯ 패널에 노출된다 (TinyMCE contextual 관례).
  본줄 폭이 컨텍스트와 무관하게 일정해 한 줄이 유지된다
- `visibleTools(state)`/`menuTools(state)`/`blockFormatAt(state)`가
  공통 필터·표시 로직을 담당하고 `installMenuAutoClose()`가
  바깥 클릭 닫기를 제공한다

크롬 레이아웃은 `.me-frame` 래퍼가 담당한다 — 툴바(상단) +
`.me-editor` + 상태바(하단)를 하나의 카드로 묶고 포커스 링은
`:focus-within`으로 frame에 그린다. frame 밖에서 단독으로 쓰는
`.me-toolbar`/`.me-editor`도 각자 테두리를 갖는다.

## 빈 블록과 Enter

- **bogus `<br>`** — 텍스트가 없는 노드(빈 텍스트 홀더, 자식 없는
  컨테이너)는 렌더 트리에 `<br>` 자리표시자를 넣는다 (PM과 같은
  방식). `data-me-path`가 없어 DOM↔doc 1:1 매핑을 깨지 않으면서
  빈 블록에 높이와 캐럿 자리를 준다. 없으면 빈 문단이 0px로
  렌더돼 클릭/입력 대상이 사라진다 (실제 회귀로 잡음).
- **블록 끝 Enter → 기본 블록** — `split()`은 캐럿이 블록의 마지막
  자식 텍스트 끝에 있고 그 블록이 기본 텍스트블록(paragraph)이
  아니면, 타입 복제 대신 뒤에 새 기본 블록을 만든다. heading 끝
  Enter → paragraph (PM `splitBlock`의 `deflt` 동작과 동일).
  중간 분할은 타입을 유지한다.

## 줄 경계 이동과 셀렉션 동기화

Blink는 contenteditable의 중첩 인라인(`strong>span` 등)에서
네이티브 Home/End의 줄 경계 계산을 확률적으로 깬다(실측 30회 중
~15회 — headed·순수 ce에서도 재현). 그래서 input.ts가 Home/End를
직접 처리한다 (`lineNav`).

- **히트테스트 기반** — 캐럿의 collapsed range rect에서 y를 구하고
  `caretRangeFromPoint`/`caretPositionFromPoint`로 블록 좌·우 끝의
  DOM 위치를 잡는다. 클릭 배치와 같은 엔진이라 네이티브 줄 계산보다
  신뢰할 수 있다. Vue가 렌더하는 빈 텍스트 앵커 노드는 collapsed
  rect이 비므로 인접 한 글자 → 부모 요소 rect 순으로 근사한다.
- **doc 우선, DOM 보조** — 포커스는 DOM 셀렉션이 유효한 리프
  포인트일 때 쓰고(클릭 직후 doc 셀렉션이 stale할 수 있다),
  rect 계산은 `pointToDOM`으로 현재 DOM에 재해석한다(리렌더로
  분리된 노드 위의 브라우저 셀렉션에 흔들리지 않기 위해).
  다른 블록의 위치가 잡히면 신뢰 불가로 네이티브에 폴스루한다.
- **트랜잭션으로 적용** — DOM을 직접 쓰지 않고 `setSelection`
  트랜잭션을 dispatch한다. doc이 동기 갱신돼야 뒤따르는 비동기
  렌더 커밋이 DOM을 파괴해도 restore가 같은 위치를 다시 적용한다
  (직접 쓰기는 pending selchg 사이 커밋에 되돌아간다).
- **키 범위** — Home/End·Shift-Home/End, macOS Cmd-←/→(줄 경계
  관례), Mod-Home/End(문서 가장자리). Ctrl-←/→는 Windows/Linux의
  단어 이동이라 인터셉트하지 않는다.

`restoreSelection`은 `lastApplied`(마지막으로 DOM에 반영했거나
selectionchange로 확인한 위치)를 추적한다. DOM이 lastApplied와
다르고 **doc이 lastApplied 그대로**인 채 리프 포인트에 있으면
selectionchange가 아직 처리되지 않은 사용자 이동이다 — stale doc으로
캐럿을 되돌리지 말고 doc을 DOM에 맞춘다. doc이 트랜잭션으로 바뀐
뒤거나 DOM이 요소 경계(렌더가 노드를 교체할 때 Chrome의 재앵커)면
doc 쪽이 최신 의도라 doc을 DOM에 적용한다.
