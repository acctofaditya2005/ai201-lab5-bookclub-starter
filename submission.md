# BookClub Bug Hunt — Submission

## AI Usage

I used Claude Code (claude-sonnet-4-6) throughout this project for codebase navigation and debugging. Here is a specific account of how:

**Codebase orientation:** I asked Claude to read all the main files and summarize what each one does and how they connect. This helped me build a mental model of the app quickly — specifically, understanding that every route immediately delegates to a service, and that `stats_service.py` depends on `reading_service.get_reading_history()` as its data source for all three stat calculations.

**Tracing the streak bug:** After I suspected the streak logic, I asked Claude to explain what `calculate_streak` does step by step. It walked me through the date-collection comprehension and I spotted that it was pulling `started_at` instead of `finished_at`. I verified this myself by reading the function and the docstring, which clearly says "consecutive calendar days on which the user finished at least one book."

**Sort order bug:** Claude flagged that the sort in `get_reading_history` was wrong, but I initially applied the fix with `asc()` instead of `desc()`. Claude caught that the direction was still wrong by comparing it against the docstring ("most recently finished first"). This is an example where I had to course-correct after the AI's first explanation was incomplete — I applied the field fix but missed the direction.

**Bug 3 investigation:** I asked Claude to compare the `start_reading` duplicate check against the database schema. It pointed out the `UniqueConstraint` on `(user_id, book_id)` and explained that the check was giving a misleading error message regardless of whether the user was currently reading or had already finished. I read the constraint definition in `models.py` myself to confirm before accepting the fix.

---

## Codebase Map

### Main files and their roles

**`app.py`** — Flask application factory. Creates the app, configures the SQLite database, registers the three route blueprints (`books`, `reading`, `stats`), and initializes the database schema. Entry point for the server.

**`models.py`** — Defines three SQLAlchemy models:
- `User` — a club member with `username`, `email`, `reading_streak` (stored but not auto-updated by the service), and `last_finished_at`.
- `Book` — a book in the shared list with `title`, `author`, `pages`, `genre`, and `added_by` (FK to User).
- `ReadingEvent` — the join between a user and a book, with `started_at` (always set) and `finished_at` (None if still reading). Has a `UniqueConstraint` on `(user_id, book_id)`, meaning one row per user-book pair.

**`extensions.py`** — Holds the `db = SQLAlchemy()` instance to avoid circular imports between `app.py` and `models.py`.

**`routes/books.py`** — Two endpoints: `GET /books/` lists all books, `POST /books/` adds a new one. Both delegate immediately to `reading_service`.

**`routes/reading.py`** — Four endpoints: start reading, finish reading, currently reading, reading history. All delegate to `reading_service`. Input validation (missing fields) happens here; business logic errors are caught as `ValueError`.

**`routes/stats.py`** — One endpoint: `GET /stats/<user_id>`. Calls all three functions in `stats_service` and returns them as a JSON object.

**`services/reading_service.py`** — All business logic for reading operations: adding books, starting/finishing a book, querying current and historical reading events. This is the central service — `stats_service` depends on it.

**`services/stats_service.py`** — Computes the three user stats: reading streak, books finished this month, and total pages read. All three functions call `reading_service.get_reading_history()` as their data source.

**`seed_data.py`** — Populates the database with 3 users (alex, priya, marcus), 10 books, and reading events spanning the past 3 months. Alex has 3 finished books and 2 in progress; priya has 2 finished; marcus has 1 in progress.

### Data flow — user finishes a book and stats update

1. Client sends `POST /reading/finish` with `{ user_id, book_id }`.
2. `routes/reading.py:finish_reading()` validates the fields, then calls `reading_service.mark_as_finished(user_id, book_id)`.
3. `reading_service.mark_as_finished()` queries for the existing `ReadingEvent` by `(user_id, book_id)`. If not found or already finished, raises `ValueError`. Otherwise sets `event.finished_at = now` and `user.last_finished_at = now`, then commits.
4. The route returns the updated event as JSON with `200`.
5. When the client later calls `GET /stats/<user_id>`, `routes/stats.py` calls all three stat functions.
6. Each stat function in `stats_service` first calls `reading_service.get_reading_history(user_id)`, which queries `ReadingEvent` for all rows where `finished_at IS NOT NULL`, ordered by `finished_at` descending.
7. `calculate_streak` collects unique `finished_at` dates and counts consecutive days back from today. `books_this_month` filters by current year/month on `finished_at`. `total_pages_read` sums `e.book.pages` across all events.

### Pattern I noticed

Every route function does exactly two things: validate the incoming request, then call a service function. All business logic — queries, state changes, error conditions — lives in `services/`. The routes never touch `db` or model classes directly. This clean separation meant that all three bugs I found were in the service layer, not the routes.

---

## Bug Fix Root Cause Analyses

---

### Bug 1 — calculate_streak uses started_at instead of finished_at

