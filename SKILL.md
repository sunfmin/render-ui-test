---
name: render-ui-snapshot
description: Render project UI to PNG from a TEST (not a script or one-off exec method) with least code, by mocking UI data at LOWEST layer so every real project code path runs. Builds reusable render-glue helper for future tests. Use when asked to render UI to image, screenshot a component/page/view, set up snapshot/visual tests, or "see what the UI looks like" without running the full app.
---

# render-ui-snapshot

Goal: one PNG of **real** UI, from a **test**. Least code. Real code path. Glue -> reusable helper.

Deliverable is a test in project's test framework — not a standalone script, not a `main()`, not a one-off exec method. Render runs when test runs.

## Rules

1. **Mock at bottom, not UI.** Fake lowest data seam (data source, transport, store, clock/env). Everything above = real project code. Never stub UI components -> defeats point.
2. **One helper does glue.** `RenderToImage(buildSubject) -> png`. Build deps -> call real render entry -> encode PNG -> write file. Lives in test-support, called from test.
3. **Reuse.** Future tests call helper + swap mock data. No copy-paste.
4. **Assert content, not pixels.** Every test asserts the key thing that *should* render — the must-show data/text/node count the real logic produces (e.g. user name shows, list has N rows, error banner present). Query the rendered view/tree/markup, not the PNG. This proves mock data flowed through real code into the UI. Eyeball once; content asserts guard forever.

## Workflow

1. **Find render entry** — topmost fn app actually calls to make UI for page/component (not leaf widget). Trace route/handler/screen down to view it returns.
2. **Find lowest data seam** — where external data enters. Pick deepest injectable -> max real code above.
3. **Write helper** (see template). One mock impl -> canned data. One render call. One file write.
4. **Write test** — test fn picks subject, overrides mock data, calls helper. This is the deliverable.
5. **Render** — run the test. Project's native offscreen/headless path turns view into pixels. No app launch, no server.
6. **Eyeball + assert content** — test writes `out/<name>.png`, open, confirm real component + mock data showing. Then bake the must-show content into asserts on the view/tree (mock name visible, right row count, key node present). Blank/wrong -> seam too high or entry wrong -> back to 1–2.

## Helper template (pseudocode)

```
# test-support: reusable glue
RenderToImage(name, buildSubject):
    deps    = Deps{ data: fakeData() }   # mock injected at lowest seam
    subject = buildSubject(deps)         # real project code runs above
    view    = render(subject)            # real render entry
    png     = rasterize(view)            # offscreen/headless -> bytes
    write("out/" + name + ".png", png)
    return path, view                    # path for eyeball, view for content asserts

# the test = deliverable
Test_RendersLoginPage:
    fakeData = canned user "Alice", 3 orders   # override mock at seam
    path, view = RenderToImage("login", buildLoginPage)

    # assert the LOGIC rendered the must-show content (query view/tree/markup, not PNG):
    assert view.findText("Alice")              # mock data reached UI
    assert view.count(".order-row") == 3       # real list logic ran
    assert view.has("#welcome-banner")         # key node present
```

New test = pick subject + override `fakeData()` -> call `RenderToImage` + assert key content. Done.

Query API is whatever project gives: testing-library `getByText`, HTML-string `contains`, render-tree walk, view snapshot. Pick must-show items only — name, row count, key node — not every pixel/field.

## Keep small

- It's a real test, runs in test framework. Render first to see it, then add content asserts before done.
- Assert the logic's must-show output (text, node, count) — not pixels. Pixel/snapshot diff optional, add later.
- One mock struct, hardcoded data. No mock libs unless seam needs.
- Render entry needs many deps = smell. Inject one data seam, default rest.
