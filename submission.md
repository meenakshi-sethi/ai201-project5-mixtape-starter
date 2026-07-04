# Project 5: Mixtape Bug Hunt — Submission

---

## AI Usage

I used GitHub Copilot (Claude Sonnet 4.6) as an AI assistant throughout this project for codebase orientation and debugging.

**Specific uses:**

1. **Codebase orientation:** I gave the AI the full contents of all service files and `models.py` and asked it to summarize each module's responsibility and trace the data flow for key features (e.g., how rating a song creates a notification, how the playlist song list is built). This saved significant time compared to reading every line cold — though I verified every summary by reading the actual code myself.

2. **Bug diagnosis:** After I had already located the suspicious code in each service file, I asked the AI to explain what `today.weekday()` returns for each day of the week (confirming `6` = Sunday, `0` = Monday), which validated my hypothesis for Issue #1. For Issue #3, I asked it to explain what an `outerjoin` on a many-to-many table does to result count when a row has multiple associated rows — which confirmed the duplicate mechanism.

**Where I verified or overrode AI output:**

- For Issue #2 (feed threshold), the AI initially suggested the bug might be a timezone comparison issue between aware and naive datetimes. I read the code carefully and confirmed SQLAlchemy was passing the filter correctly — the real bug was simply the 24-hour window being too broad for a "listening NOW" feature. I overrode the AI's diagnosis and used the simpler fix.
- For every bug, I read the actual lines of code myself and confirmed the root cause before writing a single fix.

---

## Codebase Map

*(Written before any bug work.)*

### Main Files and Their Roles

| File | Responsibility |
|------|---------------|
| `app.py` | Flask app factory; initializes the app, registers blueprints, and sets up the SQLAlchemy DB connection. |
| `models.py` | Defines all SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. Also defines three association tables: `friendships` (User↔User), `song_tags` (Song↔Tag), and `playlist_entries` (Playlist↔Song with `position`, `added_by`, `added_at` columns). |
| `routes/songs.py` | Endpoints for sharing songs (`POST /songs`), searching (`GET /songs/search`), and rating (`POST /songs/<id>/rate`). |
| `routes/playlists.py` | Endpoints for creating playlists and managing their songs. |
| `routes/users.py` | Endpoints for user profiles, streaks, and notifications. |
| `routes/feed.py` | Endpoints for "Friends Listening Now" and the activity feed. |
| `services/streak_service.py` | Manages listening streak logic: records a `ListeningEvent` and updates `user.listening_streak` based on consecutive calendar days. |
| `services/feed_service.py` | Returns friends who have listened recently (`get_friends_listening_now`) and a broader activity feed (`get_activity_feed`). |
| `services/search_service.py` | Searches `Song` by title or artist (case-insensitive), joining with `song_tags` for tag data. |
| `services/notification_service.py` | Creates `Notification` records when friends interact with a user's shared songs. Also handles rating persistence. |
| `services/playlist_service.py` | Creates playlists and retrieves their songs ordered by `position`. |

### Data Flow: User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id": "...", "score": 4 }`.
2. `routes/songs.py` parses the request and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` validates the score (1–5), fetches the `Song` and `User`, checks for an existing `Rating`, upserts it, commits to DB, then calls `create_notification()` to notify the song's original sharer.
4. `create_notification()` inserts a `Notification` row for the sharer with type `"song_rated"`.
5. The route returns the saved `Rating` as JSON.

### Pattern I Noticed

Every route is a thin layer — it only parses input and formats the response. All business logic lives in `services/`. This makes it straightforward to trace any bug: start at the route, find the service call, read the service function.

---

## Bug Fix Root Cause Analyses

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**

Looked at the streak logic in `streak_service.py` and found the condition that handles the "day increment" path. Traced through what happens when `today` is a Sunday (weekday = 6): the `elif` branch is skipped and the `else` (reset to 1) runs instead. Confirmed by reading `datetime.weekday()` docs: Monday=0, Sunday=6.

**How I found the root cause:**

Went directly to `services/streak_service.py` → `update_listening_streak()`. The branch structure was: `if days_since_last == 0` (no-op), `elif days_since_last == 1 and today.weekday() != 6` (increment), `else` (reset). The extra condition `today.weekday() != 6` stood out immediately — there's no business reason why Sunday should be excluded from streak increments.

**The root cause:**

`datetime.weekday()` returns `6` for Sunday and `0` for Monday. The streak code had an erroneous guard `today.weekday() != 6` on the increment branch, meaning any listening event on a Sunday with exactly 1 day since last listen would fall through to the `else` block and reset the streak to 1 instead of incrementing it.

**Fix and side-effect check:**

Removed the `and today.weekday() != 6` condition, leaving `elif days_since_last == 1:`. Checked the other two branches (`days_since_last == 0` no-op, `else` reset) — neither was affected. All 5 streak tests pass including the new `test_streak_increments_on_sunday` test.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:**

Read `feed_service.py`. `RECENT_THRESHOLD = timedelta(hours=24)` means the cutoff is 24 hours ago — any listening event within the past 24 hours is included. A friend who listened 23 hours ago (yesterday) would appear in "Listening Now," which is misleading.

**How I found the root cause:**

Went directly to `services/feed_service.py`. `get_friends_listening_now()` computes `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and filters `ListeningEvent.listened_at >= cutoff`. The threshold constant `timedelta(hours=24)` was the only variable controlling the window — and 24 hours is far too broad for a feature named "Listening **Now**."

