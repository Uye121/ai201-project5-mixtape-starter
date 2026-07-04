# Mixtape Bug Hunt Submission

## AI Usage
I used Claude to understand how the Flask app works. I also used it to generate the codebase map based on the code in this repository.

## Codebase Map

Mixtape is a backend application built using Flask and SQLAlchemy. The application follows a route-to-service pattern.

### Main Files

#### Application

- [app.py](app.py) - Flask application factory
    - Creates and configures the Flask app with SQLAlchemy and environment variables
    - Registers all route blueprints (`/songs`, `/playlists`, `/users`, `/feed`)
    - Initializes the database and creates tables on startup
    - Entry point for running the development server
- [seed_data.py](seed_data.py) - Populate the database with test data
- [models.py](models.py) - SQLAlchemy ORM models defining the database schema
    - **User** - User accounts with `username`, `email`, `listening_streak`, and `last_listened_at`. Has a self-referential many-to-many `friends` relationship via the `friendships` association table.
    - **Song** - Music tracks with `title`, `artist`, `album`, `genre`, and `shared_by` (the user who shared it). Has a many-to-many `tags` relationship via `song_tags`.
    - **ListeningEvent** - Audit log recording when a user listened to a song (`user_id`, `song_id`, `listened_at`).
    - **Rating** - User ratings (1-5 stars) for songs. Has a unique constraint on (`user_id`, `song_id`) so each user can rate a song only once.
    - **Playlist** - Named collections of songs with is_collaborative flag. Songs are ordered via the `playlist_entries` join table which stores `position`, `added_by`, and `added_at`.
    - **Tag** - Categorization labels for songs (genre, mood, etc.).
    - **Notification** - User alerts for social interactions (`notification_type`, `body`, `read` flag).

#### Routes

- [routes/songs.py](routes/songs.py) - Song discovery and interaction endpoints.
    - `GET /songs/search?q=<query>` - Search songs by title or artist (case-insensitive). Delegates to search_service.`search_songs()`.
    - `GET /songs/<song_id>` - Get detailed metadata for a single song. Delegates to `search_service.get_song()`.
    - `POST /songs/<song_id>/rate` - Rate a song (1-5 stars). Requires `user_id` and `score` in request body. Delegates to `notification_service.rate_song()` which creates a Rating record and notifies the song's original sharer.
    - `POST /songs/<song_id>/listen` - Record that a user listened to a song. Requires `user_id` in request body. Delegates to `streak_service.record_listening_event()` which creates a ListeningEvent and updates the user's listening streak.
- [routes/playlists.py](routes/playlists.py) - Playlist management endpoints.
    - `POST /playlists/` - Create a new playlist. Requires `name` and `created_by`; optional is_collaborative (defaults to True). Delegates to `playlist_service.create_playlist()`.
    - `GET /playlists/<playlist_id>` - Get playlist metadata (name, creator, creation date). Delegates to `playlist_service.get_playlist()`.
    - `GET /playlists/<playlist_id>/songs` - Get songs in a playlist in explicit position order. Delegates to `playlist_service.get_playlist_songs()`.
    - `POST /playlists/<playlist_id>/songs` - Add a song to a playlist. Requires `song_id` and `added_by` in request body. Delegates to `notification_service.add_to_playlist()` which notifies the song's original sharer if they aren't the adder.
- [routes/users.py](routes/users.py) - User profile and notification endpoints.
    - `GET /users/<user_id>` - Get public user profile (`username`, `streak`, `last_listened_at`). Delegates to direct User model query.
    - `GET /users/<user_id>/streak` - Get current listening streak. Delegates to `streak_service.get_streak()`.
    - `GET /users/<user_id>/notifications?unread_only=true` - Get user's notifications (optionally unread only). Delegates to `notification_service.get_notifications()`.
    - `POST /users/notifications/<notification_id>/read` - Mark a notification as read. Delegates to `notification_service.mark_as_read()`.
- [routes/feed.py](routes/feed.py) - Social feed endpoints.
    - `GET /feed/<user_id>/listening-now`- Get what friends have listened to recently (last 24 hours). Delegates to `feed_service.get_friends_listening_now()` which deduplicates to show only the most recent song per friend.
    - `GET /feed/<user_id>/activity` - Get recent listening activity from all friends (most recent N events, no time window). Delegates to `feed_service.get_activity_feed()`.

#### Services

