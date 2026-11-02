# 멘션 (mentionFeature)

인라인 atom 노드 — `@이름` 칩을 텍스트 흐름 안에 끼운다.
`starterFeatures()`에는 없는 à-la-carte 피처 — 각 앱이
`mentionFeature`를 `extensions`에 직접 추가한다 (4개 어댑터와
데모 5앱에 재수출됨).

```ts
editor.commands["insertMention"]!("parks");
// 캐럿 자리에 mention + 공백을 삽입하고 캐럿은 멘션 뒤로
```

## 동작 모델

- `type: "mention"` — `inline: true` + `atom: true`. 텍스트를 갖지
  않는 칩이라 `Backspace`는 통째로 지우고, 커서는 건너뛴다.
- `insertMention`은 `insertSlice`로 `[mention, " "]`를 한 조각으로
  넣는다 — 리프 끝 삽입 시 공백이 멘션 **뒤**에 오도록 오프셋을
  직접 계산하지 않는다.
- 칩의 라벨은 `attrs.label`이 `data-label`로 나가 `::after`가
  출력한다 — 텍스트 노드가 아니라 오프셋 매핑에 영향이 없다.

## DOM 계약

```text
span.me-mention[data-me-path][contenteditable="false"][data-id][data-label]
```

- `ce=false` — 브라우저가 칩 안으로 캐럿을 못 넣게 한다.
- 라벨은 `::after { content: "@" attr(data-label) }` — CSS가 렌더한다.
- DOM↔Point 매핑에서 칩은 하나의 리프 — `domToPoint`/`pointToDOM`
  모두 atom으로 취급한다.
