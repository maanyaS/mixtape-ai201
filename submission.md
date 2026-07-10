# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy social music app: friends share songs, build
collaborative playlists, rate songs, track listening streaks, and see what
their friends are playing. It's organized as a clean three-layer stack —
**routes → services → models** — backed by a SQLite database.

---

## Main files and what each does

### Bootstrap

- **[app.py](app.py)** — Application factory. Creates the shared `db =
  SQLAlchemy()` instance, and `create_app()` configures it (DB URI from
  `DATABASE_URL`, default `sqlite:///mixtape.db`), binds the DB, registers the
  four route blueprints under URL prefixes (`/songs`, `/playlists`, `/users`,
  `/feed`), and runs `db.create_all()`. Contains no business logic.

- **[models.py](models.py)** — The full data model (see below). Also defines
  three raw association tables and a `generate_uuid()` helper (all primary keys
  are string UUIDs, not autoincrement ints).

- **[seed_data.py](seed_data.py)** — Drops and recreates all tables, then
  populates 5 users with friendships, 25 songs (deliberately grouped into 0-tag,
  1-tag, and 3+-tag sets), 3 playlists, listening events spanning the past two
  weeks, streaks, and a sample notification. The seed data is engineered to
  expose the app's bugs (e.g. the multi-tag songs surface the search-duplication
  issue).

### Models ([models.py](models.py))

Seven models plus three association tables:

| Model | Purpose | Notable fields |
|---|---|---|
| `User` | An account | `listening_streak`, `last_listened_at`; self-referential `friends` many-to-many |
| `Song` | A shared track | `shared_by` (FK to sharer), `share_note`; many-to-many `tags` |
| `Tag` | A genre/mood label | `name` (unique) |
| `Rating` | A user's 1–5 score for a song | `UniqueConstraint(user_id, song_id)` — one rating per user per song |
| `ListeningEvent` | One playback event | `user_id`, `song_id`, `listened_at` — the source of both streaks and feeds |
| `Playlist` | A named collection | `created_by`, `is_collaborative` |
| `Notification` | An in-app alert | `notification_type`, `body`, `read` |

Association tables:
- **`friendships`** — symmetric user-to-user (seeded with a row in *each*
  direction, so friendship is stored bidirectionally).
- **`song_tags`** — song-to-tag many-to-many.
- **`playlist_entries`** — playlist-to-song, and crucially it carries extra
  columns: **`position`** (explicit ordering, not insertion order), `added_by`,
  and `added_at`. This is why playlist songs have a defined sequence.

Every model has a `to_dict()` for JSON serialization — the routes never touch
model objects directly in their responses; they serialize through these.

### Routes (thin HTTP layer — one blueprint per file)

Each route parses input, delegates immediately to a service, and formats the
response (including turning service `ValueError`s into 400/404 JSON). No
business logic lives here.

