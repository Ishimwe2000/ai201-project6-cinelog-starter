# PR Response Doc — CineLog Watchlist Feature

## AI Usage

Used Claude Code as a pair programmer, directing each step rather than a single end-to-end run.
- Diagnosed the `feature/watchlist` vs `origin/feature/watchlist` divergence; traced it to a main-rebase conflict that had silently dropped `WatchlistEntry` from `models.py` (confirmed via `pytest` `ImportError`).
- Fixed Comment 6 by restoring just `WatchlistEntry` (UUID-adjusted), leaving the separate backref bug (Comment 5) alone.
- Caught that Comment 4's doc claimed a visibility toggle that didn't exist in code; had it implemented for real instead of editing the claim away.
- All "tests pass" / curl-verification claims below were actually run, not just asserted.
- I made the design calls (Comments 4, 5), the scope decision to leave the backref bug unfixed, and commit structure/messages.

## Comment 1 — Rename
**Did:** Renamed `save_to_watchlist()` → `add_to_watchlist()` (matches `verb_to_noun` convention). Updated the one call site.
**Verified:** `grep` confirms no remaining references; full suite passes.

## Comment 2 — Deduplication
**Did:** Added `AlreadyInWatchlistError`, following `AlreadyInCollectionError`'s pattern — `add_to_watchlist()` checks for an existing `(user_id, film_id)` row before inserting. Route now catches it → 409 (and `FilmNotFoundError` → 404).
**Verified:** `test_add_to_watchlist_duplicate_raises` (Comment 3).

## Comment 3 — Missing test
**Did:** Added `tests/test_watchlist.py` with the requested nonexistent-film test, plus happy-path and duplicate tests, per CONTRIBUTING.md's convention (happy path / conflict / bad ID).
**Verified:** All pass.

## Comment 4 — Default visibility
**Position:** Default `public=True`. CineLog's watchlist is a discovery feature — public-by-default gives it immediate social value; private is one flag away for users who want it.
**Tradeoff:** Private-by-default is the safer norm and watchlists can be more revealing than collections, but I weighted discovery value higher for this feature.
**Did:** Added optional `public` param to `add_to_watchlist()`, threaded through the route's JSON body (`public`, defaults `true`).
**Verified:** New tests for default/explicit-false; manually curled the running app to confirm both cases in the response body.

## Comment 5 — Sort order
**Position:** Sort by `date_added` descending, not alphabetically — matches `get_collection()`'s existing sort key and shows recently-added items first.
**Known pre-existing bug (out of scope):** `get_watchlist()` raises `AttributeError: 'WatchlistEntry' object has no attribute 'film'` — `models.py` never defines a `Film ↔ WatchlistEntry` relationship (unlike `CollectionEntry`). Not one of the six comments, so flagged here rather than fixed. The sort-order change itself is correct but couldn't be tested against `get_watchlist()` without also fixing this.

## Comment 6 — Rebase
**What conflicted:** Rebasing onto `main` put the branch's old `models.py` (`WatchlistEntry.film_id` as `Integer`) against main's UUID-migration refactor. Resolving that conflict dropped `WatchlistEntry` entirely instead of updating its type, breaking imports in the service and tests.
**Resolved:** Re-added `WatchlistEntry` with `film_id` as `String(36)` to match the UUID schema; nothing else changed.
**Verified:** Full suite collects and passes again (previously couldn't even collect `test_watchlist.py`).

## PR Description

Adds a watchlist to CineLog — a list of films a user wants to watch, separate from their collection. `POST /watchlist/<user_id>/add` adds a film (public by default, optional `"public": false`); `GET /watchlist/<user_id>` lists it newest-first.

**Design decisions:** public-by-default visibility (Comment 4), sort by `date_added` desc (Comment 5), `add_to_watchlist` naming (Comment 1), duplicate → 409 (Comment 2).

**Manual test:**
1. `python app.py`
2. Seed a user/film via a Python shell (no create endpoints exist yet)
3. `POST .../add {"film_id": "<uuid>"}` → 201, `public: true`
4. Repeat → 409; bogus film_id → 404
5. `POST .../add {"film_id": "<uuid2>", "public": false}` → 201, `public: false`

**Known limitation:** `GET /watchlist/<user_id>` 500s — see Comment 5's flagged bug. Not fixed here; unrelated to the six comments.
