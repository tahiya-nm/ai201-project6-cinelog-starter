# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun`
naming convention used by `add_to_collection()`. Updated the import and
call site in `routes/watchlist/watchlist.py`'s `add_film()` route.

**How I verified:** Ran `grep -rn "save_to_watchlist" --exclude-dir=.git .`
before the rename to find all three references (the function definition
and the two usages in the route file), made the changes, then re-ran the
same grep and confirmed zero matches remained. Also ran `pytest tests/ -v`
to confirm the app still imports and runs cleanly with no breakage.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()` in
`services/watchlist_service.py`. After the existing film-existence check,
I query for an existing `WatchlistEntry` matching `user_id` + `film_id`.
If one is found, I raise a new `AlreadyInWatchlistError` (defined in the
same file, following the same one-line `Exception` subclass style as
`FilmNotFoundError` and `AlreadyInCollectionError` in
`collection_service.py`) instead of silently inserting a duplicate row.

**How I verified:** Modeled this directly on `add_to_collection()` in
`services/collection_service.py`, which does the same query-then-raise
check (query by `user_id`+`film_id`, raise a custom error if found) rather
than relying on a DB-level constraint to throw. Ran `pytest tests/ -v` to
confirm no regressions in the existing collection tests (a dedicated
watchlist test for this case comes next, in Comment 3).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with
`test_add_to_watchlist_nonexistent_film_raises`, modeled directly on
`test_add_to_collection_nonexistent_film_raises` in `test_collection.py`.
Since `test_collection.py`'s `app` and `sample_user` fixtures aren't in a
shared `conftest.py`, I redefined the same fixtures locally in the new
test file, matching the in-memory SQLite setup pattern exactly.

**How I verified:** Used a fake `film_id` of `999999` (an integer, since
`Film.id` is still an `Integer` column on this branch pre-rebase — unlike
`test_collection.py`'s UUID-string fake ID). Ran
`pytest tests/test_watchlist.py -v` to confirm it passes, then
`pytest tests/ -v` to confirm the full suite (5 tests) passes with no
regressions.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end -->