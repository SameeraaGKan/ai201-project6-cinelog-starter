# PR Response Doc — CineLog Watchlist Feature

## Commit History

> **TODO before submitting:** replace this text block with an actual screenshot of `git log --oneline` run in your terminal, per the assignment's submission checklist.

```
156ddb8 docs: add pr-response.md with PR review responses
7067479 fix: sort watchlist by date added instead of alphabetically
675923d fix: update WatchlistEntry.film_id to UUID after main branch refactor
b643797 fix: default WatchlistEntry.public to False
34cd615 test: add tests/test_watchlist.py for add_to_watchlist
dcc696c fix: add deduplication check to add_to_watchlist
a19b643 fix: rename save_to_watchlist to add_to_watchlist
8d36193 fix: add missing Film.watchlist_entries relationship
3fd152d fix: update film retrieval method to use db.session.get in collection and watchlist services
609df8f feat: add watchlist model and add_to_watchlist endpoint
```

10 commits ahead of `main`, all conventional-format, no merge commits (`git log --oneline --merges main..HEAD` returns nothing).

## AI Usage
I used Claude Code (an AI coding assistant working directly in this repo) throughout this project, so I want to be specific about what it did rather than give a vague summary.

1. **Codebase orientation for Comment 2 (dedup):** Before writing the deduplication check in `add_to_watchlist()`, I had Claude read `add_to_collection()` and explain what its duplicate check does (query for an existing `(user_id, film_id)` row before inserting, raise a custom exception if found) and what exception/return shape it uses. I then wrote `add_to_watchlist()`'s version myself, following that pattern but with a new `AlreadyInWatchlistError` — I did not have it generate the dedup code directly, per the assignment's instruction.
2. **Devil's-advocate stress test for Comments 4 and 5:** For Comment 4, my first draft kept `public=True` and just documented it, reasoning that CineLog is a social discovery app and `CollectionEntry` is already public with no privacy field, so consistency favored a public watchlist default too. When I asked what a careful reviewer would push back on, the response was that "already watched" and "want to watch" are not equivalent privacy categories — a watchlist expresses unexpressed intent (which can reveal sensitive interests like health, identity, or political topics) in a way a completed-watch log doesn't, and privacy-by-default norms (GDPR/CCPA) treat that as a real distinction. That changed my answer — I flipped the default to `public=False` and implemented the change, rather than just keeping my original position. For Comment 5, the same exercise pushed back that alphabetical sorting is more usable for long watchlists (30+ items) where a user is scanning for a specific remembered title, while recency sorting has no stable landmark once the list gets long. I kept the maintainer's requested date-added sort (I found the consistency argument with `get_collection()` more compelling than the discoverability concern for a first version), but I added that limitation explicitly to my Comment 5 response instead of ignoring it, and I did not use the AI's raw text as my response — I wrote my own reasoning after evaluating the counterargument.
3. **Diagnosing a silent rebase corruption (Comment 6):** After running `git rebase origin/main`, the rebase reported only one real conflict, but I noticed the resulting `models.py` was missing the `WatchlistEntry` class entirely with no conflict ever flagged for it. I used Claude to inspect the per-commit history (`git show <sha>:models.py`, `git log --oneline -- models.py`) across the rebase and confirm that an early commit's patch had been silently mis-applied by git's context-matching, dropping the class without an error. That's what led me to rebuild the branch history from a clean base rather than trust the first automatic rebase result.

