# Mixtape — Codebase Map

## AI Usage

I used Claude (Claude Code, in-editor) throughout this project as a debugging
partner rather than to write fixes unprompted — for each bug I described the
user-reported symptom first and asked Claude to locate and explain the cause
before I decided how to fix it.

- **Tracing symptoms to code**: For all three bugs, I gave Claude the
  bug-report-style description (steps taken, expected vs. actual) without
  pointing it at a file first. It read the relevant service file, matched
  specific lines against the function's own docstring, and pointed to the
  exact line/condition responsible (e.g., the extra `today.weekday() != 6`
  clause in `streak_service.py`, the rolling 24h `RECENT_THRESHOLD` cutoff in
  `feed_service.py`, and the unused `outerjoin(song_tags, ...)` fan-out in
  `search_service.py`). This was the most useful part of the collaboration —
  it consistently traced from "user says X happens" to a specific
  comparison/condition rather than stopping at "there's a bug in this area."
- **Reproducing before fixing**: For the streak bug, Claude found that
  `tests/test_streaks.py` already had a test (`test_streak_increments_on_sunday`)
  encoding the correct expected behavior, and running it reproduced the
  failure without me touching the app. For the feed bug, no test existed, so
  I asked Claude for reproduction options; I chose to verify manually against
  the running dev server rather than write a permanent test file, since I
  wanted to see the actual API response rather than trust an assertion.
- **Where I verified myself / where AI needed correction**: When Claude
  started the dev server with `debug=True`, the Werkzeug reloader threw an
  unrelated `RuntimeError` (Flask app not registered with the SQLAlchemy
  instance) on the first couple of requests — a red herring caused by the
  reloader's subprocess restart, not by our fix. I had it restart the server
  without the reloader to confirm the error was environmental, not a real
  regression, before trusting the "empty feed" result as evidence the fix
  worked. I also explicitly declined Claude's first instinct to write a new
  pytest file for the feed bug — I wanted to drive a manual repro against a
  real running server myself rather than have it encode the fix as a test I
  hadn't run through the actual API.
- **What I fixed myself**: In each case, Claude explained the root cause and
  proposed a fix, but I made the actual edits to `streak_service.py` and
  `feed_service.py` myself before asking Claude to help verify them — I only
  had Claude apply the `search_service.py` join removal directly after we'd
  agreed on the fix.
- **Where the explanation was incomplete**: Claude's fix for the search
  duplication bug removed the join but didn't flag that `Tag`/`song_tags`
  became unused imports in `search_service.py` until I asked about it
  directly — it mentioned this only after I looked at the diff, not as part
  of its initial proposed fix.

## Overview

Mixtape is a Flask JSON API (no templates/frontend) for sharing songs, building
collaborative playlists, rating songs, and following friends' listening activity.
It's a single Flask app factory (`app.py`) with routes split into four
blueprints, each of which delegates to a `services/` module for business logic.
There is no `/` route — the app is pure API, so hitting the bare root 404s by
design.

## File map

