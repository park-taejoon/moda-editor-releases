# moda-editor

프레임워크 비의존 헤드리스 웹 에디터 — 불변 JSON AST 문서 상태,
트랜잭션 기반 업데이트, 5개 렌더러(React / Vue 3 / Vue 2 / Svelte /
Vanilla)가 같은 DOM 계약을 공유한다.

## 문서

- [가이드](./guide/README.md) — 플랫폼별 사용법, 기능 가이드, 테마, 마이그레이션

## 패키지

| 패키지 | 설명 |
| --- | --- |
| `@moda-editor/core` | 헤드리스 코어 — 문서 모델/트랜잭션/입력 바인딩 |
| `@moda-editor/react` | React 어댑터 |
| `@moda-editor/vue` | Vue 3 어댑터 |
| `@moda-editor/vue2` | Vue 2.7 어댑터 |
| `@moda-editor/svelte` | Svelte 어댑터 |

이 레포의 문서는 `moda-editor` 레포 `docs/`에서 자동 동기화된다 —
`docs/` 변경이 main에 push되면 GitHub Action(`deploy-docs.yml`)이
이 레포로 배포한다. 별도 배포 작업은 없다.
