# Mixtape — Codebase Map (Milestone 1)

## Main files and what they actually do

**`app.py`** is just the Flask app factory. `create_app()` builds the `db` instance, points `SQLALCHEMY_DATABASE_URI` at sqlite by default, registers the four blueprints (songs, playlists, users, feed), and calls `db.create_all()`. That's it — this is the one place anything gets wired together, nothing else touches Flask directly.

**`models.py`** has all seven models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. There are also three association tables for the many-to-many relationships — `friendships`, `song_tags`, and `playlist_entries`. The last one isn't a plain join table though, it actually carries `position`, `added_by`, and `added_at` columns, so playlist ordering and who-added-what lives at the join-table level instead of on `Song` or `Playlist` themselves. Every model also has its own `to_dict()`, which is how everything gets serialized — there's no separate schema layer, routes and services just call `.to_dict()` straight up. IDs are all UUID strings from a shared `generate_uuid()` helper, and timestamps default to `datetime.now(timezone.utc)`.

**`routes/songs.py`** — `search()` calls `search_service.search_songs()`, `get_song_detail()` calls `search_service.get_song()`, `rate()` calls `notification_service.rate_song()`, and `listen()` calls `streak_service.record_listening_event()`.

**`routes/playlists.py`** — `create()` calls `playlist_service.create_playlist()`, `get_detail()` calls `playlist_service.get_playlist()`, `get_songs()` calls `playlist_service.get_playlist_songs()`, and `add_song()` calls `notification_service.add_to_playlist()`.

**`routes/users.py`** — `get_user()` is actually the one exception to the pattern, it queries `db.session.get(User, user_id)` directly instead of going through a service. `streak()` calls `streak_service.get_streak()`, and the two notification routes call `notification_service.get_notifications()` / `notification_service.mark_as_read()`.

**`routes/feed.py`** — `listening_now()` calls `feed_service.get_friends_listening_now()`, `activity()` calls `feed_service.get_activity_feed()`.

**`services/streak_service.py`** is where the streak rules actually live. `record_listening_event()` logs a `ListeningEvent` and calls `update_listening_streak()`, which is where the day-comparison logic sits: first listen ever → streak = 1, same day → no change, exactly one day gap → increment, more than one day → reset to 1. I noticed it also has a special case that skips the increment specifically on Sundays (`today.weekday() != 6`), which seems like it's tied to issue #1.

**`services/feed_service.py`** — `get_friends_listening_now()` pulls the user's friends, filters `ListeningEvent` rows in the last `RECENT_THRESHOLD` (a flat `timedelta(hours=24)`, not a calendar-day cutoff), sorts by most recent, then de-dupes in a Python loop so each friend only shows their latest song. `get_activity_feed()` is the more general version — no time filter at all, just the most recent 20 events across friends, no de-dupe.

**`services/search_service.py`** — `search_songs()` does an outer join from `Song` to the `song_tags` table and filters on title/artist with `ILIKE`, returning `to_dict()` for every row that comes back. `get_song()` is just a lookup by ID.

**`services/notification_service.py`** does more than notifications, honestly. `add_to_playlist()` appends the song to the playlist (writes into `playlist_entries`) and, if the person adding it isn't the original sharer, calls `create_notification()`. `rate_song()` upserts a `Rating` (there's a unique constraint on user+song so it updates in place if you've already rated it) — but it never calls `create_notification()` anywhere, which lines up with issue #4. There's also a locally-imported `get_playlist_songs` inside `add_to_playlist()` that never actually gets used — looks like leftover/dead code.

**`services/playlist_service.py`** — `create_playlist()` is a simple insert. `get_playlist_songs()` orders songs by `position` through the `playlist_entries` join, but then returns `songs[:-1]` — it slices off the last item before returning, which is almost certainly why issue #5 happens. `get_playlist()` returns metadata only, and `get_user_playlists()` returns everything a user created.

---

## Tracing a feature end to end: adding a song to a playlist

I picked this one because it's the cleanest example of a route triggering a mutation *and* a notification in the same call:

