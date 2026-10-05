# Code review: `frontend`

I split the findings into three categories:  high, medium and low. The high category contains issues that are likely to cause a failure in production, or a security vulnerability. Medium issues are less likely to cause a failure, but they are still worth fixing. Low issues are mostly about clarity and style.


## High

| # | Location | Principle | Finding |
|---|---|---|---|
| 1 | `App.test.js`, `Spinner.jsx` | Proper function, tests | **The only App test fails** with `Invariant Violation: Target container is not a DOM element`. `Spinner` portals into `#modal-root`, which only exists in `public/index.html`, not in the test DOM. `fetch` isn't mocked either. Fix: create `modal-root` in the test setup (or have `Spinner` fall back to `document.body`), and mock `fetch`. |
| 2 | `App.js` `fetchData` | Proper function, error logging | There's no error handling. A network failure, a non-2xx status, or an HTML error page from TomEE makes `json()` throw. `setLoading(false)` then never runs, so **the spinner spins forever**, and the promise rejection goes unhandled. Check `res.ok`, use `try/catch/finally`, log the error, and show a message to the user. |
| 3 | `App.js` render | Performance, proper function | `data.sort(sortFunc)` **sorts the state array in place** on every render. That means 5,000 rows are re-sorted each time the page, the loading flag, or the sort changes. Use `useMemo(() => [...data].sort(multiSort(sort.split(','))), [data, sort])`. |
| 4 | `App.js` overall | Scalability, "do it in the DB" | The app downloads every row, then sorts and pages in the browser. It already shows only 20 rows at a time. Fetch `?page=&sort=` from the server and let PostgreSQL do the `ORDER BY`/`LIMIT` (see `pr_comments_db.md`). |
| 5 | `Row.jsx`, `ControlBar.jsx`, cross-layer | Proper function | `participantfullname` isn't returned by `get_awarded_transactions`. As a result the participant tooltip is always empty, and the default sorts `contractsize,participantfullname` and `tradetype,participantfullname` have no working tie-breaker. |

## Medium

| # | Location | Principle | Finding |
|---|---|---|---|
| 6 | `App.js` `List` | No warnings | Rows have no `key` prop, so React logs a warning and can't match rows between renders. `index` is unused. Rows have no stable ID, because `trans` has no primary key; add one in the database and use it as the key. |
| 7 | `ControlBar.jsx` page input | Proper function | `min`/`max` only limit the spinner arrows. Typing `0`, `-3`, `999`, or clearing the field sets an invalid page and shows an empty table, because `Number('')` is `0`. Also, with no data `pages` is 0, so "Last Page" sets `page = 0`. Clamp the value to `[1, max(pages, 1)]`. |
| 8 | `FormatUtil.js` | Proper function | `formatCurrency(null)` shows `$0.00`, so a missing value looks like a real zero. `formatCurrency(undefined)` shows `$NaN`, for example on the extra `{}` row from Java finding #1. Return an empty string or `—` for empty values. |
| 9 | `FormatUtil.js` and its test | Tests, clarity | The locale is `undefined`, so output depends on the host machine's locale, and the test expects `$123.40`, which fails on a `de-DE` machine. Use `'en-US'` explicitly and the usual uppercase `'USD'`. |
| 10 | `Row.jsx` / `App.css` | Proper function, clarity | Every row reuses the same `id`s (`participant`, `cost`, …), giving about 200 duplicate IDs per page, which is invalid HTML. CSS styles these by `#id`, so switch to class names. |

## Low

| # | Location | Principle | Finding |
|---|---|---|---|
| 11 | `Spinner.jsx` | Clarity | When not spinning it renders an empty `<div />`; return `null` instead. The progress indicator has no `aria-label`, and the table stays clickable while loading. |
| 12 | `Row.jsx` `Header` | Clarity | `Header` declares a `{ item }` prop it never uses, and the `id="header"` passed from `App` is ignored. |
| 13 | `Row.jsx` | Accessibility | The table is built from `div`s with no `table`/`role="table"`/`columnheader` markup, so screen readers can't read it as a table. |
| 14 | `ControlBar.jsx` | Accessibility | The page input has only a `title`, with no label or `aria-label`. |
| 15 | `App.js` | Proper function | Changing the sort keeps the current page, for example page 50 of a newly sorted list. Usually you'd reset to page 1. Sorting is ascending only; `reverse` exists but isn't used. Should Profit or Revenue sort highest first? |

