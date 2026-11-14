# Filter the schedule by several tracks at once

## What this changes

On `/schedule` the **Track** chips become toggles. Clicking an unselected track
adds it, clicking a selected one removes it, and the results are the sessions in
**any** selected track. **All** clears the selection; removing the last selected
track does the same. The selection is one URL key, `?track=a,b`, sent to the API
as `trackSlug=a,b`; `GET /api/sessions` splits it into `t.slug IN (?, ?)`. A
single value is the same query as today, so `?track=a` links keep working. Every
track chip, and **All**, carries `aria-pressed`.

## Where

- `server/routes/sessions.js` — the `trackSlug` line in `sessionsRouter.get('/')`:
  split on `,`, drop empties, `t.slug IN (?, …)`; update the doc comment.
  Test in `tests/api/endpoints.test.js`, a new `describe('Session filters')` using
  `startApi` / `client` from `tests/api/harness.js` and `boot.tracks`.
- `src/pages/SchedulePage.jsx` — the Track chip block (`tracks.map`, ~line 163):
  derive `selectedTracks` from `filters.track` (split, `ALL` → `[]`), toggle a
  slug in/out and write it back with `set('track', list.join(','))` (an empty
  list deletes the key via `set`). `active = selectedTracks.includes(t.slug)`;
  add `aria-pressed` to each chip and to **All**. `useFetch` keeps passing
  `filters.track` as `trackSlug` — `qs()` in `src/lib/api.js` passes the string
  through unchanged. `BLOCKERS` and `activeKeys` need no change.
- `tests/schedule.spec.js` — a new test in `describe('Schedule')`, read-only so
  no lane: `visit(page, '/schedule?day=…', { at: await momentOn(0, MID_SESSION_TIME) })`,
  `waitForResults`, `conferenceDays()`, API counts via `request.get(`${API}/sessions?day=…&trackSlug=…`)`.
  On mobile, click `Filters` before the chips (schedule.md Gotchas).
- `context/engineering-context/schedule.md` — the `trackSlug` row of the params
  table becomes `t.slug IN (…)`, comma-separated.

## How it will be proved

| Done when | Check | Layer |
| --- | --- | --- |
| Two tracks show exactly those tracks' sessions; `result-count` matches the API, grid and list, desktop and mobile | API: `trackSlug=a,b` returns the union of `trackSlug=a` and `trackSlug=b` (ids and length), and `trackSlug=a` alone is unchanged. Spec: pick a day where two tracks both have sessions; click chip A then chip B; `result-count` reads the API's `trackSlug=a,b` count; repeated starting from `view=grid` and `view=list`; runs in both Playwright projects | API + spec |
| Deselecting one narrows to the remaining track | Spec, same test: click A again; URL is `track=b`, `result-count` equals the API's `trackSlug=b` count | spec |
| **All**, or deselecting the last track, shows every track | Spec, same test: from `a,b` click **All** → no `track` in the URL and `result-count` equals the API's day count; reselect A, click A again → the same | spec |
| Each chip exposes `aria-pressed` | Spec, same test: A and B `aria-pressed="true"`, a third chip and **All** `"false"`; after deselecting, A `"false"` | spec |
| Reload / shared `?track=a,b` restores it; `?track=a` still works | Spec: `visit` `?track=a,b` → both chips pressed, count matches API; `page.reload()` → same; `visit` `?track=a` → only A pressed, count matches `trackSlug=a` | spec |

## Decisions

1. Two criteria already hold on main for a single track: `?track=a` shows that
   track, and clicking **All** or the active chip returns to all tracks. This
   build proves them starting from a two-track selection (and keeps the `?track=a`
   check as a regression guard), not from one track.
2. A track filter keeps blocking the grid, as today: with `view=grid` requested
   the page falls back to the list and shows the "not available with a track
   filter" notice. "In grid and list view" is read as: the count matches whichever
   view was requested.
3. The **All** chip keeps its current label "All" rather than "All tracks".

## Out of scope

Multi-select for day, venue, level, format or tag; `/speakers`' own `trackSlug`.