- [services/feed_service.py](services/feed_service.py) - Social feed generation.
    - `get_friends_listening_now(user_id)` - Queries ListeningEvents from friends in the last 24 hours. Deduplicates to show only the latest song per friend. Returns enriched feed entries with friend details and song metadata.
    - `get_activity_feed(user_id, limit=20)` - Returns the most recent N listening events from friends, regardless of when they happened. No deduplication, shows all events.
- [services/notification_service.py](services/notification_service.py) - Notification management and social actions.
    - `create_notification(user_id, notification_type, body)` - Creates a Notification record in the database.
    - `rate_song(user_id, song_id, score)` - Creates or updates a Rating record. Validates score is 1-5. If the rater isn't the song's sharer, creates a Notification for the sharer.
    - `add_to_playlist(playlist_id, song_id, added_by_user_id)` - Adds a song to a playlist with ordering. Checks if song is already in the playlist first. Calculates new position as max(position) + 1. If the adder isn't the song's sharer, creates a Notification for the sharer.
    - `get_notifications(user_id, unread_only=False)` - Returns notifications for a user, most recent first.
    - `mark_as_read(notification_id)` - Marks a notification as read.
- [services/playlist_service.py](services/playlist_service.py) - Playlist business logic.
    - `create_playlist(name, created_by_user_id, is_collaborative=True)` - Creates a new Playlist record.
    - `get_playlist_songs(playlist_id)` - Returns songs ordered by position.
    - `get_playlist(playlist_id)` - Returns playlist metadata as a dict.
    - `get_user_playlists(user_id)` - Returns all playlists created by a user.
- s[ervices/search_service.py](ervices/search_service.py) - Song discovery.
    - `search_songs(query)` - Searches songs by `title` or `artist` using case-insensitive `LIKE` queries (e.g., `%query%`). Returns songs with their associated tags.
    - `get_song(song_id)` - Returns a single song's metadata.
- [services/streak_service.py](services/streak_service.py) - Listening streak tracking.
    - `record_listening_event(user_id, song_id)` - Creates a ListeningEvent and updates the user's streak. Delegates to `update_listening_streak()`.
    - `update_listening_streak(user, now)` - Core streak logic:
        - First listen ever → streak = 1
        - Already listened today → no change
        - Listened yesterday → streak += 1
        - More than 1 day gap → streak resets to 1
    - `get_streak(user_id)` - Returns the user's current streak value.

### Data Flows

#### User rates a song

```text
POST /songs/song123/rate
Body: { "user_id": "user456", "score": 5 }

routes/songs.py
  ↓ Validates required fields (user_id, score)
  ↓ Delegates to notification_service.rate_song()

services/notification_service.py
  ↓ Validates score is 1-5
  ↓ Fetches Song by ID — gets shared_by = "user789", title = "Bohemian Rhapsody"
  ↓ Checks existing Rating for (user456, song123)
  ↓ If exists → updates score, else creates new Rating
  ↓ If song.shared_by != user_id (user789 != user456):
      create_notification(
          user_id="user789",
          notification_type="song_rated",
          body="user456 rated your song 'Bohemian Rhapsody' 5 stars."
      )
  ↓ Commits to database

Database
  ↓ INSERT/UPDATE rating table
  ↓ INSERT INTO notification table

Response: 201 Created + Rating JSON
```

#### User listens to a song (streak update)

```text
POST /songs/song123/listen
Body: { "user_id": "user456" }

routes/songs.py
  ↓ Validates required field (user_id)
  ↓ Delegates to streak_service.record_listening_event()

services/streak_service.py
  ↓ Fetches User by ID
  ↓ Creates ListeningEvent(user_id, song_id, listened_at=NOW)
  ↓ update_listening_streak(user, NOW):
      today = NOW.date()
      if no last_listened_at → streak = 1
      elif today == last_date → no change
      elif days_since_last == 1 → streak += 1
      else → streak = 1 (reset)
  ↓ Updates user.last_listened_at = NOW
  ↓ Commits to database

Database
  ↓ INSERT INTO listening_event
  ↓ UPDATE user SET listening_streak = X, last_listened_at = NOW

Response: 201 Created + ListeningEvent JSON
```

#### Friend adds a song to a collaborative playlist

