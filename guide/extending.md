# 기능 추가 워크플로

새 기능은 아래 순서로, 같은 커밋에 반영한다 (AGENTS.md의 규칙).

## 1. 코어 — 기능 모듈

`packages/core/src/features/<name>.ts`에 `defineFeature()`로
기능을 정의한다 — 스키마(nodes/marks) + 커맨드 + 키맵 + 입력
규칙을 한 파일에 묶는 자기완결 번들:

```ts
export const myFeature = defineFeature({
  name: "myFeature",
  description: "한 줄 설명",
  nodes: { ... }, commands: { ... }, keymap: { ... },
  inputRules: [ ... ], input: { ... },
});
```

- 공용 헬퍼·커맨드는 `src/commands.ts`에, 기능 전용 헬퍼는 같은
  파일에 둔다. 코어(view/state/model)는 `features/`를 참조하지
  않는다 — 공용 도구가 필요하면 commands.ts로 올린다.
- 커스텀 타이핑 동작은 `Extension.input` 훅으로 구현한다 —
  input.ts를 고치지 않는다. 컨텍스트는 훅마다 다르다:
  `insertText`/`insertParagraph`/`deleteBackward`는 `BlockTextCtx`
  (한 블록 안 캐럿/범위의 플랫 텍스트 좌표 뷰), `insertFromPaste`는
  `RangeCtx`(정규화 범위 + 클립보드 html/text), `files`는
  `FileDropCtx`(에디터 파사드 + File 목록 + 드롭 좌표)다.
  `paintOverlay`는 렌더 커밋 후 DOM 마킹만 한다(doc 불변).
- `features/index.ts`의 `starterFeatures()`에 등록한다 — **배열
  순서가 계약**이다. 같은 키를 바인딩하는 기존 기능과의 우선순위
  (keymap 폴스루)와 입력 규칙·input 훅 순서를 보고 위치를 정한다.
- 스냅샷/옵션 필드가 필요하면 `types.ts`/`core.ts`에도 추가한다 —
  setter는 동일 값에 멱등, 스냅샷은 변경 시에만 새 참조.

## 2. 유닛 테스트

`features/<name>.test.ts`에 케이스를 추가한다 — `features/testkit.ts`의
`editor`/`para`/`type`/`run` 헬퍼로 실제 입력 경로(insertText →
입력 규칙)와 커맨드를, `textCtx`/`rangeCtx`/`fileDropCtx`로
`Extension.input` 훅을 DOM 없이 직접 검증한다.
상태 전이, 이벤트 발행, 멱등성, 예외 폴백을 본다.

## 3. 5개 렌더러

- `mount.ts` — DOM 계약에 새 요소/속성 추가 (`me-` 접두어)
- React `EditorView` / Vue3·Vue2 `EditorView.vue` / Svelte `EditorView.svelte` —
  같은 DOM 계약으로 렌더 + prop/이벤트 배선

## 4. 컨포먼스 계약

DOM으로 검증 가능하면 `conformance.ts`에 it 블록을 추가한다 —
선택자 + 상호작용 + 기대 DOM. 5개 어댑터 테스트가 같은 스펙을 실행하므로
한쪽 구현이 빠지면 즉시 실패한다.

## 5. 데모

`apps/dev-*/src/data.ts`의 `FEATURES`에 항목을 추가하고 실제로 옵션을
켠다. 새 옵션을 실제로 쓰는 화면이 없으면 E2E가 검증할 대상이 없다.

## 6. E2E

`e2e/`에 스펙을 추가한다 — 역할이 크면 파일을 나눈다(features/…).
`helpers.ts`의 `apps` 루프로 작성해 5개 렌더러를 동시에 검증한다.

## 7. 문서

- `docs/guide/features/<feature>.md` — 기능 가이드
- 플랫폼별 prop/슬롯이 다르면 `platforms/*.md` 갱신
- `migration.md` — 유사 라이브러리에서 온 사용자를 위한 매핑 표에 추가

## 8. 검증

```bash
pnpm test && pnpm typecheck && pnpm build && pnpm build:apps && pnpm e2e
```
