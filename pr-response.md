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
I would set `public` to default to `False` (private-by-default), with 
users able to opt in to making a watchlist entry public.
**Reasoning:**
By defaulting to private, we avoid the risk of users accidentally sharing 
their watchlist entries with the public. This aligns with the principle of 
"safe by default," ensuring that users who are not actively thinking about 
privacy settings do not inadvertently expose their viewing habits. Right 
now, CineLog doesn't have a mechanism that would allow anyone to see a 
public watchlist entry, so defaulting to public today doesn't deliver on 
the "community" value the app describes yet — the default currently has no 
functional effect either way, so there's no cost to choosing the safer 
option now, and it's easy to flip once real community features exist.
**Tradeoff acknowledged:**
The cost of private-by-default is that users have to remember to opt in if 
they want to share their watchlist entries publicly. This introduces a small
amount of friction for users who want to share, but it is a conscious action
users must take, which is preferable to the risk of unintentional public 
sharing. I think this tradeoff is acceptable because the consequences of 
forgetting to opt in (the entry remains private) are less severe than the 
consequences of being opted in without realizing it (the entry is public).

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