**`app.py`** — Flask application factory (`create_app`). Configures the
SQLAlchemy DB URI (`DATABASE_URL` env var, defaulting to local SQLite),
initializes the shared `db = SQLAlchemy()` instance, registers the four
blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`,
`/feed`), and calls `db.create_all()` inside an app context so tables exist on
startup. `models.py` imports `db` from here (`from app import db`), so
`app.py` must be importable without pulling in `models.py` at module load
time — that's why the blueprint imports happen *inside* `create_app()` rather
than at the top of the file (avoids a circular import between `app` and
`models`/`routes`/`services`, all of which import `db` from `app`).

**`models.py`** — All SQLAlchemy models, plus three association tables:
- `User` — has a `listening_streak` counter and `last_listened_at` timestamp
  (streak state lives directly on the user row, not derived on the fly), plus
  a **self-referential many-to-many** `friends` relationship via the
  `friendships` table (symmetric, but see quirks below).
- `Song` — carries `shared_by` (FK to `User`) and `share_note` directly on the
  row. There's no separate "Share" model — sharing a song *is* creating a
  `Song` row.
- `Tag` — many-to-many with `Song` via `song_tags`.
- `ListeningEvent` — one row per listen, drives both the streak and the
  activity/"listening now" feeds.
- `Rating` — one row per (user, song) pair, enforced by a
  `UniqueConstraint("user_id", "song_id")`. Rating a song twice **updates**
  the existing row rather than inserting a new one (see `rate_song` below).
- `Playlist` — songs attached via `playlist_entries`, a many-to-many table
  that (unlike `song_tags`/`friendships`) carries extra columns:
  `position`, `added_by`, `added_at`. This is what lets playlists have an
  explicit song order and track who added each song, instead of relying on
  insertion order.
- `Notification` — flat `notification_type` + `body` string, `read` boolean.
  No polymorphic payload — the body is just pre-rendered text baked in at
  creation time (e.g. `"{adder.username} added your song '...' to..."`).
  This means notification text can't be re-localized or restructured later
  without a migration; it's baked into history.

**`routes/`** — four blueprints, one per resource area (`songs.py`,
`playlists.py`, `users.py`, `feed.py`). Every route function does the same
three things and nothing else: (1) parse the request (`request.args` for
query strings, `request.get_json()` for POST bodies), (2) call exactly one
`services/` function, (3) translate the result/exception into a
`jsonify(...)` response with a status code. There is no business logic in
routes — no direct DB queries, no validation beyond "is this field present."

**`services/`** — one module per feature area, holding all business logic and
all direct DB access (`db.session.query`, `db.session.add/commit`):
- `search_service.py` — case-insensitive `ILIKE` search over `Song.title`/
  `Song.artist`.
- `playlist_service.py` — create playlists, fetch a playlist's songs in
  position order, list a user's playlists.
- `notification_service.py` — create/list/mark-read notifications, plus
  `add_to_playlist` (adds a song to a playlist *and* notifies the original
  sharer) and `rate_song` (upsert a rating).
- `streak_service.py` — the day-based listening-streak state machine.
- `feed_service.py` — "friends listening now" (last 24h, deduped to one
  entry per friend) and a general activity feed (last N events, no time
  cutoff).

Every service function that looks up an entity by ID raises `ValueError` if
it's missing, and every route catches that `ValueError` and turns it into a
`404` (or `400` for the create/write endpoints). This is the app's
error-handling convention end to end — no custom exception classes, just
`ValueError` + a try/except in the route.

## Data flow: adding a song to a playlist (triggers a notification)

This is the clearest example of the routes → services → models pipeline:

1. **`POST /playlists/<playlist_id>/songs`** hits `add_song()` in
   [routes/playlists.py:43-54](routes/playlists.py#L43-L54). The route parses
   `song_id` and `added_by` out of the JSON body and 400s if either is
   missing — that's the *only* validation done at this layer.
2. The route calls `notification_service.add_to_playlist(playlist_id,
   song_id, added_by)` — note this lives in `notification_service.py`, not
   `playlist_service.py`, because the point of the function is really
   "mutate the playlist *and* fire the notification," not just playlist CRUD.
3. Inside `add_to_playlist` ([services/notification_service.py:35-70](services/notification_service.py#L35-L70)):
   - Loads `Song`, `User` (the adder), and `Playlist` by ID, raising
     `ValueError` (→ 400 at the route) if any is missing.
   - If the song isn't already in `playlist.songs`, appends it via the
     ORM-managed `playlist_entries` many-to-many relationship and commits.
   - **Only if** `song.shared_by != added_by_user_id` (i.e. you didn't add
     your own shared song to the playlist), it calls `create_notification(...)`
     targeting `song.shared_by` — the person who originally shared the song,
     not the playlist owner.
4. `create_notification` ([services/notification_service.py:13-32](services/notification_service.py#L13-L32))
   just builds a `Notification` row with a pre-formatted `body` string and
   commits it.
5. The route returns `{"message": "Song added to playlist"}, 201`.

The recipient later discovers this via **`GET /users/<user_id>/notifications`**
(`routes/users.py` → `notification_service.get_notifications`), which queries
`Notification` filtered by `user_id` and optionally `read=False`, ordered
newest-first — a completely separate request/response cycle from the one that
created it. Nothing pushes the notification to the client; it's pull-only.

## Patterns noticed

- **Strict three-layer separation**: routes parse/format HTTP, services hold
  all logic and DB access, models hold schema + `to_dict()` serialization.
  No route ever imports `db.session` directly to run a query — always through
  a service function.
- **`ValueError` as the one error-handling primitive.** Every "not found" or
  "invalid input" case raises `ValueError` from a service and is caught at
  the route boundary. There's no shared exception hierarchy
  (e.g. no `NotFoundError` vs `ValidationError`), so 400 vs 404 is decided
  per-route by which try/except block wraps the call, not by exception type.
- **State is denormalized onto `User` for cheap reads.** `listening_streak`
  and `last_listened_at` live on `User` rather than being computed from
  `ListeningEvent` history on every request — trades write-time complexity
  (the streak state machine in `streak_service.py`) for O(1) reads.
- **Every `to_dict()` is hand-written per model**, not a generic serializer —
  this is why `Notification.to_dict()` renames `notification_type` to `type`
  and `Song.to_dict()` flattens `tags` into a list of names instead of nested
  dicts.
- **UUID primary keys everywhere** (`generate_uuid()` in `models.py`), not
  auto-increment ints — consistent across all six models.

## Quirks / things worth flagging (found while reading, not yet fixed)

- **`get_playlist_songs` drops the last song.** [services/playlist_service.py:66](services/playlist_service.py#L66)
  returns `songs[:-1]` instead of `songs` — this silently omits the
  most-recently-positioned song from every playlist's song list. Looks like
  an off-by-one, not intentional (the docstring says "returns all songs").
- **`add_to_playlist` appends to `playlist.songs` without setting `position`
  or `added_by`.** [models.py:31-38](models.py#L31-L38) defines those columns
  on `playlist_entries` as `nullable=False` with no default (`added_at` has a
  default, the other two don't). Appending through the plain ORM
  `secondary=` relationship in [services/notification_service.py:61](services/notification_service.py#L61)
  doesn't populate them, so this insert should be violating a NOT NULL
  constraint — worth verifying this path actually works end-to-end.
- **Streak logic special-cases Sundays.** [services/streak_service.py:73](services/streak_service.py#L73)
  only increments the streak on a consecutive day if `today.weekday() != 6`
  (Sunday) — listening on a Sunday after listening Saturday resets the streak
  to 1 instead of incrementing it. This isn't mentioned in the function's own
  docstring rules and looks unintentional.
- **`friendships` is modeled as symmetric but stored as directed rows.** The
  table has no `UniqueConstraint` or code visible here that inserts both
  `(a,b)` and `(b,a)` when a friendship is formed — if only one row is ever
  inserted, `get_friends_listening_now`/`get_activity_feed` would only see
  the friendship from one side.
- **`rate_song` never notifies anyone**, despite `notification_service.py`'s
  module docstring saying notifications fire "when friends interact with a
  user's shared songs." Only `add_to_playlist` actually creates a
  notification — rating a song updates the `Rating` row silently.

## Root Cause Analysis

### Bug 1: Listening streak resets to 1 every Sunday

**How I reproduced it:** A user reported that after listening every day for
weeks (streak at 12 on Saturday), listening again Sunday morning dropped the
streak to 1, and listening Monday brought it back to 2 (i.e., it started
counting fresh from Sunday). To confirm before touching any code, I ran the
existing test suite in `tests/test_streaks.py` — specifically
`test_streak_increments_on_sunday`, which calls
`update_listening_streak(u, saturday)` followed by
`update_listening_streak(u, sunday)` on consecutive calendar dates
(2024-06-15 → 2024-06-16) and asserts the streak goes from 1 to 2. Running
`pytest tests/test_streaks.py::test_streak_increments_on_sunday` failed with
`assert 1 == 2`, reproducing the exact user-reported behavior (a consecutive
day being treated as a skipped day) without needing to touch the app through
the API.

**How I found the root cause:** I opened `services/streak_service.py` and
read `update_listening_streak`, the only function that mutates
`listening_streak`. The docstring lists the rules as: no prior listen → 1,
same day → no change, consecutive day → increment, gap → reset to 1. Reading
the actual `if/elif/else` block against those stated rules, line 73 read
`elif days_since_last == 1 and today.weekday() != 6:` — an extra condition
not mentioned anywhere in the docstring. That was the moment I was confident
this was the exact cause, not just a suspicious area: `days_since_last == 1`
already fully captures "consecutive day," and the `weekday() != 6` clause
only changes behavior on one specific day of the week (`weekday()` returns
`6` for Sunday), which lines up precisely with the user's report that this
only ever happened on Sundays.

**The root cause:** The increment branch required two conditions
(`days_since_last == 1 and today.weekday() != 6`) instead of one. Since
Python's `datetime.weekday()` returns `6` for Sunday, any listen that landed
on a Sunday — even a genuinely consecutive one — made the second condition
`False`, which sent execution into the `else` branch and reset
`listening_streak` to `1`, regardless of how long the prior streak was. On
every other day of the week the clause was `True` and had no effect, which is
why the bug only ever showed up on Sundays and looked intermittent.

**My fix and side-effect check:** I removed the `and today.weekday() != 6`
clause, leaving `elif days_since_last == 1:` as the sole condition for
incrementing, matching the docstring's stated rule with no day-of-week
exception. I re-ran the full `tests/test_streaks.py` suite (not just the
Sunday test) to check for regressions in the other three rules: new user
starts at 1, consecutive-day increment on non-Sunday days, same-day listen is
a no-op, and a skipped day resets to 1. All 5 tests passed, confirming the
fix resolves the Sunday case without changing behavior for any other day or
scenario.

### Bug 2: "Friends Listening Now" shows listens from the previous evening all morning

**How I reproduced it:** A user reported that a friend's 11pm listen from the
night before was still showing up as "listening now" at 9am the next morning,
even though the friend hadn't opened the app since. To confirm the behavior
before changing any code, I seeded data directly against the running app's
dev database (there's no API to create users/friendships/songs, so I used a
one-off script against the same `SQLAlchemy` app/db instance the server
uses): two friended users, a song, and a `ListeningEvent` for the friend with
`listened_at` set to 11pm the previous UTC calendar day — about 3h42m before
"now." Hitting `GET /feed/<user_id>/listening-now` against the running server
returned that friend in the feed, confirming a listen from a previous
calendar day was still being treated as "now."

**How I found the root cause:** I opened `services/feed_service.py` and read
`get_friends_listening_now`, the only function backing that endpoint
(traced from `routes/feed.py` → `services/feed_service.py`). The docstring
says it should show what friends "have played today." Scanning the function
body, the only recency filter is `cutoff = datetime.now(timezone.utc) -
RECENT_THRESHOLD` at the top, where `RECENT_THRESHOLD = timedelta(hours=24)`
is a module-level constant. That was the moment I was confident this was the
actual cause, not just a suspicious area: there is no `.date()` comparison
or calendar-day boundary anywhere in the function — the entire "today" concept
is implemented as "within the last 24 hours," which are not the same thing
whenever the request happens at a different clock time than the original
listen.

**The root cause:** `get_friends_listening_now` computed its recency cutoff
as a rolling 24-hour window (`now - timedelta(hours=24)`) instead of the
start of the current calendar day. A `ListeningEvent.listened_at >= cutoff`
check against that rolling window keeps any event less than 24 hours old
regardless of whether it happened yesterday or today. An 11pm listen is
still less than 24 hours old at 9am the next morning (only ~10 hours have
passed), so it stayed in the feed — and would keep showing up right up until
24 full hours had elapsed (i.e., ~11pm the *next* night), matching the user's
observation that stale entries "hang around... until the same time the next
day."

**My fix and side-effect check:** I changed the cutoff calculation to the
start of the current UTC calendar day instead of a rolling window:
`now = datetime.now(timezone.utc)` then `cutoff =
datetime.combine(now.date(), datetime.min.time(), tzinfo=timezone.utc)`, and
left the rest of the filtering/dedup logic untouched. This means any event
before midnight UTC today is excluded, however recent it was in wall-clock
terms, and anything from today (even from just after midnight) is included.
I verified this against the running server: the previously-seeded
"yesterday 11pm" event now returns `{"feed": [], "count": 0}`, and after
adding a second event for the same friend at 1am *today*, the feed correctly
returned that friend (`count: 1`) — confirming the fix excludes yesterday's
listens without breaking same-day listens. I did not change
`get_activity_feed`, which is documented as intentionally unfiltered by
recency, so it's unaffected by this fix. Note this fix defines "today" as
the current UTC calendar day — there's no per-user timezone field on `User`
anywhere in this codebase, so a friend in a different timezone would still
have their "today" boundary computed in UTC, not their local midnight.

### Bug 3: Song search returns duplicate results for songs with multiple tags

**How I reproduced it:** A user reported that searching `GET
/songs/search?q=Anthem` returned "Crown Heights Anthem" three times as
identical entries, while other matching songs in the same result set only
appeared once. To confirm before touching any code, I read `Song.to_dict()`
in `models.py`, which includes a `tags` list per song — this pointed at tags
being the differentiating factor between the songs that duplicated and the
ones that didn't, since "appears more times" tracks with "has more tags"
rather than with anything about the song's own data being different.

**How I found the root cause:** I opened `services/search_service.py` and
read `search_songs`, the only function backing `/songs/search`. The query
was:
```python
db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(db.or_(Song.title.ilike(...), Song.artist.ilike(...)))
    .all()
