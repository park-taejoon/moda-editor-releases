# 코드블록 (codeBlockFeature / aceFeature)

Ace 에디터급 코드 편집 — 구문 하이라이트, 줄 단위 편집, 자동 쌍.
`starterFeatures()`에 포함된다.

```ts
editor.commands["toggleCodeBlock"]!(); // 현재 블록 ↔ 코드블록
editor.commands["setCodeLanguage"]!("ts"); // 현재 블록 언어
editor.commands["setCodeLanguageAt"]!([2], "python"); // 경로 지정
editor.commands["formatCode"]!(); // 들여쓰기 정리 (VS Code 관례)
editor.commands["formatCodeAt"]!([2]); // 경로 지정
editor.commands["toggleCodeComment"]!(); // Mod-/ — 언어별 주석 토글
editor.commands["moveCodeLines"]!("up" | "down"); // Alt-↑/↓ 줄 이동
editor.commands["duplicateCodeLines"]!(); // Mod-Shift-D 줄 복제
```

## 키맵

| 키                  | 동작                                                           |
| ------------------- | -------------------------------------------------------------- |
| `Tab` / `Shift-Tab` | 줄 들여쓰기/내어쓰기 (여러 줄 선택 가능)                       |
| `Enter`             | 자동 들여쓰기 — `{` 뒤는 한 단계 더, 공백 꼬리 뒤 Enter는 탈출 |
| `ArrowUp/Down`      | 가장자리 줄에서 블록 밖으로 탈출 (Tiptap 관례)                 |
| `Mod-/`             | 언어별 주석 토글 (`//`, `#`, `/* */`)                          |
| `Alt-↑/↓`           | 줄 이동                                                        |
| `Alt-Shift-F`       | 서식 정리                                                      |
| `Mod-Shift-D`       | 줄 복제                                                        |

## 동작 모델

- 언어는 `language` attr — `presets/highlight.ts`가 토큰 스팬으로
  렌더한다. 미지원 언어는 GENERIC 패턴으로 폴백한다.
- 붙여넣기는 항상 순수 텍스트 — HTML 클립보드도 코드로 들어간다.
- 문자/주석 토큰 안에서는 자동 쌍·타입오버가 리터럴로 눌린다
  (Ace `isCodeLike` 판정).
- 캐럿이 들어가면 `.me-codeui` 크롬 바(언어 선택·정리·복사)와
  줄 번호 거터(`.me-gcode`)가 뜬다 — 둘 다 오버레이라 DOM 매핑을
  침범하지 않는다.

## DOM 계약

```text
pre[data-me-path][data-language] > 텍스트 홀더 > .me-cl > 토큰 span.me-tok
```

- `.me-cl`이 줄 단위 래퍼 — 클릭하면 그 자리 정확한 캐럿으로 간다.
- `pre`는 `white-space: pre`라 `\n`이 줄바꿈이다 — hardBreak 노드가
  아니라 리터럴 개행이다.