1. `POST /playlists/<playlist_id>/songs` hits `add_song()` in `routes/playlists.py`.
2. That route pulls `song_id` and `added_by` out of the JSON body, checks they're both present, and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. Inside `add_to_playlist()`, it loads the `Song`, `User` (adder), and `Playlist` by ID, raising `ValueError` if any of them don't exist (which the route turns into a 400). If the song isn't already on the playlist, it appends it — that's a new row in `playlist_entries` — and commits.
4. Then it checks if `song.shared_by` is different from whoever just added it. If so, it calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`.
5. `create_notification()` builds the `Notification` row and commits it.
6. The route just returns a success message — the notification only actually surfaces later, when the recipient hits `GET /users/<user_id>/notifications`.

So the mutation and the notification happen back-to-back in the same function call, in the same request. There's no queue or event system in between — it's all synchronous.

---

## Patterns I noticed

- Routes are really thin. Basically every route just validates the request body, calls one service function, and catches `ValueError` to turn it into a 400 or 404. All the actual logic lives in `services/`.
- `ValueError` is the app's one and only way of signaling "not found" or "bad input" — services never return `None` or use custom exceptions, they just raise `ValueError` with a message, and the route decides what HTTP status that becomes.
- Notifications aren't a separate system — they're just a side effect tacked onto whatever mutation function triggered them. That's why `add_to_playlist()` sends a notification but `rate_song()` doesn't: there's no shared "always notify on this kind of action" mechanism, each function has to remember to call `create_notification()` itself.
- Every model serializes itself via `to_dict()` instead of using a schema library.
- The `playlist_entries` join table carries real data (`position`, `added_by`, `added_at`), not just foreign keys — so ordering and provenance are tracked at the join level.
- IDs and timestamps are consistent everywhere — UUID strings via `generate_uuid()`, timestamps via `datetime.now(timezone.utc)`. No integer PKs or naive datetimes anywhere.
- Found one bit of dead code: `add_to_playlist()` imports `get_playlist_songs` but never calls it.

---

## Root Cause Analysis

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py` before changing anything. `test_playlist_returns_all_songs` seeds a 5-song playlist and expects 5 back — it failed with only 4 returned (the test even has a comment calling this out).

**How I found the root cause:** Traced from the failing test into `get_playlist_songs()` in `services/playlist_service.py`. The query (join through `playlist_entries`, ordered by `position`) was fine — the problem was the last line, `return [song.to_dict() for song in songs[:-1]]`.

**The root cause:** `songs[:-1]` drops the last element of the query result before serializing, so the final song in the playlist was always silently excluded.

**My fix:** Removed the slice — `[song.to_dict() for song in songs[:-1]]` → `[song.to_dict() for song in songs]`.

**Side-effect check:** Full suite passes for `test_playlists.py` (all 3 tests, including empty-playlist and ordering cases). No other test file touches this function, and `test_search.py`/`test_streaks.py` are unaffected.

### Issue #1 — Listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py` before changing anything. `test_streak_increments_on_sunday` simulates listening on Saturday then Sunday and expects the streak to go from 1 to 2 — it failed, staying at 1.

**How I found the root cause:** Traced from the failing test into `update_listening_streak()` in `services/streak_service.py`. The increment branch read `elif days_since_last == 1 and today.weekday() != 6`, so a one-day gap only incremented the streak on every day except Sunday.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday, so the extra `and today.weekday() != 6` condition specifically excluded Sundays from the normal "consecutive day" increment path, sending them to the `else` branch instead, which resets the streak to 1.

**My fix:** Removed the weekday condition — `elif days_since_last == 1 and today.weekday() != 6` → `elif days_since_last == 1`, so any one-day gap increments regardless of which day of the week it is.

**Side-effect check:** Full suite passes, all 5 tests in `test_streaks.py` including same-day (no double-count) and skipped-day (reset) cases, so the fix didn't loosen the other boundary conditions. `test_playlists.py` and `test_search.py` are unaffected.

### Issue #4 — Notified for playlist adds but not for ratings

**How I reproduced it:** Called `rate_song()` directly in a Python shell with a rater who wasn't the song's sharer, then checked `get_notifications()` for the sharer — it came back empty, confirming ratings never generate a notification while `add_to_playlist()` does.

**How I found the root cause:** Compared `rate_song()` to `add_to_playlist()` in `services/notification_service.py` line-by-line, since both live in the same file and only one of them calls `create_notification()`. `add_to_playlist()` checks `if song.shared_by != added_by_user_id` and notifies; `rate_song()` had no equivalent call at all — it saves the `Rating` and returns.

**The root cause:** `rate_song()` was simply missing the notification step. There's no shared "notify on interaction" mechanism in this codebase — each mutation function has to remember to call `create_notification()` itself, and whoever wrote `rate_song()` never added it.

**My fix:** Added a `create_notification()` call after the rating commits, guarded by `if song.shared_by != user_id` (mirroring `add_to_playlist()`'s self-action guard so you don't get notified for rating your own song), with `notification_type="song_rated"` and a body referencing the rater's username, the song title, and the score.

**Side-effect check:** Manually verified three cases in a Python shell: (1) another user rating a song notifies the sharer, (2) the sharer rating their own song does not self-notify, (3) re-rating the same song (the existing-rating update path) still notifies correctly. Ran the full `pytest tests/` suite afterward — all 13 existing tests still pass, since none of them touch notifications.
