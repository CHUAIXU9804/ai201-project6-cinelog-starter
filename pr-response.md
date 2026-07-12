# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

