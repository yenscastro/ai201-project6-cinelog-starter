# PR Response — feature/watchlist

Thanks for the thorough review! Here are my responses and the changes I made for each comment.

---

## Comment 1 — Rename `save_to_watchlist` to `add_to_watchlist`

**Status: Fixed.**

Renamed the service function to `add_to_watchlist` so it matches the `verb_to_noun` naming convention documented in `CONTRIBUTING.md` and already used in `collection_service.py` (`add_to_collection`, `remove_from_collection`, `get_collection`).

Updated in three places:
- `services/watchlist_service.py` — function definition.
- `routes/watchlist/watchlist.py` — the import in the `from services.watchlist_service import (...)` block.
- `routes/watchlist/watchlist.py` — the call site inside `add_film_to_watchlist()`.

The full watchlist service now reads: `add_to_watchlist()`, `remove_from_watchlist()`, `get_watchlist()` — parallel to the collection service.

---

## Comment 2 — Add deduplication logic

**Status: Fixed.**

`add_to_watchlist()` now checks for an existing entry before inserting, mirroring `add_to_collection()`:

```python
existing = WatchlistEntry.query.filter_by(
    user_id=user_id, film_id=film_id
).first()
if existing:
    raise AlreadyInWatchlistError(
        f"Film '{film_id}' is already in this user's watchlist"
    )
```

The route maps `AlreadyInWatchlistError` to HTTP **409 Conflict**. This is defense-in-depth alongside the DB-level `UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")` on the `WatchlistEntry` model — the service check gives a clean, typed error instead of surfacing a raw `IntegrityError`.

---

## Comment 3 — Add missing test

**Status: Fixed.**

Added `test_add_to_watchlist_nonexistent_film_raises` to `tests/test_watchlist.py`, following the exact pattern from `test_collection.py`:

```python
def test_add_to_watchlist_nonexistent_film_raises(app, sample_user):
    """Adding a film_id that doesn't exist should raise FilmNotFoundError."""
    with app.app_context():
        fake_film_id = "00000000-0000-0000-0000-000000000000"

        with pytest.raises(FilmNotFoundError):
            add_to_watchlist(user_id=sample_user, film_id=fake_film_id)
```

The suite now covers the three cases required by `CONTRIBUTING.md`: happy path (`test_add_to_watchlist_creates_entry`), conflict (`test_add_to_watchlist_duplicate_raises`), and nonexistent ID (`test_add_to_watchlist_nonexistent_film_raises`), plus sort-order coverage (`test_get_watchlist_returns_newest_first`). All tests pass.

---

## Comment 4 — Default visibility (`public=True`)

**Decision: Keep `public=True` as the default.** (Open to changing if the team disagrees.)

Reasoning:

- **CineLog is community-driven.** The core value of the app is discovery — seeing what other people want to watch is how users find new films. A public-by-default watchlist maximizes that shared surface area; a private-by-default one would leave most watchlists invisible and hollow out the social feed.
- **Consistency with existing social features.** The rest of CineLog leans public: collection entries and average ratings are visible so films can be discovered and ranked. Making the watchlist public by default keeps the mental model consistent — "what I log is shareable unless I say otherwise."
- **The tradeoff is real but bounded.** Some users will want a private "someday" list. We handle that by letting them opt out per entry rather than forcing everyone into a private default.
- **Users can always override.** The `add` endpoint accepts an optional `public` flag, so a user who wants a private entry sends `{ "film_id": "...", "public": false }`. The default only sets the *common* case; it doesn't remove the choice.

If the team decides privacy-by-default is safer, the change is a one-liner (`public=True` → `public=False` in the model column default and the service signature) — happy to flip it.

---

## Comment 5 — Sort order (`date_added` descending)

**Decision: Keep `date_added` descending (newest first).** (Acknowledging the reviewer may prefer alphabetical.)

Reasoning:

- **Recency is what users act on.** A watchlist is a queue of intent — "what do I want to watch next?" The most recently added films are top of mind, so surfacing them first matches how people actually use the list.
- **Consistency with the collection feature.** `get_collection()` already sorts `CollectionEntry.date_added.desc()`. Using the same order in `get_watchlist()` keeps behavior predictable across the two nearly identical features, and keeps the code parallel.
- **Alphabetical was considered.** It's better for *looking up a known title* in a large list, but it buries newly added films and makes the list feel static — every add lands in the middle somewhere instead of at the top where the user expects it.
- **I recognize this is a judgment call.** If the reviewer feels strongly about alphabetical, a good compromise is to keep `date_added desc` as the default and add an optional `?sort=title` query param later, so both behaviors are available without changing the default. I'm happy to file that as a follow-up.

---

## Comment 6 — Rebase onto main (UUID migration)

**Status: Resolved.**

`main` was refactored from integer IDs to UUIDs (commit `refactor: migrate film IDs from integer to UUID`). The watchlist code is written to be UUID-native, so no integer assumptions remain:

- `WatchlistEntry.id`, `.user_id`, and `.film_id` are all `db.String(36)` with a `generate_uuid` default — matching `Film`, `User`, and `CollectionEntry`.
- Film lookups use `db.session.get(Film, film_id)` with the string UUID directly — no `int(film_id)` casting anywhere.
- Tests use UUID-format IDs (e.g. `"00000000-0000-0000-0000-000000000000"`), not integers.

The rebase and conflict-resolution commands are in the PR notes below.

---

## Stretch features

- **Remove:** `remove_from_watchlist(user_id, film_id)` added, following `remove_from_collection()` — raises `NotInWatchlistError` (mapped to 404) when the entry doesn't exist. Exposed as `DELETE /watchlist/<user_id>/remove`.
- **Visibility toggle:** `POST /watchlist/<user_id>/add` accepts an optional `public` field in the JSON body (`data.get("public", True)`), so callers can create private entries explicitly.