- **[routes/songs.py](routes/songs.py)** — `GET /songs/search`,
  `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- **[routes/playlists.py](routes/playlists.py)** — `POST /playlists/`,
  `GET /playlists/<id>`, `GET /playlists/<id>/songs`,
  `POST /playlists/<id>/songs`.
- **[routes/users.py](routes/users.py)** — `GET /users/<id>`,
  `GET /users/<id>/streak`, `GET /users/<id>/notifications`,
  `POST /users/notifications/<id>/read`. (This is the one file that also queries
  a model — `User` — directly for the simple lookup.)
- **[routes/feed.py](routes/feed.py)** — `GET /feed/<id>/listening-now`,
  `GET /feed/<id>/activity`.

### Services (business logic — where the real work happens)

- **[services/search_service.py](services/search_service.py)** —
  `search_songs()` (case-insensitive `ilike` on title/artist, with an
  `outerjoin` to `song_tags`), `get_song()`.
- **[services/playlist_service.py](services/playlist_service.py)** —
  `create_playlist()`, `get_playlist_songs()` (orders by
  `playlist_entries.position`), `get_playlist()`, `get_user_playlists()`.
- **[services/notification_service.py](services/notification_service.py)** —
  `create_notification()` (the shared helper), `add_to_playlist()` (adds a song
  *and* notifies the sharer), `rate_song()` (upsert of a `Rating`),
  `get_notifications()`, `mark_as_read()`.
- **[services/streak_service.py](services/streak_service.py)** —
  `record_listening_event()`, `update_listening_streak()` (consecutive-day
  logic), `get_streak()`.
- **[services/feed_service.py](services/feed_service.py)** —
  `get_friends_listening_now()` (friends active within a recency cutoff, one
  song per friend), `get_activity_feed()` (last N friend events, unfiltered by
  time).

### Tests ([tests/](tests/))

`pytest` suites — `test_streaks.py`, `test_search.py`, `test_playlists.py` —
each targeting a service directly.

---

## Data flow: adding a song to a playlist triggers a notification

A concrete trace through all three layers:

1. **HTTP** — Client sends `POST /playlists/<playlist_id>/songs` with JSON
   `{ "song_id": ..., "added_by": ... }`.
2. **Route** — [`add_song()` in routes/playlists.py](routes/playlists.py#L43)
   parses `song_id` and `added_by`, validates both are present (else 400), and
   calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. **Service** — [`add_to_playlist()` in
   notification_service.py](services/notification_service.py#L35):
   - Loads and validates the `Song`, the adder `User`, and the `Playlist`
     (raising `ValueError` → 400 if any is missing).
   - Appends the song to `playlist.songs` (via the `playlist_entries` join) if
     it isn't already present, and commits.
   - **If the adder is not the song's original sharer** (`song.shared_by !=
     added_by_user_id`), calls `create_notification()` to write a
     `song_added_to_playlist` Notification addressed to `song.shared_by`.
4. **Persistence** — `create_notification()` inserts the `Notification` row and
   commits. The sharer later retrieves it via
   `GET /users/<id>/notifications` → `get_notifications()`.

The same pattern applies to rating: `POST /songs/<id>/rate` → `rate_song()`
upserts a `Rating`. Note that rating currently does **not** create a
notification — only playlist-adds do — which is the kind of asymmetry worth
flagging.

---

## Patterns I noticed

1. **Strict three-layer separation.** Routes are uniformly thin: parse input →
   call one service → serialize/handle errors. All logic lives in `services/`,
   all schema in `models.py`. The one small exception is `routes/users.py`,
   which does a direct `db.session.get(User, ...)` for a trivial lookup.

2. **`ValueError` as the cross-layer error channel.** Services raise
   `ValueError("... not found")`; every route wraps its call in
   `try/except ValueError` and maps it to a 400 or 404. This is the single,
   consistent error-handling convention across the whole app.

3. **`to_dict()` serialization boundary.** Nothing returns a raw model to the
   client — services return `dict`s (or model instances the route immediately
   `.to_dict()`s). The JSON shape is owned by the models.

4. **String-UUID primary keys everywhere**, generated app-side via
   `generate_uuid()`, rather than DB autoincrement.

5. **Rich join tables over plain M2M.** `playlist_entries` carries
   `position`/`added_by`/`added_at`, so ordering and provenance are first-class
   — a deliberate design choice that the playlist-ordering logic depends on.

6. **Everything derives from `ListeningEvent`.** Both the streak logic and both
   feeds are computed from the same event stream, keeping a single source of
   truth for "what a user played and when."

7. **Service-to-service reuse with lazy imports.** `notification_service`
   imports from `playlist_service` (and `models.Playlist`) *inside* the function
   to sidestep circular imports — a recurring tactic (mirrored by the deferred
   blueprint imports in `app.py`).

---

## Root Cause Analysis

### Bug #1 — My listening streak keeps resetting

**How I reproduced it.** Ran the existing suite,
`.venv\Scripts\python.exe -m pytest tests/ -q`, and saw
`test_streaks.py::test_streak_increments_on_sunday` fail with `assert 1 == 2`.
The triggering condition: a user whose `last_listened_at` is a **Saturday**
(`2024-06-15`) with `listening_streak == 1`, who then listens on the **Sunday**
exactly one calendar day later (`2024-06-16`). Sequence:
`update_listening_streak(user, saturday)` → streak 1, then
`update_listening_streak(user, sunday)` → streak stays 1 instead of becoming 2.
The bug fires for *any* consecutive-day listen that happens to land on a Sunday.

**How I found the root cause.** The failing test named the function directly, so
I opened [services/streak_service.py](services/streak_service.py) and read
`update_listening_streak()`. The docstring lists four rules — new user → 1, same
day → no change, yesterday → +1, gap → reset — none of which mention weekdays. I
then read the branch conditions and hit the moment of certainty at line 73:
the "listened yesterday" branch was `days_since_last == 1 and
today.weekday() != 6`. That extra weekday clause has no basis in the documented
rules, and `weekday() == 6` is exactly Sunday — matching the symptom precisely.

**The root cause.** Python's `datetime.weekday()` returns 6 for Sunday. The
increment branch required `days_since_last == 1 and today.weekday() != 6`, so
when the current listen fell on a Sunday the branch evaluated false *even though
exactly one day had elapsed*. Control then fell through to the `else` branch,
which resets the streak to 1. The result: a correct consecutive-day streak was
silently reset every Sunday. The `!= 6` guard was spurious logic unrelated to
the actual streak rule (consecutive calendar days).

**My fix and side-effect check.** Removed the `and today.weekday() != 6` clause,
leaving `elif days_since_last == 1:`. This restores the rule to "exactly one
calendar day since the last listen → increment," independent of weekday.
- **Both sides of the boundary:** `test_streak_increments_on_consecutive_day`
  (0→1→2 on Mon/Tue) confirms the normal in-week increment still works;
  `test_streak_increments_on_sunday` now passes, confirming the Sunday boundary;
  `test_streak_resets_after_skipped_day` (Mon→Wed) confirms a genuine
  ≥2-day gap *still* resets — the fix didn't over-correct into "never reset."
- **Same-day and new-user paths:** `test_streak_does_not_double_count_same_day`
  and `test_streak_starts_at_1_for_new_user` still pass, so the other branches
  are untouched.
- **Full suite:** all 5 streak tests pass; the only remaining failures are the
  unrelated playlist bug (#5). No regressions introduced.

### Bug #5 — The last song in a playlist never shows up

**How I reproduced it.** With the streak fix already in place, running
`.venv\Scripts\python.exe -m pytest tests/ -q` left two failures, both in
`tests/test_playlists.py`. The `seed_playlist` fixture builds a playlist with 5
songs (`Track 1`–`Track 5`) at positions 1–5. Calling
`get_playlist_songs(playlist_id)` then:
- `test_playlist_returns_all_songs` failed with `assert 4 == 5` — only 4 songs
  came back.
- `test_playlist_returns_songs_in_order` failed showing the list ended at
  `Track 4`; `Track 5` was missing.

The condition that triggers it: *any* non-empty playlist. The song that
disappears is always the one at the highest `position`.

**How I found the root cause.** The failing tests import `get_playlist_songs`
directly, so I opened
[services/playlist_service.py](services/playlist_service.py) and read that
function top to bottom. The SQL was correct — it joins `playlist_entries`,
filters by `playlist_id`, and orders by `asc(position)`, so `songs` holds all 5
rows in the right order. The certainty came at the `return` statement (line 66):
`return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice, applied
*after* a correct query, is exactly a drop-the-last-element operation — matching
"returns 4, expected 5" and "last song missing" precisely. The misleading
docstring ("returns all songs in the playlist") confirmed the slice was
unintended, not a deliberate filter.

