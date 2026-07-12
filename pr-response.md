# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->
I used Claude to help myself understand the structure of each file, and logic of each function, and verify commit format for the commit history.

## Comment 1 — Rename
**What I did:** Renamed save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py and update all call sites

**How I verified:** I checked services/watchlist_service.py to rename the function, and used the project-wide search feature in my IDE to update all the call sites

## Comment 2 — Deduplication
**What I did:** Added deduplication logic to add_to_watchlist() in services/watchlist_service.py
**How I verified:** Used add_to_collection() in services/collection_service.py as reference, since the watchlist_service follows the similar structure, I could easily update the deduplication logic for add_to_watchlist()

## Comment 3 — Missing test
**What I did:** Created a new file tests/test_watchlist.py. Wrote the equivalent test for add_to_watchlist() following the same fixture and assertion structure. 
**How I verified:** I used the test_add_to_collection_nonexistent_film_raises() in tests/test_collection.py as reference, and implemented the same logic for the file to return exception for non-existent films.

## Comment 4 — Default visibility
**My position:** The watchlist visibility default should be public. 
**Reasoning:** A watchlist is a list of films a user wants to watch or save for later. Beyond personal tracking, a public default supports the product's social goals — letting users share what they're interested in with friends and helping surface and promote films to other users — without requiring every user to hunt for a setting to turn sharing on.
**Tradeoff acknowledged:** The honest objection is that safe defaults for personal data are usually private, and the harm here is asymmetric: a private default that a user wants to share is reversible (flip one setting), but a public default the user didn't want is not — the data may already have been seen or cached before they notice. I also acknowledge the discovery benefit accrues mainly to the platform while the privacy cost falls on the user. And the current code makes a public default hard to fully honor: `add_to_watchlist()` takes no `public` argument (so entries can't opt out at creation) and `get_watchlist()` doesn't filter on `public`, so the follow-up work is to add a `public` param to `add_to_watchlist()` and filter on `public=True` when serving another user's watchlist.

## Comment 5 — Sort order
**My position:** I agree with the maintainer's preference to sort the watchlist by date added (newest first). 
**Reasoning:** Sorting by date added surfaces what the user most recently wanted to watch, and matching the collection's ordering keeps the two views consistent so the app behaves predictably. It also lets users track when they came across each film, which reinforces the sense of building a personal collection over time.
**Engagement with reviewer's point:** I agree with the reviewer, I believe the design to sort movies by date added make the most logical sense as it allows users to keep track of what they added recently.

## Comment 6 — Rebase
**What conflicted:** After `git fetch origin` and `git rebase origin/main`, the conflict was in `models.py`. Main had refactored film IDs from integer to UUID (`Film.id` and `CollectionEntry.film_id` became `db.String(36)`), while my `feature/watchlist` branch had added the new `WatchlistEntry` model with an integer `film_id`. So the same file was changed on both sides: main's UUID migration overlapped with my new model. Naively taking main's version of `models.py` resolved the type conflict but silently dropped `WatchlistEntry` entirely (main never had it), which broke `services/watchlist_service.py` — its `from models import Film, WatchlistEntry` raised `ImportError` and the whole watchlist feature failed to load.
**How I resolved it:** I reconciled both sides instead of choosing one. I restored `WatchlistEntry` to `models.py` in its post-refactor form: `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))` (UUID, matching `CollectionEntry.film_id`), keeping its original fields (`public` defaulting to `True`, no rating). I also updated the stale docstring in `add_to_watchlist()` in `services/watchlist_service.py`, which still described `film_id` as `int (pre-refactor)`, to `str (UUID of the film)`.
**How I verified no conflict remains:** `git diff --check` reports no conflict markers. `services.watchlist_service` now imports cleanly and `WatchlistEntry.film_id` resolves to `VARCHAR(36)`. The full test suite passes (5 passed — 4 collection tests + `test_add_to_watchlist_nonexistent_film_raises`). History is linear — `git log --merges origin/main..HEAD` returns nothing, confirming the branch was rebased rather than merged and no merge commits remain.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — films a user wants to watch later, kept separate from the existing "collection" (films already watched). It introduces:
- **`WatchlistEntry` model** (`models.py`) — links a user to a film, with a `date_added` timestamp and a `public` visibility flag.
- **Service layer** (`services/watchlist_service.py`):
  - `add_to_watchlist(user_id, film_id)` — validates the film exists (`FilmNotFoundError`) and rejects duplicates (`AlreadyInWatchlistError`).
  - `get_watchlist(user_id)` — returns the user's saved films with `date_added` and `public` attached.
- **REST endpoints** (`routes/watchlist/watchlist.py`, registered under `/watchlist`):
  - `GET /watchlist/<user_id>` — view a user's watchlist.
  - `POST /watchlist/<user_id>/add` with body `{"film_id": "<uuid>"}` — add a film.

### Design decisions
- **Default visibility = public** (Comment 4): supports the product's sharing/discovery goals. The honest tradeoff (privacy-by-default norms, irreversible disclosure) and follow-up work (a `public` param on `add_to_watchlist` + filtering in `get_watchlist`) are documented above.
- **Sort order = date added** (Comment 5): keeps the watchlist consistent with `get_collection()`'s ordering; the tradeoff vs. alphabetical (harder lookup as the list grows) is noted, with a user-selectable sort as the long-term answer.
- **`film_id` is a UUID** (Comment 6): reconciled with main's integer→UUID refactor so `WatchlistEntry.film_id` matches `Film.id` and `CollectionEntry.film_id`.
- **Deduplication** mirrors `add_to_collection()` but raises a watchlist-specific `AlreadyInWatchlistError` rather than reusing the collection's exception.

### How to manually test
```bash
# 1. Set up and run
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python app.py                     # serves on http://127.0.0.1:5000

# 2. Create a user and a film (via a Python shell or existing endpoints) and note
#    their UUIDs, then:

# Add a film to the watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
# → 201 with the new entry (public: true)

# View the watchlist
curl http://127.0.0.1:5000/watchlist/<user_id>
# → JSON list of films, each with date_added and public
```
Expected edge-case behavior:
- Adding the **same film twice** → `AlreadyInWatchlistError` (no duplicate created).
- Adding a **nonexistent `film_id`** → `FilmNotFoundError`.

Automated coverage: `python -m pytest tests/` — 5 tests pass, including `test_add_to_watchlist_nonexistent_film_raises`.

