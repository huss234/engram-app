# Engram app: notes for editing

Single-file app (`index.html`), served by GitHub Pages from `main`.

## Versioning
- One version string: `const APP = { name: 'Engram', version: 'x.y.z' }`. Bump the last part on every shipped change and end the commit message with `(x.y.z)`.

## Study time: three different measures
- **Card time** (`rt` on each revlog row): from card shown to grade. Summed as "time on cards" in Stats. Excludes the gaps between cards and any card left unanswered.
- **Session clock** (`Session.active(pg)`): wall clock from session start minus self-paused time. It's what the top-bar chip shows.
  - `r.sa` on a revlog row = the clock *at that answer*.
  - The `SessionLog` row (`engramSess_*` conf keys, synced) holds the clock at close.
- Lesson (2.1.32): Stats used the largest `r.sa` as a session's length, so everything after the last grade was dropped: the last card you were looking at when you pressed X. A session's clock is `max(r.sa, SessionLog row.active)`, matched by `sid === row.id`. Its end is likewise `max(last answer, row.end)`. Apply both in `Insights.sessions` and the day sheet.
- `Session.save(pg)` writes the row with the current clock. It runs on close (`Session.end`) and whenever the page is hidden, because a mobile app killed in the background never reaches close. Any new session view must read the row, not only `r.sa`.

## Testing timing
- Use Playwright's `page.clock.install()` / `clock.runFor(ms)` to fake minutes of study instantly.
  1. Create a deck with `Col.ensureDeck(name).id` and `Col.newNote` + `Col.addNote`.
  2. Close onboarding pages: `while (App.pages.length) closePage(App.pages.at(-1))`.
  3. Run `startStudy(did)`, then press Space / `3` and click `.page.study [data-act="back"]`.
  4. Compare `Insights.sessions(Insights.all('all'), 0)` with the elapsed fake time.
