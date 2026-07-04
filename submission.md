# AI Usage

I used Claude throughout this assignment mainly as a tracing and debugging partner, not as a code generator — I didn't ask it to write the fixes for me, but I did lean on it heavily to help me read and navigate an unfamiliar codebase, interpret errors, and think through hypotheses before I tested them myself.

## What I asked it to explain or summarize:


Basic Flask/Python syntax I wasn't sure about (e.g., what request.args.get("q", "") does, what the "q" parameter actually represents).
Environment errors — a ModuleNotFoundError for flask_sqlalchemy that turned out to be a venv/system-Python mismatch, and later a ModuleNotFoundError: No module named 'app' caused by running a test file directly instead of as a module.
What a 404 on / actually meant (i.e., that the server was running fine and the routing was correct — there just wasn't a homepage route defined).


## What it helped me trace:


For the streak bug (Issue #1), I had it walk with me from models.py → the /streak route → get_streak() → record_listening_event() → update_listening_streak(), confirming at each step whether that function read or wrote the streak value. That helped me rule out the read path early instead of guessing randomly.
For the notification bug (Issue #4), it helped me compare add_to_playlist() and rate_song() side by side once I pasted both, which is what made the missing create_notification() call in rate_song() obvious.
For the playlist bug (Issue #5), I applied the same "run the existing test file first" approach it had suggested for the streak bug, and used that pattern on my own to find and confirm the songs[:-1] slicing bug in get_playlist_songs().


## Where I had to verify things myself or the AI was wrong/incomplete:


On the duplicate search bug (Issue #3), the AI's working theory for a while was that the outerjoin with song_tags in search_songs() was fanning out into duplicate rows for songs with 3+ tags. I tested that hypothesis directly against several multi-tag seeded songs ("Harlem Renaissance," "Crown Heights Anthem," etc.) and never once got a duplicate — count was always 1. The AI's theory didn't hold up under actual testing, and we never landed on a confirmed root cause for this one; I did not end up fixing Issue #3.
On the notification bug, there was a confusing stretch where I had already added the missing create_notification() block myself, but when I pasted the code back to ask a follow-up question, the AI treated it as the original (unfixed) code and spent a few turns proposing unrelated theories (that I was accidentally rating my own song, then that Flask's debug-mode-off meant the server was running stale code) before I clarified that I'd already made the edit myself. Those theories weren't wrong in general — they were reasonable things to rule out — but they weren't actually relevant to what I was asking at that point, and I had to redirect it back on track.
I had to catch several small but real mistakes in commands the AI gave me: it once told me to literally type <placeholder> angle brackets into a curl command instead of substituting a real value; I also caught myself (not the AI) reusing the same UUID for both a playlist ID and a song ID in one command, which the AI then correctly flagged. It also guessed wrong field names once (user_id vs. the actual added_by expected by the playlist-add route) — that one I only resolved by reading the actual error message myself and reporting it back.
The AI never independently found the Issue #5 root cause — I found and fixed that one on my own by recognizing the same "off-by-one slice" pattern from the streak fix and applying it to playlist_service.py, then told the AI after the fact so it could help me write it up.


Overall, AI was most useful for structured tracing (route → service → model), interpreting stack traces/errors, and helping me organize what I'd already found into a clear write-up. It was least reliable when it was pattern-matching on hypotheses without direct evidence (the tag-join theory for Issue #3), and I made sure not to accept those theories as confirmed until I'd tested them myself against the actual running app or test suite.

# Architecture Overview

## Main files and what they do

**`models.py`**
Defines the SQLAlchemy models and association tables:
- `User` — has `username`, `email`, `listening_streak`, `last_listened_at`, plus a `listening_events` relationship.
- `Song` — has `title`, `artist`, `album`, `genre`, `shared_by` (FK to `User`), `shared_at`, `share_note`, plus relationships to `ratings`, `listening_events`, and `tags`.
- `Rating` — stores a user's score (1–5) for a song. This is a separate model, not a column on `Song` — a song can have many ratings, one per user.
- `ListeningEvent` — records each time a user listens to a song (`user_id`, `song_id`, `listened_at`). Streaks are derived from these plus the `User.last_listened_at`/`listening_streak` fields, not recalculated from the full event history each time.
- `Notification` — `user_id` (recipient), `notification_type`, `body`, `read`, `created_at`.
- `Tag` — a tag name, linked to songs many-to-many via the `song_tags` association table.
- `Playlist` — a named collection of songs.
- `playlist_entries` — the join table between `Playlist` and `Song`. It's not just a plain many-to-many table: it adds `position` (explicit ordering, not insertion order), `added_by` (who added the song), and `added_at`. This is why songs in a playlist have a defined order and a traceable "who added this" history.

**`routes/`**
Four blueprints, each mounted with its own prefix in `app.py`:
- `songs.py` (`/songs`) — search, song detail, rate, listen.
- `playlists.py` (`/playlists`) — create playlist, get playlist, get/add playlist songs.
- `users.py` (`/users`) — get user, get streak, get notifications, mark notification read.
- `feed.py` (`/feed`) — listening-now and activity feeds.

**`services/`**
Business logic lives here, one module per feature area:
- `feed_service.py` — powers the `/feed` routes (listening-now, activity).
- `notification_service.py` — `create_notification()`, plus, somewhat less obviously, `rate_song()` and `add_to_playlist()`. Both of those live here (rather than in a "rating" or "playlist" service) specifically because their core responsibility, beyond saving the rating/playlist entry itself, is triggering a notification — so the module is organized around the notification side-effect, not just the entity being modified.
- `playlist_service.py` — `get_playlist_songs()` and related playlist-read logic.
- `search_service.py` — `search_songs()`.
- `streak_service.py` — `update_listening_streak()`, `get_streak()`, `record_listening_event()`.

**`seed_data.py`**
Populates a dev database with realistic test data: 5 users with friendships, 25 songs with varying tag counts, 3 playlists, two weeks of listening events, some pre-set streaks, and a few notifications — deliberately shaped to exercise edge cases (e.g., songs with 3+ tags, users who listened "today" vs. "yesterday").

**`tests/`**
Pytest files, one per feature area, each exercising the corresponding service module directly (not just through the HTTP routes):
- `test_streaks.py` — covers `streak_service.py`. Includes the same-day no-change case, the multi-day-gap reset case, the first-time-listen case, and the Saturday→Sunday case that caught the Issue #1 bug.
- `test_playlists.py` — covers `playlist_service.py`, including `test_playlist_returns_all_songs`, which caught the Issue #5 off-by-one (last song missing from the returned list).
- `test_search.py` — covers `search_service.py`, including a test asserting a multi-tag song appears exactly once in search results (`test_search_no_duplicates_multi_tag_song`), aimed at Issue #3 — though as noted below, this test currently passes against the seeded data rather than reproducing the reported bug.

## Data flow — user rates a song

1. `POST /songs/<song_id>/rate` in `routes/songs.py` parses the JSON body (`user_id`, `score`) and calls `rate_song(user_id, song_id, score)` in `notification_service.py`.
2. `rate_song()` validates the score (1–5), looks up the `Song` and `User`, then checks whether a `Rating` already exists for that user/song pair — if so it updates the score in place, otherwise it creates a new `Rating` row. It commits this first.
3. After the rating is saved, `rate_song()` checks `if song.shared_by != user_id` — i.e., don't notify someone about their own action — and if the rater isn't the song's original sharer, it calls `create_notification(user_id=song.shared_by, notification_type="song_rated", body=...)`, also in `notification_service.py`.
4. `create_notification()` builds a `Notification` row and commits it directly.
5. The song's original sharer can later see it via `GET /users/<user_id>/notifications`, which calls `get_notifications()` — a straight read of `Notification` rows filtered by `user_id` (and optionally `unread_only`), ordered by most recent first. No recalculation happens on read; by the time it's queried, the notification either exists or doesn't.

The parallel flow — adding a song to a playlist — follows the identical shape and lives in the same file: `POST /playlists/<playlist_id>/songs` → `add_to_playlist()` (also in `notification_service.py`) → same `if song.shared_by != added_by_user_id` guard → same `create_notification()` call, just with `notification_type="song_added_to_playlist"`. Both actions converge on the same notification primitive; the only difference is which route/function triggers it and what message it generates. This is also exactly why Issue #4 was possible: `add_to_playlist()` and `rate_song()` sit side by side in the same file, following the same pattern, but `rate_song()` simply never had its notification block written in.

## Patterns noticed

- **Routes are thin, services hold the logic.** Every route function parses the request (JSON body or query params), calls exactly one service function, and formats the response (`jsonify(...)`, with a try/except ValueError → 404 or 400). None of the actual business rules — score validation, streak math, notification conditions — live in `routes/`.
- **A single shared `create_notification()` helper, called from multiple features, all within one file.** Rather than each feature building its own `Notification` row inline, both the rating flow and the playlist-add flow funnel through `create_notification()` in `notification_service.py`. This makes the notification system consistent, but it also means a feature that *forgets* to call it (as `rate_song()` originally did) fails silently — there's no shared enforcement that every "friend interacted with your content" action must notify. The service module is organized around the side-effect (notifying) rather than the entity (rating vs. playlist), which is a slightly unusual but deliberate grouping worth noting when navigating the codebase.
- **Derived state is stored, not recomputed.** `listening_streak` and `last_listened_at` are columns on `User`, updated incrementally on each listen event, rather than being calculated fresh from the full `ListeningEvent` history on every read. This is efficient but means any bug in the update logic (e.g., the weekday-based off-by-one we found) silently corrupts stored state that persists until the next correct update.
- **Ordering is explicit where it matters.** `playlist_entries.position` exists specifically so playlist song order doesn't depend on insertion order or primary key order — queries explicitly `order_by(asc(position))`.
- **IDs are UUIDs everywhere**, generated via a shared `generate_uuid()` default, not auto-incrementing integers — relevant when writing manual test/reproduction scripts, since IDs can't be guessed and must be looked up from the database.

# Root Cause Analysis

## Issue #1: Streak doesn't increment on Sundays

**How I reproduced it**

Ran the existing test suite (`pytest tests/test_streaks.py -v`). The test `test_streak_increments_on_sunday` simulated a user listening on Saturday (`datetime(2024, 6, 15)`, `weekday() == 5`) followed by listening the next day, Sunday (`datetime(2024, 6, 16)`, `weekday() == 6`), calling `update_listening_streak()` directly for each. Expected the streak to go from 1 to 2, since exactly one day had passed. Instead, the assertion `assert u.listening_streak == 2` failed with `assert 1 == 2` — the streak stayed at 1 instead of incrementing.

**How I found the root cause**

Started at `models.py` to understand `User.listening_streak` and `last_listened_at`. Traced `GET /users/<id>/streak` → `get_streak()`, which just reads the stored value with no calculation — ruling out the read path. Traced `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()`, the only place `listening_streak` is written. Tried manually reproducing with seeded users (one who listened "yesterday," one who listened "today") — both updated correctly, which didn't match the bug description. Ran the project's existing pytest file instead of continuing to hand-construct dates, and `test_streak_increments_on_sunday` failed immediately, isolating the exact line responsible.

**The root cause**

In `update_listening_streak()`, the increment condition was:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

Python's `datetime.weekday()` returns `6` for Sunday. This condition explicitly excludes Sundays from the increment branch — so any case where exactly one day had passed (a legitimate streak continuation) but the current day happened to be a Sunday fell through to the `else` branch and incorrectly reset the streak to 1, even though the user hadn't broken their streak at all.

**My fix and side-effect check**

Removed the `weekday()` check from the condition entirely, since the streak rule only cares about the number of days elapsed (`days_since_last == 1`), not which day of the week it is — there was no legitimate reason for Sunday to be treated differently. After the fix, the increment fires purely based on `days_since_last == 1`. Re-ran `test_streak_increments_on_sunday` and confirmed it passed. Also re-ran the full test file (`pytest tests/test_streaks.py -v`) to confirm the other passing tests (same-day no-change, multi-day-gap reset, first-time listen) still passed, since a weekday-based fix risked only patching one specific day rather than the general case.

---

## Issue #4: No notification when a friend rates your song

**How I reproduced it**

Used seeded data to identify a song's `shared_by` user (`70c6fd7d-1a69-4928-b727-fea9d03d7156`). Called `POST /songs/<song_id>/rate` with a *different* user's ID (`119b083a-b470-40d6-bd73-2e03ff4271f5`) as `user_id`, confirming the rating itself was created successfully (`201` response with a valid `Rating` object). Then checked the song owner's notifications via `GET /users/70c6fd7d-1a69-4928-b727-fea9d03d7156/notifications` and found no `song_rated` notification, despite a different user having rated their song. Cross-checked by directly querying the database for any `Notification` rows with `notification_type='song_rated'` — there were zero, confirming the notification was never created, not just filtered out on read. As a control, confirmed the parallel action — `POST /playlists/<id>/songs` (adding a friend's song to a playlist) — *did* correctly produce a notification for the same kind of relationship (acting user ≠ song owner), showing the notification system itself worked, but only for that one action.

**How I found the root cause**

Compared `add_to_playlist()` and `rate_song()` in the same service file, since both are meant to notify the original song-sharer when someone else interacts with their song. `add_to_playlist()` contained:

```python
if song.shared_by != added_by_user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_added_to_playlist",
        body=f"{adder.username} added your song '{song.title}' to the playlist '{playlist.name}'.",
    )
```

`rate_song()` had no equivalent block at all — it saved/updated the `Rating` and committed, then returned, with nothing calling `create_notification()`. Confirmed `create_notification()` itself worked correctly by calling it directly in isolation (bypassing both routes), which successfully created and persisted a notification. This ruled out a bug in notification creation/persistence and confirmed the issue was specific to `rate_song()` never invoking it.

**The root cause**

`rate_song()` was simply missing the notification step. Unlike `add_to_playlist()`, which notifies the song's original sharer whenever someone else acts on their song, `rate_song()` saved the `Rating` record and returned without ever calling `create_notification()`. There was no faulty condition or comparison — the call was absent entirely, so a friend rating your song never generated any record for you to see, regardless of who rated it.

**My fix and side-effect check**

Added the missing notification block to `rate_song()`, mirroring the pattern already used in `add_to_playlist()`:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )
```

placed after the rating is saved and committed. Re-tested by rating the same song again as the different user and confirmed a `song_rated` notification now appeared for the song's owner via `GET /users/<owner_id>/notifications`. Also re-tested rating one's own shared song to confirm the `song.shared_by != user_id` guard still correctly suppresses self-notifications, and re-ran the playlist-add flow to confirm it was unaffected by the change.

---

## Issue #5: The last song in a playlist never shows up

**How I reproduced it**

Ran the existing test suite for playlists (`pytest`), specifically `test_playlist_returns_all_songs`, using the `seed_playlist` fixture. The test asserted that `get_playlist_songs()` returns every song that was added to the playlist. The assertion failed — the returned list was missing the final song in the playlist's ordered sequence, one element short of the expected count.

**How I found the root cause**

Opened `playlist_service.py` and located `get_playlist_songs()`, which queries `Song` joined to the `playlist_entries` association table, filtered by `playlist_id`, and ordered by `position` ascending. The query itself (`.order_by(asc(playlist_entries.c.position)).all()`) correctly returned every song in the playlist, in the right order. The bug was in the return line, which sliced the result list before converting it to dicts:

```python
return [song.to_dict() for song in songs[:-1]]
```

Since `songs` was already fully and correctly ordered by the query, applying `[:-1]` had no purpose other than dropping the last element — confirmed by checking `songs` length against the sliced output length, which differed by exactly one.

**The root cause**

`get_playlist_songs()` built the correct, fully-ordered list of songs from the database, but then discarded the last item with a `[:-1]` slice on the return line. `[:-1]` returns all elements except the final one, so any playlist's last song by position was silently dropped from every response, regardless of playlist length (as long as it had at least one song).

**My fix and side-effect check**

Changed the return line from `songs[:-1]` to `songs`, since the query already returns the complete, correctly-ordered list and no slicing was needed:

```python
return [song.to_dict() for song in songs]
```

Re-ran `test_playlist_returns_all_songs` and confirmed it passed. Also checked playlists with only a single song to make sure that song still appears (rather than the slice bug causing an empty list in a different way), and re-checked a multi-song playlist's ordering to confirm songs still appear in correct `position` order, not just correct count.
