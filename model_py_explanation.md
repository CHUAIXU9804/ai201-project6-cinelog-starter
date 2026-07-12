# Models.py
Responsibility
It defines the SQLAlchemy data layer for CineLog — a film-logging app. It declares four ORM models (database tables) and the relationships between them. Per the header comment, this file is on the feature/watchlist branch, reflecting state before a main-branch refactor that migrated film IDs from Integer to UUID (a detail that matters for merge conflicts — see note below).

The models
User (models.py:17) — account record. UUID string PK, unique username/email, created_at. Has many CollectionEntry rows.
Film (models.py:29) — a movie. Integer autoincrement PK, plus title/year/director/genre/poster/average_rating.
CollectionEntry (models.py:52) — a film a user has watched and logged. Links a user↔film with an optional 1–5 rating and a UniqueConstraint preventing duplicate user+film pairs.
WatchlistEntry (models.py:74) — a film a user wants to watch. Similar to CollectionEntry but no rating; instead has a public boolean. Notably it has no unique constraint.
Main functions
generate_uuid() (models.py:13) — returns a UUID4 string; used as the default for every UUID primary key.
.to_dict() on each model — serializes a row into a plain dict for JSON API responses. CollectionEntry/WatchlistEntry call .date_added.isoformat().
Dependencies
app.db (models.py:10) — imports the shared SQLAlchemy db instance from app.py. Every model subclasses db.Model, so this file is tightly coupled to app.py's DB setup.
Standard library — uuid, and datetime/timezone for UTC timestamps.
Consumers — whatever route/view code (likely in app.py or a routes module) queries these models and calls .to_dict().
One thing worth flagging
There's an inconsistency in the foreign keys: Film.id is an Integer, and both CollectionEntry.film_id (models.py:56) and WatchlistEntry.film_id (models.py:78) are Integer FKs to it — while User.id and all entry PKs are UUID strings. The header comment says main has since migrated film IDs to UUID, so this branch will conflict with main on the Film model and the two film_id columns.

Would you like me to look at app.py to map out exactly how these models are consumed?


# Collection_service.py
Here's a summary of services/collection_service.py:

Responsibility
It's the business-logic (service) layer for a user's film collection — films they've already watched and logged. It sits between the route handlers and the database: routes call these functions, and the functions handle validation, DB queries, and commits. It also defines the domain-specific exceptions the routes translate into HTTP responses.

Exceptions (lines 12–24)
FilmNotFoundError — the given film_id doesn't exist.
AlreadyInCollectionError — film is already in the collection (enforces the "no duplicates" rule at the app level, mirroring the DB's UniqueConstraint).
NotInCollectionError — trying to remove a film that isn't there.
Main functions
add_to_collection(user_id, film_id, rating=None) (line 27) — verifies the film exists, checks it isn't already logged, then creates and commits a new CollectionEntry. Returns the entry. Raises FilmNotFoundError / AlreadyInCollectionError.
remove_from_collection(user_id, film_id) (line 61) — finds the matching entry, deletes and commits it. Returns True, or raises NotInCollectionError.
get_collection(user_id) (line 88) — fetches all of the user's entries newest-first, then returns a list of film dicts (via entry.film.to_dict()) with the entry's date_added and rating merged in — so the caller gets film info + collection metadata in one flat object.
Dependencies
app.db (line 8) — the shared SQLAlchemy session, used for get, add, delete, and commit.
models.Film and models.CollectionEntry (line 9) — the ORM models it queries and creates. It relies on the CollectionEntry.film backref (from models.py:38) inside get_collection, and on Film.to_dict().
Consumers — route handlers (in routes/) that call these functions and catch the exceptions.
One thing worth flagging
The docstrings say film_id is a UUID string, but on this branch models.py still defines Film.id as an Integer. This is the same integer-vs-UUID mismatch I noted earlier — the docstrings here appear to have been written against (or in anticipation of) the main-branch UUID refactor, while the models haven't caught up. Worth reconciling when this branch merges.

Want me to look at the matching route file or watchlist_service.py to complete the picture? 


# Collection_Service.py Add_to_Collection()
Give the AI a function and ask: "What does this function do? Walk me through what happens at each step, and what it returns if the film_id doesn't exist." Use this on add_to_collection()

Here's a walkthrough of add_to_collection(user_id, film_id, rating=None) (collection_service.py:27–58).

What it does, overall
It records that a user has watched a film by creating a CollectionEntry row — but only after two safety checks: the film must exist, and it must not already be in that user's collection.

Step by step
1. Look up the film (line 43)


film = db.session.get(Film, film_id)
Fetches the Film row by primary key. db.session.get() returns the object if found, or None if no row has that id.

2. Guard: film must exist (lines 44–45)


if film is None:
    raise FilmNotFoundError(f"No film found with id '{film_id}'")
If the lookup returned None, it stops here (see the "what if it doesn't exist" section below).

3. Check for a duplicate (lines 47–49)


existing = CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()
Queries for an existing entry matching this user and this film. .first() gives the first match or None.

4. Guard: no duplicates (lines 50–53)


if existing:
    raise AlreadyInCollectionError(...)
If the user already logged this film, it raises rather than creating a second row. This mirrors the DB-level UniqueConstraint on (user_id, film_id) in models.py:60–62 — catching it here gives a clean, specific error instead of a raw database IntegrityError.

5. Create, persist, return (lines 55–58)


entry = CollectionEntry(user_id=user_id, film_id=film_id, rating=rating)
db.session.add(entry)
db.session.commit()
return entry
Builds the new entry (with the optional rating), stages it, commits it to the database, and returns the persisted CollectionEntry object. After the commit its auto-generated fields — id (a UUID) and date_added — are populated.

What happens if film_id doesn't exist
It does not return a value at all. At step 2, db.session.get() yields None, so the function raises FilmNotFoundError with the message No film found with id '<film_id>'. Execution stops immediately — the duplicate check never runs, nothing is added to the session, and nothing is committed. The caller (a route handler) is expected to catch FilmNotFoundError and turn it into an error response, typically an HTTP 404.

So the three possible outcomes are: return a new CollectionEntry (success), raise FilmNotFoundError (unknown film), or raise AlreadyInCollectionError (duplicate).

One caveat tied to what I flagged earlier: the docstring says film_id is a UUID, but on this branch Film.id is an Integer (models.py:30). If a UUID string is actually passed, the db.session.get(Film, ...) lookup won't match any integer PK, so a valid-looking film could be wrongly reported as FilmNotFoundError.