```
Nothing in the `filter()` or the returned columns references `Tag` or
`song_tags` at all — the join isn't used to filter or select anything. I then
checked `models.py` and found `Song.tags = db.relationship("Tag",
secondary=song_tags, lazy="subquery")` and `to_dict()` reading `self.tags`
directly ([models.py:90,102](models.py#L90-L102)) — tags are already loaded
through the ORM relationship, independent of the search query. That was the
moment I was confident this was the exact cause: the join in `search_songs`
serves no purpose, and a SQL join against a many-to-many association table
fans out one result row per matching row on the other side — a song with 3
tags produces 3 joined rows, and querying a plain `Song` entity without
`.distinct()` returns that song 3 times in the list, once per fanned-out row.

**The root cause:** `search_songs` joined `Song` to `song_tags` via
`.outerjoin(song_tags, Song.id == song_tags.c.song_id)` without ever
filtering or selecting on the tag side, so the join contributed nothing to
the query except its side effect. Because a join multiplies rows by the
number of matches on the joined table, a song with N associated tags produced N
identical `Song` rows in the SQL result, which SQLAlchemy hydrated into N
separate (but identical) entries in the returned list. Songs with 0 or 1
tags only ever produced 1 row, so they appeared to search correctly, which
is why the bug looked inconsistent — it silently scaled with each song's tag
count rather than being visibly broken for every result.

**My fix and side-effect check:** I removed the `.outerjoin(song_tags, ...)`
call entirely, since it wasn't feeding any filter or column and tags are
already populated independently via the `Song.tags` relationship inside
`to_dict()`. The query now filters directly on `Song.title`/`Song.artist`
with no join, so it returns exactly one row per matching song regardless of
tag count. I checked that `to_dict()`'s `tags` field still renders correctly
after the change (it does, since it was never sourced from this query's join
in the first place), and confirmed the `Tag`/`song_tags` imports at the top
of the file are now unused — worth a follow-up cleanup, but left as-is since
removing unrelated imports wasn't part of this fix.
