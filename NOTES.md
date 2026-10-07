# NOTES

## Summary of changes
1. **SQL precedence bug (highest value)**: `AND`/`OR` was unparenthesised in `TaskRepository`, `db/queries/search_tasks.sql` and both queries in the Oracle package. Archived tasks leaked into results and the status filter only applied to title matches. Fixed with parentheses; also added `id DESC` as a pagination tie-breaker.
2. **Artificial latency**: removed `Thread.sleep` from `TaskController` (short or empty searches were delayed up to ~1s).
3. **Input handling**: invalid `status` now returns 400 instead of 500; `page`/`pageSize` are clamped (page >= 1, size 1-100) and index math uses `long`, so `page=0` or huge values no longer throw.
4. **Frontend**: `useTasks` aborts stale requests (race condition), clears errors, and resets `loading` on failure (was stuck on "Loading"). Added 300 ms search debounce and reset to page 1 when search/filter changes.
5. Oracle: widened `v_term` from 257 to 4000 chars to avoid ORA-06502 on long terms.

## Deliberately not changed
- Pagination is still in memory (`subList`); moving it to the DB (`Pageable` / `OFFSET FETCH`) is a bigger diff than a patch should be.
- LIKE wildcards (`%`, `_`) typed by the user aren't escaped.
- `@CrossOrigin` is hardcoded; no tests added.

## Biggest remaining risk
In-memory pagination: every request loads all matching rows. Fine for 50 rows, a problem at tens of thousands. Next would be unbounded `LIKE '%x%'` scans with no index.

## Tools / AI
Used Claude to review the code and draft the fixes. I checked the SQL fix against the seed data (old query with `q=api&status=DONE` returned 5 rows incl. 2 archived; fixed query returns 0), confirmed the frontend builds, and compiled the controller. I overrode the first instinct to rewrite pagination in SQL to keep the diff small.

## Assumptions
No bug reports were available, so I prioritised by user-visible impact.