```text
POST /playlists/playlist123/songs
Body: { "song_id": "song456", "added_by": "friend1" }

routes/playlists.py
  ↓ Validates required fields (song_id, added_by)
  ↓ Delegates to notification_service.add_to_playlist()

services/notification_service.py
  ↓ Fetches Song — gets shared_by = "user123", title = "Bohemian Rhapsody"
  ↓ Fetches User (friend1)
  ↓ Fetches Playlist — gets name = "Best of Rock", is_collaborative = True
  ↓ Checks if song already in playlist:
      Query playlist_entries for (playlist123, song456)
  ↓ If not present:
      Get max position from playlist_entries
      Calculate new_position = max + 1
      INSERT INTO playlist_entries (playlist_id, song_id, position, added_by, added_at)
  ↓ If song.shared_by != added_by (user123 != friend1):
      create_notification(
          user_id="user123",
          notification_type="song_added_to_playlist",
          body="friend1 added your song 'Bohemian Rhapsody' to 'Best of Rock'."
      )
  ↓ Commits to database

Database
  ↓ INSERT INTO playlist_entries
  ↓ INSERT INTO notification

Response: 201 Created + { "message": "Song added to playlist" }
```

#### Get friends' listening feed

```text
GET /feed/user123/listening-now

routes/feed.py
  ↓ Delegates to feed_service.get_friends_listening_now()

services/feed_service.py
  ↓ Fetches User — gets friends list: [friend1, friend2, friend3]
  ↓ Query ListeningEvents from friends in last 24 hours
      WHERE user_id IN (friend1, friend2, friend3)
        AND listened_at >= NOW() - 24 hours
      ORDER BY listened_at DESC
  ↓ Deduplicate:
      seen_friends = set()
      For each event (most recent first):
        if event.user_id not in seen_friends:
          Add to result, mark seen
          Fetch friend details and song details
  ↓ Returns list of feed entries

Database
  ↓ SELECT * FROM listening_event WHERE user_id IN (...) AND listened_at >= cutoff
  ↓ SELECT * FROM user WHERE id IN (friend IDs)
  ↓ SELECT * FROM song WHERE id IN (song IDs)

Response: 
{
  "feed": [
    { "friend": {...}, "song": {...}, "listened_at": "..." },
    ...
  ],
  "count": 2
}
```

### Patterns I noticed
- **Separation of concern:** Route, service and model all have separate function, and they do not overlap. Routes is in charge of input parsing and response formatting. Service takes care of all business logic. Model define classes and relationships between the classes.
- **Simplicity:** all functions perform one thing only.

---

## Root Cause Analysis

### Issue 1 - My listening streak keeps resetting

#### how I reproduced it

I used Flask shell to call `update_listening_streak()` with controlled datetime input. Since the bug occurred on Sunday, I created a test `User` object with a streka of 1 and set `last_listened_at` to Saturday. First, I call the function with a Saturday datetime, then another call with Sunday. After both calls, the streak printed as 1 instead of 2.

#### How I found the root cause

Since the issue seems to be related to updating the streak, I start from the bottom and look at `update_listening_streak()` first. The condition `today.weekday() != 6` stood out as an issue since `weekday()` returns 6 for Sunday. This prevents the streak update on Sunday as it defaults to 1.

#### The root cause

`datetime.weekday()` returns 6 for Sunday, but `update_listening_streak()` has another caluse that prevents the streak from continuing on Sunday as it blocks `today.weekday() != 6`. The function then defaults to the default, which is reset streak to 1.

#### The fix and side-effect check

Remove the clause `today.weekday() != 6` and retest the streaking used to reproduce the bug. Ran `pytest tests/test_streaks.py` and see if it passes the tests.

---

### Issue 5 - The last song in a playlist never shows up

#### how I reproduced it

I called `GET /playlists/<id>/songs for the "Late Night Vibes" playlist. The correct behavior should return 7 songs, but it returned 6 songs. I checked the other playlists and got 1 less than the expected result.

#### How I found the root cause

I check the function that is responsible for getting the song for the playlist, which is `get_playlist_songs()`. I went through the function and saw that it has slice off the last song via `[:-1]`.

#### The root cause

The function `get_playlist_songs()` return all but the last song.

#### The fix and side-effect check

Remove the index slicing `[:-1]`. The fix is verified by checking the count for all three playlists and running `pytest tests/test_playlists.py`.

---

### Issue 4 - I got notified when a friend added my song to a playlist but not when they rated it

#### how I reproduced it

I have the user darius rate a song shared by user simone. User simone did not receive notification even though the rating was saved successfully.

#### How I found the root cause

I made sure to note simone's notifications before and after each actions: `add_to_playlist()`, then `rate_song()`. Adding the song creates a notification for sharer but rating the song creates none.

#### The root cause

`rate_song()` saves the rating and commits it, but it never calls `create_notification()`.

#### The fix and side-effect check
Call `create_notification()` at the end of `rate_song()`. Verify the correct number of notification using the steps in reproducing the issue. Run the pytests to make sure no other side effects.