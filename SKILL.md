---
name: render-ui-test
description: Render project UI to PNG from a TEST — not a script, main(), or one-off exec method — with least code, by mocking data at the LOWEST seam so every real project code path runs. Builds a reusable render-glue helper for future tests. Use when asked to render or screenshot a component/page/view to an image, set up snapshot/visual tests, or "see what the UI looks like" without launching the app.
---

# render-ui-test

Goal: one PNG of **real** UI, from a **test**. Least code. Real code path. Glue -> reusable helper.

Deliverable is a test in the project's test framework — not a script, `main()`, or one-off exec method. A script renders once and is thrown away; a test renders on every run and its content asserts **guard** the behavior forever (rule 4).

## Rules

1. **Mock at bottom, not UI.** Fake lowest data seam (data source, transport, store, clock/env). Everything above = real project code. Never stub UI components -> defeats point.
2. **Lowest *already-injectable* seam.** Pick the deepest point the code already lets you swap. Refactoring to create a seam, or faking a wire format (raw HTTP bytes) when a repository/client interface sits just above it = not least code. Go one level up instead.
3. **Use the platform's renderer, don't build one.** Rasterizing is solved per platform (table below). Helper wraps that tool; it never hand-rolls headless browsers or bitmap encoding.
4. **One helper does glue, data is a parameter.** `RenderToImage(name, data, buildSubject) -> (path, view)`. Lives in test-support. Tests pass their data in — the helper must not hardcode it, or overrides silently do nothing.
5. **Assert content, not pixels.** Every test asserts the key thing that *should* render — the must-show data/text/node count the real logic produces (e.g. user name shows, list has N rows, error banner present). Query the rendered view/tree/markup, not the PNG. This proves mock data flowed through real code into the UI. Eyeball once; content asserts guard forever.
6. **Deterministic or it lies.** Before capture: wait until data/fonts/images are loaded; freeze clock, timezone, locale, random seeds; disable animations/transitions; neutralize lifecycle side effects (`onAppear`, `useEffect` fetches) through the same data seam. A PNG that differs per run is noise.

## Renderer by platform

Pick what the project already uses; else the first fit. `jsdom`/`happy-dom` do **no layout** -> fine for content asserts, cannot produce a PNG.

| Stack | Offscreen render -> PNG |
|---|---|
| Web (React/Vue/Svelte/Lit) | Vitest browser mode (`page.screenshot`, `toMatchScreenshot`) or Playwright component testing; Storybook story as subject if the project has stories |
| Server-rendered HTML (Go, Rails, etc.) | Render the real handler to an HTML string, load it in headless Chromium (Playwright / chromedp), screenshot |
| SwiftUI / UIKit | `ImageRenderer`, or pointfreeco `swift-snapshot-testing` (`.image`) |
| Android (Compose / Views) | Roborazzi (Robolectric, supports interaction) or Paparazzi (LayoutLib, fastest, static) |
| Flutter | `testWidgets` + `matchesGoldenFile`; load real fonts first or text renders as Ahem boxes |

## Workflow

1. **Find render entry** — topmost fn app actually calls to make UI for page/component (not leaf widget). Trace route/handler/screen down to view it returns.
2. **Find lowest injectable data seam** (rule 2) — where external data enters.
3. **Pick renderer** from the table; check it runs headless in this repo before writing more.
4. **Write helper** (template). Takes data, builds real subject, renders, waits (rule 6), writes PNG.
5. **Write test** — one per meaningful state: populated, empty, loading, error. Each passes its own data, calls helper, asserts content. This is the deliverable.
6. **Run the test.** No app launch, no server.
7. **Eyeball** — open every PNG with the Read tool and look at it; "file exists" is not verification. Blank/wrong -> seam too high, entry wrong, or not waited -> back to 1–3. Then bake the must-show content into asserts before done.

**Stop and ask** instead of forcing it when: the entry can only render inside a running app host, there is no test framework, or the renderer needs a large new dependency the project doesn't have.

## Helper template (pseudocode)

```
# test-support: reusable glue
RenderToImage(name, data, buildSubject):
    deps    = Deps{ source: FakeSource(data), clock: FixedClock(...) }  # mock at lowest seam
    subject = buildSubject(deps)          # real project code runs above
    view    = render(subject)             # real render entry
    waitUntilSettled(view)                # data loaded, fonts ready, no animation
    png     = rasterize(view)             # platform renderer from the table
    path    = outDir() + "/" + name + ".png"
    write(path, png)
    return path, view                     # path to eyeball, view for content asserts

# the test = deliverable
Test_OrdersPage_Populated:
    data = { user: "Alice", orders: 3 canned orders }
    path, view = RenderToImage("orders-populated", data, buildOrdersPage)

    assert view.findText("Alice")              # mock data reached UI
    assert view.count(".order-row") == 3       # real list logic ran
    assert view.has("#welcome-banner")         # key node present

Test_OrdersPage_Empty:
    path, view = RenderToImage("orders-empty", { user: "Alice", orders: [] }, buildOrdersPage)
    assert view.findText("No orders yet")
```

New test = pick subject + pass data -> call `RenderToImage` + assert key content. Done.

Query API is whatever the project gives: testing-library `getByText`, HTML-string `contains`, render-tree walk, view hierarchy.

**Output dir:** follow the renderer's convention if it has one (e.g. Vitest `__screenshots__/` beside the test); else one configurable dir under the test framework's artifacts/temp path, unique name per test so parallel runs don't collide, and git-ignored unless the images are committed baselines.

## Keep small

- One fake, hardcoded data. No mock libs unless the seam needs them.
- Render entry needs many deps = smell. Inject one data seam, default the rest.
- Pixel/snapshot diff is optional — add it later; content asserts first. When added, compare only on one pinned environment (same OS/browser/Xcode as CI) with a small tolerance; rendering differs across machines.
