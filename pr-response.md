# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude throughout this project as a guide and checker, not as a
code generator for the parts that mattered most.

- **Codebase orientation (Milestone 1):** Had Claude read through
  `models.py`, `services/collection_service.py`, and
  `tests/test_collection.py` on `main`, plus the pre-refactor state of
  `models.py`, `watchlist_service.py`, and the watchlist route file on
  `feature/watchlist`, to confirm the exact naming conventions, error
  patterns, and fixture structures before I wrote any code.

- **Reviewing my own code before committing:** For each of the six
  comments, I wrote the fix myself first, then had Claude check it
  against the `add_to_collection()` / `remove_from_collection()`
  patterns before I committed — this caught a couple of real issues,
  including a stale docstring after the UUID rebase and, more
  significantly, a genuine bug: `Film` had no `db.relationship` backref
  for `WatchlistEntry`, which meant `get_watchlist()` would have thrown
  an `AttributeError` the first time anyone actually populated and read
  back a watchlist. That surfaced when I wrote my own sort-order test
  (the second-test stretch feature), not from Claude reviewing code in
  the abstract.

- **Stress-testing Comments 4 and 5:** For the default-visibility
  decision, my first instinct was "public makes sense, it's a community
  app." Claude pushed back by asking what mechanism in the actual
  codebase would let anyone see a public watchlist entry today — there
  isn't one (no feed, no friend list, no browsable profiles) — which
  changed my position to private-by-default, since a public default
  wasn't delivering any real benefit yet while still carrying a real
  privacy cost. For the sort-order comment, I already agreed with the
  maintainer's date-added suggestion, so Claude's main role there was
  making sure my written response actually engaged with the maintainer's
  stated reasoning rather than just restating my own preference in
  parallel.

- **Git mechanics:** Used Claude to walk through the rebase conflict
  resolution (the `.gitignore` and `models.py` conflicts) and the
  interactive rebase for cleaning up commit history, including recovering
  from a bad rebase plan (an early attempt to reorder commits caused a
  new conflict, so we aborted and used a safer plan that only reworded/
  fixed up two commits without reordering anything else).

I did not have Claude write the Comment 4 or Comment 5 responses
directly — both are in my own words, refined after Claude asked
clarifying questions about CineLog's actual current features rather than
generating an argument for me.

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
`pytest tests/ -v` to confirm the full suite passes with no
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
I'm implementing date-added order (most recent first), matching the
maintainer's suggestion.

**Reasoning:**
Date-added order is the most intuitive for a watchlist because it reflects 
the user's recent interests and recommendations. When users add films to 
their watchlist, they are often doing so based on what they've recently 
discovered or are currently interested in, rather than an alphabetical 
or other arbitrary order. This allows users to quickly access the films 
they are most likely to want to watch next, aligning with the dynamic 
and evolving nature of a watchlist compared to a static film catalog 
or an already-watched collection.

**Engagement with reviewer's point:**
I agree with the maintainer's reasoning that sorting by date-added is more 
user-friendly for a watchlist because it prioritizes the most recently 
added films, which are likely to be of immediate interest to the user. 
This is particularly relevant for a watchlist, as it serves as a dynamic 
list of films the user intends to watch in the near future, rather than a 
static collection of previously watched films or a browsable catalog.
I considered the alphabetical alternative, which could be useful for very 
long lists or for users wanting to check if a specific title has already 
been added. However, I decided that date-added order still wins because 
it better reflects the user's current interests and viewing intentions, 
and the need to check for duplicates can be addressed through other 
means (e.g., search functionality as a future feature). This also brings 
`get_watchlist()` in line with `get_collection()`, which already sorts 
by `date_added.desc()`, ensuring consistency across the app's features.