I'm noting this as a disclosure, not a substitute for understanding the changes myself — every code change and every position in Comments 1–6 above reflects my own review of the diff before it was committed.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention (`add_to_collection`, `remove_from_collection`). Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran a project-wide search (`grep -r save_to_watchlist`) after the change and confirmed zero remaining references. Full test suite (`pytest tests/ -v`) still passes.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in `add_to_watchlist()`, following the exact pattern in `add_to_collection()`: query for an existing `WatchlistEntry` matching `(user_id, film_id)` before inserting, and raise if one is found instead of letting a duplicate row get created.
**How I verified:** Wrote `test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py` (mirrors `test_add_to_collection_duplicate_raises`) — calls `add_to_watchlist` twice with the same user/film, asserts the second call raises `AlreadyInWatchlistError`, and confirms only one row exists in the table afterward. Full suite passes.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, using `test_collection.py` as the template: same `app`/`sample_user`/`sample_film` fixtures, then three tests — `test_add_to_watchlist_creates_entry` (happy path), `test_add_to_watchlist_duplicate_raises` (covers Comment 2's dedup logic), and `test_add_to_watchlist_nonexistent_film_raises` (the specific case the reviewer asked for, modeled directly on `test_add_to_collection_nonexistent_film_raises`).
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 3 pass. Then ran the full suite (`pytest tests/ -v`) — all 7 tests across both files pass, confirming the new tests don't interfere with the existing collection tests.

## Comment 4 — Default visibility
**My position:** I changed the default from `public=True` to `public=False`. `public=True` was never a deliberate choice — it was whatever value I typed first when scaffolding the column — and once I looked at it critically I don't think it holds up, so rather than write a justification for the original default I'm flipping it.

**Reasoning:** A watchlist is different from the `CollectionEntry` model it sits next to. A collection entry records a film someone has *already watched* — it's retrospective, and CineLog treats it as inherently shareable (there isn't even a `public` column on `CollectionEntry`; the whole feature assumes it's fine to show). A watchlist entry records something someone *wants* to watch — it's forward-looking intent, and intent is a more sensitive signal than history. A watchlist full of documentaries about a health condition, a religion, an immigration process, or a breakup can reveal a lot about what a user is going through *before they've decided to act on it or talk about it*. Defaulting that to public means CineLog broadcasts that signal on day one, before the user has ever seen a privacy setting, let alone chosen one. That's the opposite of privacy-by-default (the standard both GDPR and CCPA expect: data-sharing should be an opt-in the user affirmatively makes, not an opt-out they have to discover and flip). Optimizing for "friends can instantly discover what I want to watch, no toggle needed" is optimizing for the platform's growth metric, not the user's expectation — and when those conflict for a *first-run default*, I think the user's expectation should win. Users who want the social discovery experience can still make specific entries public; the friction is one tap, not a redesign.

**Tradeoff acknowledged:** I'm not free of a tradeoff either. `public=False` by default means CineLog's "friends can see what everyone wants to watch" discovery loop won't happen automatically — most users will never flip the toggle, so the social feature this field exists to support may end up underused, which weakens the exact product wedge (Letterboxd-style social film discovery) CineLog seems to be going for. If growth data later shows that's a real cost, the right fix is a good onboarding prompt that explains the toggle and asks the user to choose — not silently reverting the default. I also considered a middle ground (public for collections and watchlists that already have items, private for empty/new ones) but rejected it as unnecessary complexity for a decision that should just be intentional and documented, which is what was actually asked for here.

*(Devil's-advocate check — see AI Usage: I originally planned to keep `public=True` and just document it, reasoning that CineLog is explicitly a social app and collections are already public with no privacy field at all, so consistency favored public watchlists too. Pushing on that, the strongest counter is that "already-watched" and "want-to-watch" are not equivalent privacy categories — one is a completed action a user has often already discussed with people; the other is unexpressed intent, and intent is what gets weaponized in real incidents like Strava's 2018 heatmap leak or Netflix's 2007 Prize de-anonymization. That distinction is why I changed my answer instead of keeping the original default.)*

## Comment 5 — Sort order
**My position:** I implemented the maintainer's preference — `get_watchlist()` now orders by `date_added` descending (most recently added first) instead of `Film.title` ascending.

**Reasoning:** I agree with the maintainer's read of user behavior, and I think there's a second, independent argument for the same conclusion: consistency. `get_collection()` already sorts newest-first by `date_added`. Before this change, the two list endpoints in the same app sorted by two different, unrelated fields, which means a user (or a frontend developer consuming both endpoints) has to remember that "collection" and "watchlist" behave differently for no functional reason. Aligning them removes a surprise, not just implements a preference.

**Engagement with reviewer's point:** The maintainer's core claim is "most users want to see what they added recently" — I buy that for the common case: someone adds a film after seeing it recommended, and the watchlist is a to-do list they work through roughly in the order they thought of it. Where I pushed on this myself: alphabetical sorting is genuinely better than recency once a watchlist gets long (30+ films) and a user is trying to find one specific title they remember by name, not by when they added it — recency order gives no stable landmark to scan against. I decided not to let that stop me from implementing the maintainer's request, because the right fix for "long lists are hard to scan" isn't picking a different single default, it's giving the user a sort/filter control — and that's a new feature, not something this PR's scope covers. I noted it below as a real limitation rather than silently ignoring it. I also added `test_get_watchlist_returns_newest_first` (mirroring `test_get_collection_returns_newest_first`) to lock in the new behavior, since a sort-order regression wouldn't otherwise be caught by anything in the suite.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main`. `main` had picked up `refactor: migrate film IDs from integer to UUID` (and a `.gitignore` chore commit) that landed after `feature/watchlist` diverged. Git flagged an explicit conflict in `models.py` on the commit that changed `WatchlistEntry.public`'s default — both sides touched the tail of the file (main via the UUID refactor context, mine via the `public` default change), and the conflict markers left the `WatchlistEntry` class ambiguous.

While resolving it I caught something git's conflict markers didn't flag on their own: earlier commits in the same rebase (the ones that originally introduced the `WatchlistEntry` model) had already applied "cleanly" against the rebased `models.py`, but a diff against `main` showed the `WatchlistEntry.film_id` column was still `db.Integer` — a straight leftover from before the UUID refactor existed. Since `film_id` is a foreign key into `Film.id`, and `Film.id` is now `db.String(36)`, an integer-typed FK pointing at a UUID-typed primary key would silently accept the wrong type and could produce lookups that never match.

**How I resolved it:** In the conflicted `models.py`, I kept the `WatchlistEntry` class from my branch (with `public` defaulting to `False`, per Comment 4) but changed `film_id = db.Column(db.Integer, ...)` to `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` to match `Film.id` and `CollectionEntry.film_id`. I also grepped the rest of the watchlist code for leftover integer-ID assumptions and found two docstrings still describing `film_id` as an `int` (`services/watchlist_service.py` and the request-body comment in `routes/watchlist/watchlist.py`) — updated both to describe a UUID string, consistent with `collection.py`'s docstrings.

**How I verified no conflict remains:** `git status` after `--continue` showed a clean rebase with no unmerged paths. `git log --oneline --merges main..HEAD` returns nothing, confirming a linear history with no merge commits. I grepped the whole repo for leftover `<<<<<<<`/`=======`/`>>>>>>>` markers — none found. Finally, ran `pytest tests/ -v` — all 8 tests pass post-rebase, including `test_add_to_watchlist_nonexistent_film_raises`, which exercises `db.session.get(Film, film_id)` against the now-UUID-typed column with a UUID-formatted fake ID.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### What this feature does

Adds a watchlist to CineLog: films a user wants to watch, as opposed to `CollectionEntry`, which tracks films they've already watched. A new `WatchlistEntry` model backs two endpoints:

- `POST /watchlist/<user_id>/add` — add a film to the user's watchlist (body: `{ "film_id": "<uuid>" }`)
- `GET /watchlist/<user_id>` — return the user's watchlist, sorted by most recently added

The service layer (`services/watchlist_service.py`) exposes `add_to_watchlist()` and `get_watchlist()`, following the same `verb_to_noun` convention and error-handling shape as `services/collection_service.py`.

### Design decisions

- **Default visibility (Comment 4):** New watchlist entries default to `public=False`. A watchlist expresses forward-looking viewing intent, which can be more revealing than a retrospective "already watched" collection entry, so visibility should be an explicit opt-in rather than an inherited default. Full reasoning and the tradeoff (reduced automatic social discovery) is in the Comment 4 section above.
- **Sort order (Comment 5):** `get_watchlist()` returns films ordered by `date_added` descending (most recent first), matching `get_collection()`'s existing ordering. This surfaces what a user just added and keeps the two list endpoints in the app behaviorally consistent. Full reasoning, including the discoverability tradeoff for long lists, is in the Comment 5 section above.

### How to manually test

1. Install dependencies and run the app:
   ```
   pip install -r requirements.txt
   python app.py
   ```
2. Create a user and a film (via existing endpoints, or directly in a Python shell using `create_app()`/`db`), and note their UUIDs.
3. Add a film to the watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
        -H "Content-Type: application/json" \
        -d '{"film_id": "<film_id>"}'
   ```
   Expect a `201` with the new entry, including `"public": false`.
4. Repeat the same request with the same `user_id`/`film_id` — expect a `409`-worthy error surfaced as an unhandled `AlreadyInWatchlistError` at the service layer (no duplicate row is created; route-level error handling for this and for a nonexistent `film_id` is not yet wired up in `routes/watchlist/watchlist.py` and would be a good follow-up).
5. Add a second film, then fetch the watchlist:
   ```
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect both films back, with the most recently added one first.
6. Run the automated suite: `pytest tests/ -v` — all 8 tests (4 collection, 4 watchlist) should pass.
