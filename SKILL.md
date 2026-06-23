---
name: render-ui-test
description: Render project UI to PNG from a TEST — not a script, main(), or one-off exec method — with least code, by mocking data at the LOWEST seam so every real project code path runs. Builds a reusable render-glue helper for future tests. Use when asked to render or screenshot a component/page/view to an image, set up snapshot/visual tests, or "see what the UI looks like" without launching the app.
---

# render-ui-test

Goal: one PNG of **real** UI, from a **test**. Least code. Real code path. Glue -> reusable helper.

Deliverable is a test in the project's test framework — not a script, `main()`, or one-off exec method. A script renders once and is thrown away; a test renders on every run and its content asserts **guard** the behavior forever (rule 4).

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
6. **Eyeball + assert (rule 4)** — test writes `out/<name>.png`; open it, confirm real component + mock data show. Blank/wrong -> seam too high or entry wrong -> back to 1–2. Then bake the must-show content into asserts before done.

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

Query API is whatever the project gives: testing-library `getByText`, HTML-string `contains`, render-tree walk, view snapshot.

## Keep small

- One mock struct, hardcoded data. No mock libs unless the seam needs them.
- Render entry needs many deps = smell. Inject one data seam, default the rest.
- Pixel/snapshot diff is optional — add it later; content asserts first.