**The root cause:**

`RECENT_THRESHOLD` was set to `timedelta(hours=24)`, making "Friends Listening Now" equivalent to "Friends Who Listened in the Last Day." The feature name implies near-real-time presence, so the threshold should be much shorter.

**Fix and side-effect check:**

Changed `RECENT_THRESHOLD = timedelta(hours=24)` to `timedelta(minutes=30)`. The same constant is only used in `get_friends_listening_now()` — `get_activity_feed()` has no recency filter at all, so it was unaffected. Confirmed by reading both functions after the change.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**

Read `search_service.py`. The query does an `outerjoin` on `song_tags`. If a song has 2 tags, the join produces 2 rows for that song. The query returns all rows without deduplication, so `song.to_dict()` is called once per row — yielding one duplicate per extra tag.

**How I found the root cause:**

`services/search_service.py` → `search_songs()`. The query joins `Song` with `song_tags` via `outerjoin`. An outer join on a one-to-many relationship (one song, multiple tags) multiplies result rows — one row per tag per song. Without `.distinct()`, the query returns `N` copies of a song that has `N` tags.

**The root cause:**

The `outerjoin(song_tags, Song.id == song_tags.c.song_id)` expands result rows for each tag association. A song with 3 tags produces 3 identical rows in the result set. Without `.distinct()`, all 3 are returned and serialized, producing duplicate entries in the API response. Songs with no tags appear once (the `outerjoin` produces a single NULL row).

**Fix and side-effect check:**

Added `.distinct()` before `.all()` in the query chain. This collapses duplicate rows from the join back to one row per unique `Song`. The `tags` field on each song is populated via the `Song.tags` relationship (loaded separately via `lazy="subquery"`), not from the join, so `.distinct()` doesn't affect tag data. All 4 search tests pass.

---

### Issue #4 — I got notified when a friend added my song to a playlist, but not when they rated it

**How I reproduced it:**

Read `notification_service.py`. `add_to_playlist()` calls `create_notification()` after updating the playlist. `rate_song()` saves the rating and commits but has no `create_notification()` call at all — so rating a song never produces a notification.

**How I found the root cause:**

`services/notification_service.py` → compared `add_to_playlist()` and `rate_song()` side by side. `add_to_playlist()` ends with a `create_notification(...)` call guarded by `if song.shared_by != added_by_user_id`. `rate_song()` ends immediately after `db.session.commit()` with a `return rating` — the notification step is simply absent.

**The root cause:**

The `rate_song()` function was implemented to persist the rating correctly but the notification step was never added. The architectural pattern used by `add_to_playlist()` (validate → mutate DB → notify sharer → return) was not followed. The missing step meant rating a song silently succeeded without notifying the original sharer.

**Fix and side-effect check:**

Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, mirroring the pattern in `add_to_playlist()`: only notify when `song.shared_by != user_id` (don't notify someone they rated their own song). The notification type is `"song_rated"` with a human-readable body. `get_notifications()` and `add_to_playlist()` were unaffected — they don't depend on `rate_song()`.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**

Read `playlist_service.py`. `get_playlist_songs()` fetches all songs ordered by position, then returns `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of the list — so a playlist with 4 songs returns only 3.

**How I found the root cause:**

`services/playlist_service.py` → `get_playlist_songs()`. The query and ordering logic looked correct. The return statement `return [song.to_dict() for song in songs[:-1]]` was the immediate tell — `songs[:-1]` is Python for "all elements except the last."

**The root cause:**

`songs[:-1]` was used instead of `songs` in the list comprehension. This off-by-one slicing unconditionally removes the last song from every playlist response. A playlist with 1 song returns an empty list; a playlist with N songs returns N-1 songs.

**Fix and side-effect check:**

Changed `songs[:-1]` to `songs`. Verified the ordering logic (ascending by `position`) and the query filter (`playlist_id`) were both correct and untouched. All 3 playlist tests pass including `test_playlist_returns_all_songs`.

---

## Regression Test

A regression test for Issue #1 (Sunday streak) is in `tests/test_streaks.py` as `test_streak_increments_on_sunday`.

**What it verifies:** A user who listened on Saturday and then listens again on Sunday should have their streak incremented (not reset). It creates a user with `last_listened_at` set to the previous day (a Saturday), then calls `update_listening_streak` with a Sunday datetime and asserts `listening_streak == 2`.

**Why it would have failed against the buggy code:** The original code had `elif days_since_last == 1 and today.weekday() != 6`, which evaluates to `False` on Sunday (`weekday() == 6`), causing the streak to reset to 1 instead of incrementing to 2. The test assertion `assert user.listening_streak == 2` would have failed with `1`.

---

## git log --oneline

```
ddb6942 fix: remove erroneous songs[:-1] slice that dropped last playlist song
8dd8194 fix: send song_rated notification to original sharer in rate_song()
ca681b1 fix: add distinct() to search query to prevent duplicate results for multi-tag songs
b5eeaab fix: reduce Friends Listening Now threshold from 24h to 30min
f37829d fix: remove Sunday exclusion in streak increment logic
```
