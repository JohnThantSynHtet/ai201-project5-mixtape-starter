# Mixtape Bug Hunt Submission

## AI Usage

I used AI tools to help me understand unfamiliar files, summarize service functions, trace call chains, and compare similar code paths. I verified the actual bug causes by reading the code myself, reproducing the bugs locally, and testing the behavior after each fix.

## Codebase Map

### Main files and folders

- `app.py`: Creates the Flask app, connects the database, and registers the route files.
- `models.py`: Defines the SQLAlchemy database models used by the app, such as users, songs, playlists, playlist songs, and notifications.
- `routes/`: Contains the API endpoints. The route files receive HTTP requests, read inputs from the request, call service functions, and return JSON responses.
- `routes/songs.py`: Handles song search, song detail, listening, and rating routes.
- `routes/playlists.py`: Handles playlist creation, playlist detail, and adding/listing playlist songs.
- `routes/users.py`: Handles user profile, streak, and notification routes.
- `routes/feed.py`: Handles user activity feed and friends listening now routes.
- `services/`: Contains the main business logic. The bugs are expected to be here.
- `services/streak_service.py`: Handles listening streak logic.
- `services/feed_service.py`: Handles friends listening now and activity feed logic.
- `services/search_service.py`: Handles song search logic.
- `services/notification_service.py`: Handles notification creation and retrieval.
- `services/playlist_service.py`: Handles playlist retrieval logic.
- `tests/`: Contains existing tests for streaks, search, and playlists.
- `seed_data.py`: Creates sample database data for local testing.
- `requirements.txt`: Lists Python dependencies.

### Pattern I noticed

The app is organized with a route/service pattern. The route files do not contain most of the business logic. Instead, routes receive requests and call service functions. The service files contain the logic that decides what data to query, create, update, or return.

### Data flow example: rating a song

1. A user sends a `POST` request to `/songs/<song_id>/rate`.
2. The request goes to `routes/songs.py`.
3. The route reads the song ID and rating information from the request.
4. The route calls a function in `notification_service.py`.
5. The service updates the song rating behavior and should create a notification for the right user.
6. The route returns a JSON response.

### Data flow example: viewing playlist songs

1. A user sends a `GET` request to `/playlists/<playlist_id>/songs`.
2. The request goes to `routes/playlists.py`.
3. The route calls `playlist_service.get_playlist_songs()`.
4. The service queries the database for songs in that playlist.
5. The route returns the playlist songs as JSON.

## Issue #1: My listening streak keeps resetting

### How I reproduced it

I ran the streak tests using `pytest tests/test_streaks.py`. The test `test_streak_increments_on_sunday` failed because the user's listening streak stayed at 1 instead of increasing to 2 after listening on Saturday and then Sunday.

### How I found the root cause

I checked the project README to see that this issue was related to `streak_service.py`. I opened the `update_listening_streak()` function and followed the logic that updates the streak based on the number of days since the user's last listening event. I noticed an additional condition that checked whether the current day was Sunday.

### The root cause

The function only increased the streak when `days_since_last == 1` **and** the current day was not Sunday (`today.weekday() != 6`). Because Python's `weekday()` returns `6` for Sunday, listening on Saturday and then Sunday incorrectly skipped the increment and reset the streak instead.

### My fix and side-effect check

I removed the unnecessary Sunday check so the streak increments whenever the user listens on consecutive days. After making the change, I ran `pytest tests/test_streaks.py` and all streak tests passed. I also verified that the streak still does not increase when listening multiple times on the same day and still resets correctly after skipping a day.