## Comment 6 — Rebase
**What conflicted:** Two conflicts during `git rebase origin/main`:
1. `.gitignore` — both `main` and my branch independently added one;
   resolved by keeping the union of both (`main` had `.pytest_cache/`
   that mine didn't).
2. `models.py` — `main`'s refactor commit changed `Film.id` and
   `CollectionEntry.film_id` from `Integer` to `String(36)` (UUID), and
   had no `WatchlistEntry` class at all, while my branch had
   `WatchlistEntry` still referencing `film_id` as `Integer`.

**How I resolved it:** Kept `main`'s UUID-based `User`, `Film`, and
`CollectionEntry` models untouched, and added my `WatchlistEntry` class
back in with `film_id` changed from `db.Column(db.Integer, ...)` to
`db.Column(db.String(36), ...)` to match the new `Film.id` type. After
the rebase completed with no further conflicts, I found two leftover
stale references to the old integer ID scheme that the rebase didn't
touch (since git found no textual conflict there): a docstring in
`add_to_watchlist()` still said `film_id (int)`, and the route docstring
in `routes/watchlist/watchlist.py` still showed `"film_id": <int>` in
its example request body. Updated both to reflect UUID strings. Also
updated `tests/test_watchlist.py`'s fake `film_id` from an integer
(`999999`) to a UUID-formatted string
(`"00000000-0000-0000-0000-000000000000"`), matching
`test_collection.py`'s pattern and actually testing the right thing now
that `Film.id` is a UUID.

**How I verified no conflict remains:** `git log --oneline origin/main..HEAD`
shows a linear history with no merge commits. Ran `pytest tests/ -v`
after the rebase and after each cleanup fix — all 5 tests pass throughout.

## Stretch — remove_from_watchlist()
**What I did:** Added `remove_from_watchlist(user_id, film_id)` to
`services/watchlist_service.py`, following the same query-then-raise
pattern as `remove_from_collection()`. Raises a new `NotInWatchlistError`
if the entry doesn't exist, otherwise deletes it and returns `True`.
Added a corresponding `DELETE /watchlist/<user_id>/remove` route in
`routes/watchlist/watchlist.py`, matching the existing `/add` route's
request/response shape.

**Tests:** Added two tests — one confirming a successful removal actually
deletes the `WatchlistEntry` row, and one confirming `NotInWatchlistError`
is raised when trying to remove a film that was never added.

## Stretch — Second Test
**What I did:** Added `test_get_watchlist_returns_newest_first` to
`tests/test_watchlist.py`, modeled on
`test_get_collection_returns_newest_first` in `test_collection.py`. It
creates two `WatchlistEntry` rows for the same user with different
`date_added` timestamps and asserts that `get_watchlist()` returns the
more recently added film first.

**Why this case:** Of the six review comments, Comment 5's sort-order fix
was the only change with zero test coverage — every other comment's fix
had a corresponding test already (Comment 3 got its own dedicated test,
and the rename/dedup changes are implicitly exercised by the existing
tests). This closed that gap.

**Bonus finding:** Writing this test surfaced a real, previously-unnoticed
bug: `Film` only had a `db.relationship` backref for `CollectionEntry`,
not for `WatchlistEntry`. This meant `entry.film` inside `get_watchlist()`
raised an `AttributeError` any time the watchlist was actually populated
and read back — but no existing test happened to exercise that path, so
it had never been caught. Fixed by adding
`watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)`
to the `Film` model in `models.py`.

## Stretch — Visibility Toggle Endpoint
**What I did:** Added a `public` parameter (default `False`) to
`add_to_watchlist(user_id, film_id, public=False)` in
`services/watchlist_service.py`, so callers can explicitly set visibility
per entry instead of always relying on the model column default. Updated
`POST /watchlist/<user_id>/add` in `routes/watchlist/watchlist.py` to
read an optional `"public"` field from the request body
(`data.get("public", False)`), defaulting to private if omitted,
consistent with the Comment 4 decision.

**How a caller uses it:** `POST /watchlist/<user_id>/add` with body
`{ "film_id": "<uuid>" }` creates a private entry (the default). Sending
`{ "film_id": "<uuid>", "public": true }` creates a public entry instead.

**Test:** Added `test_add_to_watchlist_public_default_and_override`,
confirming a new entry defaults to `public=False` when the argument is
omitted, and correctly stores `public=True` when explicitly passed.

## PR Description
### What this adds

Adds a watchlist feature to CineLog, letting users save films they want
to watch (as opposed to the Collection feature, which tracks films
they've already watched). Includes a new `WatchlistEntry` model, service
functions (`add_to_watchlist`, `remove_from_watchlist`, `get_watchlist`),
and REST endpoints (`GET /watchlist/<user_id>`,
`POST /watchlist/<user_id>/add`, `DELETE /watchlist/<user_id>/remove`).

### Design decisions

- **Default visibility:** Watchlist entries default to `public=False`
  (private). CineLog currently has no feed, friend list, or way to browse
  other users' watchlists, so a public default wouldn't deliver on the
  app's "community" value today — it would just expose users' viewing
  habits with no corresponding benefit. Visibility can be set explicitly
  via the `public` field on `POST /watchlist/<user_id>/add`.
- **Sort order:** `get_watchlist()` returns entries sorted by date added,
  most recent first (matching `get_collection()`'s existing behavior),
  rather than alphabetically. A watchlist reflects current interest —
  users want to see what they recently added, not scan an alphabetized
  list.

### How to test manually

1. Start the app: `python app.py`
2. Create a user and a film (via existing endpoints or directly in a
   Python shell using the models).
3. `POST /watchlist/<user_id>/add` with body `{"film_id": "<uuid>"}` —
   confirm a 201 response and that the entry defaults to `public: false`.
4. `POST /watchlist/<user_id>/add` again with the same `film_id` — confirm
   it returns an error rather than creating a duplicate.
5. Add a second film with `{"film_id": "<uuid2>", "public": true}` —
   confirm the response shows `public: true`.
6. `GET /watchlist/<user_id>` — confirm the second (public, more recently
   added) film appears before the first.
7. `DELETE /watchlist/<user_id>/remove` with body `{"film_id": "<uuid>"}` —
   confirm a 200 response, and that `GET /watchlist/<user_id>` no longer
   shows that film.
8. Run `pytest tests/ -v` — confirm all 9 tests pass.


## git log --oneline - Screenshot
<img width="883" height="367" alt="Screenshot 2026-07-12 at 8 30 54 PM" src="https://github.com/user-attachments/assets/5ffdfd4f-27d3-4a08-a5ea-63a7491b796f" />
