# Engram app: notes for editing

Single-file app (`index.html`), served by GitHub Pages from `main`.

## Versioning
- One version string: `const APP = { name: 'Engram', version: 'x.y.z' }`. Bump the last part on every shipped change and end the commit message with `(x.y.z)`.

## Study time: nothing on screen is dropped (user rule)
- The user's rule: every moment a card is on screen counts as time on cards, whether it was graded, buried, suspended, skipped, undone or still open at X. "Time on cards" must add up to the session clock (except time spent waiting for learning cards, when no card is shown).
- **Card time** (`rt` on each revlog row): from when the *previous* card was left (`pg.cardFrom`: the last grade, a leave, or the session start) to the grade, minus pauses. It includes the gap and render before the card. Anki's own `time` field still runs from `shownAt` and is capped.
- **Ungraded time**: `Session.leave(pg, next)` runs in `showCard` (unless it's the same card re-rendered), `waitForLearning`, `finishStudy` and `Session.end`. It pushes `[cid, ms, at]` into the session's `u`. An undone answer's time goes into `u` too. `u` is stored on the `SessionLog` row, and `Views.list` / `Views.of(cid)` read it back.
- **Every time total must add Views**: `todayStudy`, the deck-overview chart, card info "Total time", `Insights.all`, `Insights.range` (days, prev, hours, decks, cards) and the day sheet. Per-grade and per-type paces (`kt`, `gt`, `tt`, `pt`) and the Anki-style view (`r.time`) stay answer-only. Any new time total must include `Views`.
- A day with ungraded time but no answers gets a `days` entry with `n: 0`. Streaks and active days count only days with `n > 0`.
- A session row with no answers but some `u` is kept, and it is listed as a 0-card session.

## Study time: three different measures
- **Card time** (`rt` on each revlog row): see above. Summed, with Views, as "time on cards" in Stats.
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
- To check the whole accounting: grade a card, bury one (`studyAction(pg, 'bury-card')`), pause and resume (`Session.pause` / `Session.resume`), `doUndo(pg)`, then leave a card open and press X. Then `todayStudy().t`, `Insights.all('all').days.get(0).t`, `Insights.range(7).t` and the session's `active` must all be equal.
