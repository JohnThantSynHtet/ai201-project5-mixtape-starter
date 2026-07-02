# Mixtape Bug Hunt Submission

## AI Usage

I used AI tools to help me understand the structure of the project before making any changes. AI helped summarize unfamiliar service files, explain how functions worked, trace the flow from routes to services, and compare similar code paths. During debugging, I used AI to explain suspicious sections of code and verify my understanding of Python behavior, but I confirmed every root cause myself by reading the code, reproducing the bug locally, testing my fixes, and rerunning the test suite.

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
4. The route calls the appropriate function in the `services/` layer.
5. The service updates or retrieves data from the database.
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

The function only increased the streak when `days_since_last == 1` and the current day was not Sunday (`today.weekday() != 6`). Because Python's `weekday()` returns `6` for Sunday, listening on Saturday and then Sunday incorrectly skipped the increment and reset the streak instead.

### My fix and side-effect check

I removed the unnecessary Sunday check so the streak increments whenever the user listens on consecutive days. After making the change, I ran `pytest tests/test_streaks.py` and all streak tests passed. I also verified that the streak still does not increase when listening multiple times on the same day and still resets correctly after skipping a day.

## Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I examined the notification workflow described in the README and compared how notifications were created when a song was added to a playlist versus when a song was rated. The rating logic completed successfully, but no notification was created for the song's owner.

### How I found the root cause

I followed the route from the README into `notification_service.py`. I compared the `add_to_playlist()` function with the `rate_song()` function. The playlist function created a notification after updating the playlist, while the rating function only saved the rating and returned.

### The root cause

The `rate_song()` function updated or created the rating and committed it to the database, but it never called `create_notification()`. As a result, the song owner was never notified when someone else rated their shared song.

### My fix and side-effect check

I added a call to `create_notification()` after the rating was successfully saved. I also checked that notifications are only created when someone rates another user's song, preventing users from receiving notifications for rating their own songs. After making the change, I ran the full test suite and all tests passed.

## Issue #5: The last song in a playlist never shows up

### How I reproduced it

I ran the playlist tests using `pytest tests/test_playlists.py`. The tests failed because `get_playlist_songs()` returned only 4 songs instead of the expected 5. The last song in the playlist was always missing.

### How I found the root cause

I started by reading the README, which pointed to `playlist_service.py` as the affected service. I opened the `get_playlist_songs()` function and traced how the songs were queried and returned. The database query correctly returned every song, so I focused on the return statement and noticed that it removed the final element before returning the results.

### The root cause

The function returned `songs[:-1]`, which is a Python slice that returns every element except the last one. Because of this, the last song was always excluded from the playlist even though the database query retrieved it correctly.

### My fix and side-effect check

I changed the return statement to iterate over `songs` instead of `songs[:-1]`, allowing every song to be returned. After making the change, I ran `pytest tests/test_playlists.py` and confirmed that all playlist tests passed. I also verified that the songs were still returned in the correct order.