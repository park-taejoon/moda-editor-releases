# 시작하기 — 템플릿에서 새 프로젝트 만들기

## 1. 복사

```bash
cp -R moda-editor my-lib
cd my-lib
rm -rf .git && git init   # 새 이력으로 시작
```

## 2. 이름 치환

| 치환                           | 대상           | 위치                                             |
| ------------------------------ | -------------- | ------------------------------------------------ |
| `@moda-editor`                    | `@my-lib`      | package.json name/deps, import 문                |
| `moda-editor`                     | `my-lib`       | README, 워크플로 주석                            |
| `me-`                          | `xy-` 등       | DOM 클래스 접두어 (styles.css, conformance, e2e) |
| `--me-`                       | `--xy-` 등     | CSS 변수 접두어                                  |
| `WidgetCore`/`mountWidget`/`WidgetView` | 도메인 이름    | 코어 클래스·마운트 함수·컴포넌트                 |
| `dev-*`                        | 유지 또는 변경 | 데모 앱 이름 (포트 5173~5177)                    |

치환 후에는 예시 코어(검색 가능한 목록)를 자기 도메인 로직으로 교체한다.
`types.ts`의 옵션/스냅샷 → `core.ts`의 상태 로직 → `mount.ts`의 DOM 계약
순으로 바꾸면 어댑터는 바인딩만 수정하면 된다.

## 3. 설치·검증

```bash
pnpm install
pnpm test && pnpm typecheck && pnpm build && pnpm build:apps
pnpm e2e:install && pnpm e2e
```

## 4. 데모 확인

```bash
pnpm dev    # 5개 앱 + http://localhost:50890 탭 셸
```
