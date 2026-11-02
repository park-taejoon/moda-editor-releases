# 협업/CRDT 어댑터 (bindCollab)

transport-무관 스텝 브로드캐스트 — `Point{path,offset}` + 순수 JSON
`Step`이 이미 전제 조건이라 Yjs 없이도 원격 반영이 된다.
**최소 어댑터다 — 완전한 충돌 해소 엔진이 아니다.** 진짜 공존 편집
(동시 편집 수렴)은 Yjs 바인딩 같은 별도 계층이 맡는다.

```ts
import { bindCollab } from "@moda-editor/core";

const channel = new BroadcastChannel("doc-1"); // 또는 직접 구현
const unbind = bindCollab(
  editor,
  {
    send: (m) => channel.postMessage(m),
    subscribe: (h) => {
      const fn = (e: MessageEvent) => h(e.data);
      channel.addEventListener("message", fn);
      return () => channel.removeEventListener("message", fn);
    },
  },
  { peerId: "tab-1", announce: true },
);
```

## 동작 모델

- 로컬 `editor.dispatch` → transaction 이벤트 훅이 tr의 `steps`를
  `{kind:"steps", origin, steps}`로 보낸다.
- 원격 메시지가 오면 `tr.addStep`으로 재생한다 — `REMOTE_META`
  마커를 달아 자기 메시지가 다시 전송되는 에코 루프를 막는다.
- `announce: true`면 바인딩 직후 `{kind:"doc"}` 스냅샷을 보낸다 —
  늦게 접속한 피어가 현재 문서를 받는다.
- `subscribe`의 반환값과 `bindCollab`의 반환 함수로 수명주기를
  정리한다 — 해제 후에는 송수신 모두 멈춘다.

## 메시지 계약

```ts
type CollabMessage =
  | { kind: "steps"; origin: string; steps: Step[] }
  | { kind: "doc"; origin: string; doc: DocNode };
```

- `origin`은 피어 식별자 — `peerId` 옵션으로 지정하거나 자동 발번.
- 전송 계층(`CollabChannel`)은 `send`/`subscribe` 두 메서드뿐이다 —
  BroadcastChannel, WebSocket, 테스트용 루프백 어디든 맞추면 된다.
  테스트/로컬 데모용 루프백은 `createCollabBus()`가 제공한다 — 두
  에디터를 같은 버스에 물리면 즉시 상호 반영된다(송신자 에코는
  수신 측 origin 판별로 걸러진다).

## 한계

- **동시 편집 충돌 해소 없음** — 스텝은 그대로 재생되므로 같은 위치를
  동시에 바꾸면 한쪽이 덮인다. "한 번에 한 명이 쓰는" 협업·원격
  반영·리플레이 용도다.
- 진짜 동시 편집 수렴은 `@moda-editor/collab-yjs`의 `bindYjs`가 맡는다 —
  Y.Doc을 문서의 진실로 두고 로컬 스텝을 Y 연산으로 직역하며, 원격 변경은
  observeDeep delta를 최소 스텝으로 역직역해 반영한다. 전송 계층
  (y-websocket 등)은 사용자 몫이다.

## 원격 커서 — createCollabCarets (collab-yjs)

awareness(y-protocols) 기반 원격 캐럿/선택 표시:

```ts
const carets = createCollabCarets();
const editor = createEditor({
  extensions: [...starterExtensions(), carets.extension], // 생성 시 포함
});
carets.bind(editor, awareness, { ydoc, user: { name, color } });
```

- 로컬 셀렉션을 `Y.RelativePosition`으로 인코딩해 발행 — 동시 편집으로
  문서가 밀려도 원격 커서가 같은 문자를 추적한다 (절대 좌표면 어긋난다)
- 원격 캐럿은 `paintOverlay`로 `.me-carets` 레이어에
  `.me-remote-caret`+`.me-remote-caret-label`(이름 깃발)·`.me-remote-sel`
  (선택 rect)을 그린다 — 위젯 데코레이션이 없는 코어 계약상 문서 부수
  DOM의 공식 경로. 피어 색은 요소의 `--me-remote-color` 인라인 변수
- 데모는 같은 BroadcastChannel에 `{t:"awareness"}` 메시지를 얹어
  중계한다 (`apps/dev-shared`의 `bindCollabDemo` 참고)

## 협업 undo — createCollabUndo (collab-yjs)

`Y.UndoManager`를 에디터 undo/redo와 연결 — 코어의 Step 역변환
history는 절대 좌표라 원격 편집이 섞이면 스택 top의 **남의 편집**까지
되돌린다. 여기서는 `trackedOrigins`를 로컬 업로드 origin으로 제한해
스택이 내 변경만 추적하고, 복원은 CRDT 상대 위치 기반이라 인터리빙된
원격 편집 사이에서도 정확하다:

```ts
const COLLAB_UNDO = createCollabUndo();
const editor = createEditor({
  extensions: [...starterExtensions(), COLLAB_UNDO.extension], // 생성 시 포함
});
COLLAB_UNDO.bind(editor, ydoc); // bindYjs와 같은 시그니처
```

- Mod-z/Mod-Shift-z/Mod-y 키맵과 `commands.undo/redo`를
  `Y.UndoManager`로 전환 — 미바인드 시 코어 history로 폴백
- undo가 일으키는 Y 변경은 일반 원격 반영과 같은 경로로 로컬
  dispatch·피어 전파된다 — 수렴 유지
- 각 tr의 직전 셀렉션을 `Y.RelativePosition`으로 스택 아이템에
  싣고 undo/redo 후 커서를 논리적 자리로 복원한다

## flat-Y 인코딩 — bindYjsFlat (collab-yjs)

`bindYjs`의 대체 인코딩 바인딩 — 문서를 중첩 Y 트리가 아니라
단일 루트 `Y.Array`의 마커/leaf 아이템열로 표현한다. 배경·설계는
`docs/design/flat-y.md` 참고.

```ts
import { bindYjsFlat } from "@moda-editor/collab-yjs";

const unbind = bindYjsFlat(editor, ydoc, { rootName: "moda-flat" });
```

- **왜:** deep 인코딩은 컨테이너 경계 split/join이 Y 아이템 재부착
  불가 때문에 서브트리를 재구축하고, 그 안에 anchor된 동시 원격
  편집이 tombstone에 고아화돼 유실된다. flat은 구조 변경이 마커
  연산만 하므로 콘텐츠 아이템이 절대 재구축되지 않는다.
- **공존:** 기본 루트명은 deep `"moda"` / flat `"moda-flat"`으로
  달라 한 `Y.Doc`에 공존한다 — 점진적 cutover용. `bindYjs`는
  그대로 유지되며 flat은 opt-in이다.
- **마이그레이션:** `bindYjsFlat(editor, ydoc, { migrateFrom:
"moda" })` — flat 루트가 비어 있으면 deep 루트의 스냅샷을
  일괄 변환한 뒤 평소처럼 pull한다. 독립 함수
  `migrateDeepToFlat(ydoc, {from, to})`도 export된다(멱등).
  flat 루트의 `"v"` 슬롯은 와이어 포맷 버전(`FLAT_VERSION`).
- **캐럿/undo:** `createCollabCarets`/`createCollabUndo`는 루트
  형태를 자동 감지한다 — `rootName: "moda-flat"`만 넘기면 된다.
- 데모 앱은 `?me-collab=flat` 쿼리로 flat 바인딩을 켠다
  (`apps/dev-shared`의 `bindCollabDemo`).