**The root cause.** Python's `list[:-1]` slice returns every element *except the
last one*. The query produced the complete, correctly ordered list of songs, but
the return statement sliced off the final element before serializing. Because the
rows are ordered by ascending `position`, the dropped element is always the
last-positioned song — so the highest-position track in every non-empty playlist
was silently omitted from the response.

**My fix and side-effect check.** Changed `songs[:-1]` back to `songs` in the
comprehension — one character-range edit, no other logic touched.
- **Both sides of the boundary:** `test_playlist_returns_all_songs` and
  `test_playlist_returns_songs_in_order` now pass (all 5 songs, correct order —
  the "many songs" side); `test_empty_playlist_returns_empty_list` still passes
  (an empty playlist returns `[]` — the "zero songs" side, which the old
  `[:-1]` would also have returned as `[]`, but now for the right reason).
- **Related feature check:** searched for other users of the same data.
  `add_to_playlist()` in `notification_service.py` mutates the playlist via the
  `playlist.songs` relationship, not `get_playlist_songs()`, so it is unaffected;
  no route or service other than `routes/playlists.py::get_songs` consumes this
  function.
- **Full suite:** 13 passed, 0 failed. No regressions.

### Bug #2 — Friends Listening Now shows people from yesterday

**How I reproduced it.** No test exists for this, so I drove the service against
an in-memory DB. I created a user `me` with two friends: `recent` (one
`ListeningEvent` 10 minutes ago) and `stale` (one `ListeningEvent` 10 hours
ago), then called `get_friends_listening_now(me.id)`. Before the fix the feed
returned **both** friends — `stale` appeared even though a 10-hour-old listen is
in no sense "listening now." The triggering condition: any friend whose most
recent listening event is older than a real "now" window but still within the
past 24 hours.