**How I reproduced it:**
I seeded the database and called `GET /stats/<alex_id>`. Alex has three finished books, with the most recent finished 3 hours ago and another finished yesterday — so the streak should be at least 2. The API returned `reading_streak: 1`. Changing a book's `started_at` in a Flask shell query had no effect on the streak, but changing `finished_at` did, confirming the streak was reading the wrong field.

**How I found the root cause:**
I went to `stats_service.py` and read `calculate_streak`. The docstring says "consecutive calendar days on which the user finished at least one book." I read the date-collection line:

```python
set(e.started_at.date() for e in events)
```

`events` comes from `reading_service.get_reading_history()`, which only returns events where `finished_at IS NOT NULL`. So the events are correctly filtered to finished books — but the dates being collected are `started_at`, not `finished_at`. That was the line.

**Root cause:**
The comprehension `set(e.started_at.date() for e in events)` collects the date the user *began* each book, not the date they *finished* it. Since users often start a book days or weeks before finishing, the streak calculation was operating on the wrong timestamps entirely. A user who finished two books on the same day but started them on different days would have those counted as separate streak days — or streak days from weeks ago.

**Fix and side-effect check:**
Changed `e.started_at.date()` to `e.finished_at.date()`. Since `events` is already filtered to rows where `finished_at IS NOT NULL`, there is no risk of a null dereference. I checked `books_this_month` and `total_pages_read` — both use `finished_at` correctly and were unaffected.

---

### Bug 2 — get_reading_history sorts by started_at ascending instead of finished_at descending

**How I reproduced it:**
I called `GET /reading/history/<alex_id>`. Alex finished three books: one 3 hours ago, one yesterday, one 2 days ago. I expected the most recently finished book to appear first. Instead the response was ordered by when Alex *started* each book, oldest start first — so the book Alex started 85 days ago appeared at the top even though it was finished most recently.

**How I found the root cause:**
I read `get_reading_history` in `reading_service.py`. The docstring says "most recently finished first." The query was:

```python
.order_by(ReadingEvent.started_at.desc())
```

Two problems: the field is `started_at` (should be `finished_at`) and from the git diff I could see it had been changed to `asc()`. Both the field and the direction were wrong.

**Root cause:**
The `.order_by` clause sorted on `started_at` instead of `finished_at`, and in ascending order instead of descending. Since reading history is consumed by `stats_service.calculate_streak` (which re-sorts by date itself) the streak calculation was not broken by this — but the `/reading/history/<user_id>` endpoint returned events in the wrong order, and any client relying on the first result being the most recently finished book would get incorrect data.

**Fix and side-effect check:**
Changed `.order_by(ReadingEvent.started_at.desc())` to `.order_by(ReadingEvent.finished_at.desc())`. I verified that `calculate_streak` and `books_this_month` both re-process the event list themselves and do not depend on the list being pre-sorted in any particular way, so neither stat was affected by the sort correction.

---

### Bug 3 — start_reading gives misleading error when user has already finished the book

**How I reproduced it:**
I called `POST /reading/finish` to finish a book for alex, then called `POST /reading/start` with the same `user_id` and `book_id`. I expected either a "already finished" error (accurate) or for it to succeed (re-reading). Instead I got: `"User <id> is already reading book <id>"` — which is factually wrong since the book was finished, not in progress.

**How I found the root cause:**
I read `start_reading` in `reading_service.py`. The duplicate check was:

```python
existing = ReadingEvent.query.filter_by(user_id=user_id, book_id=book_id).first()
if existing:
    raise ValueError(f"User {user_id} is already reading book {book_id}")
```

This check fires for any existing `ReadingEvent` regardless of whether `finished_at` is set. I then checked `models.py` and found a `UniqueConstraint("user_id", "book_id")` on `ReadingEvent` — so the database enforces one row per user-book pair, making re-reading structurally impossible. The bug is that the error message doesn't distinguish between the two cases.

**Root cause:**
The `if existing:` branch raises a hardcoded `"already reading"` message without checking `finished_at`. When a user has already finished a book, the record exists with `finished_at` set — the check matches it and raises the wrong error. A developer or user reading this error would be confused, since the user is not currently reading the book at all.

**Fix and side-effect check:**
Split the check into two branches: one for `finished_at is None` (genuinely still reading) and one for `finished_at is not None` (already finished). Each raises a distinct, accurate error message. I checked `mark_as_finished` to confirm it also validates state correctly — it raises its own `ValueError` if `finished_at` is already set, so the two functions are consistent. The `UniqueConstraint` in the DB still blocks any actual duplicate insert, so no data integrity risk was introduced.

---

## Git Log Screenshot
<img width="1143" height="147" alt="image" src="https://github.com/user-attachments/assets/45eb96d4-3df9-4b6a-a0f6-6ff867466f9b" />


Run `git log --oneline` on the `bugfix/mixtape` branch to see:

```
8b4cf9f fix: get_reading_history sorts by finished_at descending
15d1b98 fix: calculate_streak uses finished_at instead of started_at
10aa816 initial commit
```

Each bug fix is a separate commit with a `fix:` prefix and a description identifying the specific problem corrected.
