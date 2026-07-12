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
**What I did:**
**How I verified:**

## Comment 3 — Missing test
**What I did:**
**How I verified:**

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