**How I found the root cause.** The feed route
([routes/feed.py](routes/feed.py)) delegates straight to
[`get_friends_listening_now()`](services/feed_service.py#L16), so I opened
[services/feed_service.py](services/feed_service.py). The query itself is sound:
it filters `ListeningEvent.listened_at >= cutoff` and dedupes to the most recent
event per friend. The certainty came one line up, at the definition of `cutoff`:
`cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, where
`RECENT_THRESHOLD = timedelta(hours=24)` (line 13). Cross-checking against
`seed_data.py`, whose comments state that events "within the past 30 minutes"
*should* appear and events "1–14 days ago" should *not*, confirmed the intended
window is minutes, not a full day — the constant was simply set far too wide.

**The root cause.** The "recency" cutoff for the live *listening now* feed was
`timedelta(hours=24)`. "Listening now" is meant to be a near-real-time window
(minutes), but a 24-hour threshold admits every friend who listened at any point
in the past day. So a friend who last listened hours ago — or up to a full day
ago — was still reported as currently listening. The filter comparison was
correct; the constant it compared against encoded the wrong window.

**My fix and side-effect check.** Narrowed the constant to
`RECENT_THRESHOLD = timedelta(minutes=30)`, matching the "past 30 minutes"
window the seed data documents. Single-constant change; the query logic is
untouched.
- **Both sides of the boundary:** verified directly that a friend who listened
  **10 minutes ago** *is* included and a friend who listened **10 hours ago** is
  *excluded* — the feed returned exactly `['recent']`.
- **Related feature check:** the other function in the file,
  `get_activity_feed()`, intentionally does **not** use `RECENT_THRESHOLD` (its
  docstring says it is "not filtered by recency" and returns the most recent N
  events), so narrowing the constant does not affect it. No other module
  references `RECENT_THRESHOLD`.
- **Full suite:** 13 passed, 0 failed. No regressions.

---

## Summary

Three distinct bugs, three targeted fixes, each a single line:

| Issue | File | Change | Root cause in one line |
|---|---|---|---|
| #1 streak resets | `services/streak_service.py` | drop `and today.weekday() != 6` | `weekday()==6` (Sunday) wrongly blocked the consecutive-day increment |
| #5 last song missing | `services/playlist_service.py` | `songs[:-1]` → `songs` | slice dropped the highest-position song after a correct query |
| #2 stale "listening now" | `services/feed_service.py` | `hours=24` → `minutes=30` | recency window was a full day instead of a near-real-time window |

