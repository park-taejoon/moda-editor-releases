# 블록 드래그&드롭 (blockDndFeature)

최상위 블록 재배치 — Notion의 ⋮⋮ 핸들 관례.
`starterFeatures()`에 포함된다.

```ts
editor.commands["moveBlock"]!(from, to);
// from: 원래 최상위 인덱스, to: "원래 배열의 삽입 경계"
// — 블록0을 블록2 뒤로 옮기려면 moveBlock(0, 3)
//   (from 제거 시 생기는 인덱스 시프트는 커맨드가 정규화한다)
```

## 동작 모델

- 블록 위에 마우스를 올리면 왼쪽 거터에 `.me-drag-handle`(⋮⋮)이 뜬다.
- 핸들을 드래그하면 `application/x-me-block` MIME으로 DataTransfer를
  태우고, `dragover` 중 `.me-drop-line`이 삽입 경계를 표시한다.
- `drop`이 오면 `moveBlock` 커맨드가 실행된다 — 같은 트랜잭션
  경로라 undo에 잡힌다.
- 파일 드롭 등 다른 MIME은 관여하지 않는다 — 이미지/파일 업로드
  경로와 충돌하지 않는다.

## DOM 계약

```text
.me-root
├─ .me-editor[contenteditable]  ← 편집 DOM
└─ .me-dnd                      ← 오버레이 (pointer-events:none)
   ├─ .me-drag-handle          ← 호버 블록의 거터 자리
   └─ .me-drop-line            ← dragover 중 삽입 경계 표시
```

- 오버레이는 `.me-editor`의 **형제**다 — 편집 DOM 안에 들어가면
  `data-me-path` 매핑과 캐럿 계산을 깬다.
- `bindEditorInput`이 바인딩 시점에 오버레이를 한 번 그린다 —
  React/Vue2처럼 마운트 layoutEffect가 바인딩보다 먼저 도는
  렌더러에서도 핸들이 즉시 존재한다.
- 핸들만 `pointer-events: auto` — 래퍼와 드롭 라인은 텍스트
  클릭을 가로채지 않는다.
