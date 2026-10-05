# 테스트

3층 구조 — 각 층이 다른 회귀를 잡는다.

## 1. 유닛 테스트 (vitest)

`packages/core/src/*.test.ts` — 코어 로직만 검증. DOM 불필요.
어댑터에도 바인딩 테스트를 둘 수 있다.

## 2. 컨포먼스 (5렌더러 공용 계약)

`packages/core/src/conformance.ts`의 `runWidgetConformance(label, mount)`는
DOM 계약 스펙을 선언한다. 각 어댑터의 `conformance.test.*`가 자기 마운트
함수를 주입해 실행한다:

```ts
runWidgetConformance("react", async (opts) => {
  const el = document.createElement("div");
  const widget = new WidgetCore(opts);
  const root = createRoot(el);
  await act(() => root.render(<WidgetView widget={widget} />));
  return { el, flush: () => act(async () => {}), destroy: () => root.unmount() };
});
```

`mount`는 `{ el, flush?, destroy }`를 반환한다 — `flush`는 DOM 반영 대기
(vue: `nextTick`, svelte: `tick`, react: `act`, vanilla: 생략).

## 3. E2E (Playwright)

`e2e/*.spec.ts` — 실제 브라우저에서 5개 데모 앱을 순회한다.
`helpers.ts`의 `apps` 목록이 포트 매핑(5173~5177)을 들고 있고,
`webServer`가 dev 서버를 자동 기동한다.

```ts
for (const app of apps) {
  test.describe(app.name, () => {
    test("...", async ({ page }) => {
      await gotoApp(page, app.name);
      // ...
    });
  });
}
```

## 명령

```bash
pnpm test         # vitest 전체 (유닛 + 컨포먼스)
pnpm e2e          # Playwright
pnpm e2e:install  # 최초 chromium 설치
```
