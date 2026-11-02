# 이미지 (imageFeature)

인라인이 아닌 블록 atom — 선택·삭제·업로드 연동.
`starterFeatures()`에 포함된다.

```ts
editor.commands["insertImage"]!(src, alt); // URL 생략 시 prompt
editor.commands["updateImage"]!({ src, alt }); // 노드 셀렉션 중 속성 수정
editor.commands["deleteImage"]!(); // 노드 셀렉션 중 삭제 (Backspace와 동일)
```

## 동작 모델

- 이미지는 `type: "image"`의 **블록 atom** — 텍스트를 갖지 않고
  클릭하면 노드 셀렉션이 된다. `Backspace`/`Delete`가 통째로 지운다.
- 노드 셀렉션 중에는 `.me-imgui` 오버레이(편집 바)가 이미지 위에
  뜬다 — 파일 교체·alt 편집·삭제 버튼.
- 파일 붙여넣기/드롭은 `files` 훅이 받는다 — `image/*` 파일을
  골라 업로드 후 `insertImage`로 삽입한다.

## 업로드 연동

`Editor` 생성자에 `onImageUpload?: (file: File) => Promise<string>`
를 넘기면 그 함수의 반환값이 `src`가 된다. 없으면 `FileReader`
data URL로 인라인 삽입한다 — 데모/프로토타입용이라 실제 서비스는
`onImageUpload`를 연결한다.

## DOM 계약

```text
img[data-me-path][src][alt][width?]
```

- `img`가 최외곽이라 `data-me-path`가 여기에 달린다 — `width`는
  `attrs.width` px가 있을 때만 나간다.
- 선택 시 `.is-selected`가 `img`에 붙는다 — `::selection`과 별개의
  노드 셀렉션 채널.
