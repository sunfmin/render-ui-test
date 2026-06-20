---
name: render-ui-snapshot
description: Render project UI to PNG with least code, by mocking UI data at LOWEST layer so every real project code path runs. Builds reusable render-glue helper for future tests. Use when asked to render UI to image, screenshot a component/page/view, set up snapshot/visual tests, or "see what the UI looks like" without running the full app.
---

# render-ui-snapshot

Goal: one PNG of **real** UI. Least code. Real code path. Glue -> reusable helper.

## 3 rules

1. **Mock at bottom, not UI.** Fake lowest data seam (data source, transport, store, clock/env). Everything above = real project code. Never stub UI components -> defeats point.
2. **One helper does glue.** `RenderToImage(buildSubject) -> png`. Build deps -> call real render entry -> encode PNG -> write file. Lives in test-support.
3. **Reuse.** Future tests call helper + swap mock data. No copy-paste.

## Workflow

1. **Find render entry** — topmost fn app actually calls to make UI for page/component (not leaf widget). Trace route/handler/screen down to view it returns.
2. **Find lowest data seam** — where external data enters. Pick deepest injectable -> max real code above.
3. **Write helper** (see template). One mock impl -> canned data. One render call. One file write.
4. **Render** — turn view into pixels via project's native offscreen/headless path. No app launch, no server.
5. **Eyeball** — write `out/<name>.png`, open, confirm real component + mock data showing. Blank/wrong -> seam too high or entry wrong -> back to 1–2.

## Helper template (pseudocode)

```
RenderToImage(name, buildSubject):
    deps    = Deps{ data: fakeData() }   # mock injected at lowest seam
    subject = buildSubject(deps)         # real project code runs above
    view    = render(subject)            # real render entry
    png     = rasterize(view)            # offscreen/headless -> bytes
    write("out/" + name + ".png", png)
    return path
```

Test = pick subject + override `fakeData()` -> call `RenderToImage`. Done.

## Keep small

- No test framework, no assertions first. Make image. Add diff/assert later.
- One mock struct, hardcoded data. No mock libs unless seam needs.
- Render entry needs many deps = smell. Inject one data seam, default rest.
