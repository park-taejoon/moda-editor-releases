# Markdown (markdownFeature + presets)

Markdown 직렬화·파싱 — PM `prosemirror-markdown`·Tiptap markdown
extension 대응. 두 방향 모두 DOM 무관 순수 함수이고, 붙여넣기
자동 감지는 `markdownFeature`가 담당한다 (`starterFeatures()` 포함).

## 직렬화 — `serializeToMarkdown`

```ts
import { serializeToMarkdown } from "@moda-editor/core";

serializeToMarkdown(editor.state.doc); // 문서 전체 → md 문자열
serializeToMarkdown(doc, {
  marks: { myMark: (inner, mark) => `<x>${inner}</x>` }, // 마크 오버라이드
  nodes: { myNode: (node, ctx) => `...` }, // 블록 오버라이드
});
```

- 블록: 문단/제목(`#`~`######`)/인용/글머리·순번·작업(`- [ ]`) 목록/
  표/펜스 코드블록/수평선/이미지/hardBreak.
- 마크: 굵게·기울임·취소선·코드·링크. 마크다운이 표현 못 하는 마크
  (밑줄·하이라이트·색·글자크기·위/아래첨자)는 인라인 HTML로 보존한다.
- 알 수 없는 블록은 자식 블록 구조를 유지해 직렬화하고, 인라인
  자식뿐이면 인라인으로 폴백한다 — 어느 쪽이든 텍스트 유실이 없다.
  커스텀 노드를 특정 구문으로 표시하려면 `nodes` 오버라이드를 쓴다
  (데모의 callout은 `> [!NOTE]` GitHub alert로 표시).
- 결과는 줄바꿈 하나로 끝난다 (파일 저장 관례).

## 파싱 — `parseMarkdown`

```ts
import { parseMarkdown } from "@moda-editor/core";

const nodes = parseMarkdown(md); // EditorNode[] — tr.insertSlice 등에 사용
editor.dispatch(editor.state.tr().insertSlice(at, nodes)); // 원하는 곳에 삽입
```

- 직렬화기의 역방향: ATX 제목/펜스 코드블록/인용/중첩 목록/작업
  목록/표/수평선/문단 + 인라인 굵게·기울임·취소선·코드·링크·이미지.
- 백슬래시 이스케이프(`\*` 등), 문장 끝 공백 2개 → `hardBreak` 지원.
- 마크다운이 아닌 구문(미종결 구분자 포함)은 텍스트로 그대로 보존 —
  파싱 실패로 텍스트를 잃지 않는다.
- 왕복 계약: `serializeToMarkdown` 출력을 `parseMarkdown`에 넣으면
  구조가 복원된다 — `markdown-parse.test.ts`가 대표 구문을 검증한다.

## 붙여넣기 — `markdownFeature`

`insertFromPaste` 훅의 우선순위:

1. 클립보드에 `text/html`이 있으면 **HTML 경로가 항상 우선**한다 —
   웹페이지에서 복사한 마크다운 소스가 HTML과 함께 올 때 오인 방지.
2. `text/plain`에 마크다운 신호가 있으면(제목/목록/인용/펜스/표/
   링크/이미지/수평선/마크 구분자) `parseMarkdown`으로 파싱해
   `insertSlice`로 삽입한다.
3. 위 둘 다 아니면 평문 경로로 넘긴다 — 일반 텍스트 붙여넣기의
   동작 변화는 없다.

코드블록 안에서는 파싱을 건너뛰어 코드블록 피처의 평문 붙여넣기
정책(`x<div>`가 태그가 아닌 문자로 들어가는 계약)이 그대로 적용된다.

## 제약

- 밑줄·하이라이트·색 등 HTML 폴백으로 나간 마크는 파서가 복원하지
  않는다 — 인라인 HTML 파싱은 `parseHTML`의 몫이며 왕복 대상 밖이다.
- 마크다운 입력 규칙(`## ` 타이핑 → 제목 변환)은 별도 — 각 기능
  모듈(`heading.ts`, `marks.ts`, `lists.ts` 등)의 `inputRules`가
  담당하고 `starterExtensions()`가 조립한다.
