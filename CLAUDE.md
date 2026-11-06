# alfie-admin (Life Admin Assist)

A chat bot for two people (a couple) to capture to-dos, backlog items and reminders, with a daily digest at 08:00 on weekdays. Users chat with the bot in Element (Matrix).

Before any work, read:

- `docs/requirements.md`: what the bot must do (the source of truth for behaviour).
- `docs/build-plan.md`: architecture, increment order, tests per increment, tooling.

## Current increment

**Increment 1: echo bot.** Details in `docs/build-plan.md` under "Increment 1". When an increment is done, update this line and tick it off in the build plan.

## How we work

1. One increment at a time. Do not build anything from a later increment.
2. Start each increment with the `grill-me` skill to agree scope and edge cases.
3. Use the `tdd` skill: write the tests first, run them and show they fail, then stop and let the user review the tests before writing any implementation.
4. Implement until all tests pass, then refactor with tests green.
5. The user does one manual check from Element on a phone before the increment counts as done.
6. Work on a branch per increment (e.g. `inc-01-echo-bot`). Never force-push, never rewrite pushed history, never delete branches without asking.

## Architecture rules (do not break)

- **Ports and adapters.** `src/lifeadmin/core/` holds the rules and never imports from `src/lifeadmin/adapters/`. A structure test enforces this.
- **Swappable parts**, each behind a port in `core/ports.py`: messenger (Matrix now), extractor (rules parser now, LLM from increment 10), calendar feed, and the host machine (Mac Mini or a Windows laptop with WSL2, VPS last).
- **The core returns replies, it never sends.** `handle(msg) -> list[Reply]`; the adapter sends.
- **Time comes from a `Clock`.** Never call `datetime.now()` or `date.today()` in the core.
- **Identity is mapped in the adapter.** The core sees internal user IDs (`user_a`, `user_b`), never Matrix IDs.
- **All data lives in one folder** (`data_dir` in config): SQLite, encrypted files, `secrets/`. Moving hosts means copying that folder. Only one host runs the bot at a time.
- **Plain text replies.** Provider-only features (reactions, edits, threads) stay inside the adapter.

## Safety and privacy

- No secrets in git. Matrix token and device ID go in `data/secrets/`; API keys in `.env`. Both are git-ignored. Never print tokens or keys in output or logs.
- The bot logs in once via `scripts/login.py`. Never log in on every start (it creates a new device each time).
- Automated tests never touch matrix.org. End-to-end tests use a local Synapse server in Docker (GitHub Actions).
- Logs record event IDs, intents and errors, not full message text, unless debug logging is switched on.
- Only allowlisted Matrix IDs get replies; everyone else gets nothing.

## Stack

- Python 3.12, managed with `uv`.
- `mautrix-python` for Matrix. Check call names against the mautrix docs (https://docs.mau.fi/python/latest/), not from memory.
- `pytest`, `pytest-asyncio`. SQLite from the standard library (from increment 2).
- Dates: UK format (DD/MM/YYYY), time zone `Europe/London`, must handle clock changes.

## Commands (to be set up in increment 1)

```
uv sync                         # install dependencies
uv run pytest                   # unit + contract tests
uv run pytest tests/e2e         # end-to-end, needs the local test Matrix server (Docker)
uv run python scripts/login.py  # one-time bot login, writes data/secrets/bot.json
uv run lifeadmin                # run the bot
```

## Planned layout

```
src/lifeadmin/
  core/        bot.py  commands.py  items.py  intents.py  digest.py  ports.py
  adapters/
    matrix/    client.py  mapping.py
    extract/   rules.py  claude.py  ollama.py  fake.py
    calendar/  ics_feed.py
    fake_messenger.py
  store/       sqlite.py
  main.py      composition root: the only place that picks adapters
deploy/        macos/ (launchd)  wsl/ (systemd)  docker/ (VPS)
scripts/       login.py  create_test_users.py
tests/         unit/  contract/  e2e/  evals/
```

## Environment notes

- Developed on Windows 11 with WSL2 Ubuntu and on macOS. Keep code OS-neutral: `pathlib` for paths, no shell-specific scripts in the core.
- Keep the repo inside the Linux file system (`~/projects/...`), not on the Windows C: drive.
- Element encrypts direct messages by default and it can't be switched off later. Until increment 9, test in a room created with encryption off.
