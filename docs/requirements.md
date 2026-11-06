# Life admin assistant – requirements

28 Oct 2026 · @M · updated 06/11/2026
Copied from the Life Admin Assist project on claude.ai (`claude/life-admin-requirements.md`). Edit here from now on and keep the two in step, or treat this repo copy as the master.

**Update 06/11/2026:** we build a chat bot on Matrix, used through the Element app. Telegram is no longer the plan, and the phone-app route is dropped. The bot is built in a modular way so the chat provider, the AI model and the machine it runs on can be swapped later without rewriting the rules. Build order, architecture and tests: [build-plan.md](build-plan.md).

## Summary

A personal life admin assistant for me and my husband: capture to-dos, backlog items and long-term reminders in seconds, and get one calm digest at 08:00 on weekdays. We use it by chatting with a bot in Element (Matrix).

This is a learning project: I want to build it myself with Claude Code, with your advice on the engineering choices.

### Decisions so far

| Topic | Decision |
|---|---|
| Interface | Chat bot only. No app. |
| Chat provider | Matrix, using the Element app; accounts on matrix.org to start. |
| Reading free text | A rules parser first; an AI model (Claude API, or Ollama locally) added later, with the parser as fallback. |
| Modularity | The chat provider, the text reader (parser or AI model) and the host machine are each swappable. Rules, data and digests depend on none of them. |
| Hosting | Runs on the Mac Mini or Miki's laptop first, one at a time; a rented server (VPS) is the very last step. |
| Build style | Small increments, each with automated tests written first. |

## Requirements

### What an item stores

What's saved behind the scenes for each item. Only the title is needed; every other field is filled in when I give it, or falls back as shown. "Send back wifi router by Saturday #us" stores a title, due date, project (Other) and shared, and nothing else.

| Field | Example | Set by | If not given |
|---|---|---|---|
| Title | Return old router | Me, in plain words | Always needed |
| Type | To-do, Waiting on, Backlog, Recurring | Worked out from what I write; I can correct it | Worked out |
| Project | Home | My #tag | Other |
| Shared | Yes | #us tag | Mine only |
| Due date | 14/11/2026 | Me, or read from a screenshot | Goes to the backlog |
| Repeat rule | Every Saturday; every January | Me, in plain words | One-off |
| Reminder rule | Daily until done | Me ("remind me daily") | Default for its type |
| Urgent | Yes | Me ("urgent" or "!") | Not urgent |
| Key facts | Ref VM-48213, prepaid label | Read from a screenshot or typed by me | Empty |
| Screenshot | Email image | Me; deleted when the item is done | None |
| Status | Open, done | Changes when I tick it off | Open |
| Created / completed dates | 28/10/2026 | Automatic | Automatic |

### Item types and default reminders

| Type | How it's created | Default reminder |
|---|---|---|
| To-do | Anything with a date | In the digest from 3 days before; flagged on the day |
| Waiting on | "waiting for…" | Chase nudge after 5 days, then weekly |
| Backlog | Anything with **no date** | Never in the daily digest; reviewed in Friday's digest |
| Recurring – appointments | "every Saturday", "every January", "every 6 months" | In the digest from 4 weeks before |
| Recurring – renewals | Same | In the digest from 2 weeks before |

Defaults live in one place and can be changed later in a sentence. A waiting-on item becomes a to-do when I add the deadline (e.g. by sending the screenshot).

### Projects and sharing

1. A #tag sets the project (#home, #job); a new tag creates a new project; no tag = Other.
2. #us marks an item as shared. My husband can see, edit and tick off shared items. Everything else is mine only.
3. Urgent items are marked with "!". No extra alert.
4. No snooze: overdue items stay at the top until I tick them off or change the date.

### Daily digest

1. Sent at 08:00, Monday to Friday. None at weekends.
2. If nothing is due: a one-line message ("Nothing due today").
3. Order: Now (overdue + due today, all projects) listed in full, then one line per project with its count and next item.
4. Friday's digest adds a numbered Backlog section to review.
5. My husband gets his own digest: his own items plus shared items.
6. Items due at the weekend appear in Friday's digest. If I give a time ("Saturday morning" = 09:00), I also get one reminder message at that time.
7. Proposed: if the machine was asleep at 08:00, the digest is sent as soon as it wakes, marked "late", up to 12:00 (to confirm).

### Screenshots

1. I can attach a screenshot when adding or updating an item.
2. The AI reads it and pulls out 2–3 key facts (deadline, reference, what to do), which I confirm.
3. The screenshot is kept, stored encrypted, so I can look at it again.
4. It is deleted automatically when the item is done.

### Calendar

1. A separate "Life admin" calendar on my iPhone, which I can show or hide like my other calendars.
2. Shared with my husband so it appears on his phone too.
3. One-way: the assistant adds and updates events; moving an event in Calendar doesn't change the item.
4. Only dated items; titles only, no notes or screenshots.

### Archive

Done items are kept forever, in a Done archive I can search.

## The chat bot

Each of us has a private chat with the same bot (not a group chat).

### Messages and commands

| I send | What the bot does |
|---|---|
| Free text, e.g. `#home #us send back router by 14/11` | Creates the item, replies with one line of what it understood |
| Text with no date | Adds it to the backlog and says so |
| `pay cleaner weekly on Saturday morning` | Creates a weekly item; reminder Saturday 09:00; the next one is set when I tick it off |
| `remind me every January to book the health check` | Creates a yearly item that appears in January's first digest |
| `remind me daily` (with an item) or `daily 5` | Changes that item's reminder to daily until done |
| Voice note | Transcribes, then treats it as text |
| Screenshot (with or without text) | Reads it, attaches it to the matching item or creates one, replies with the key facts |
| `done 3` / `done router` | Ticks it off; asks if unsure which item |
| `urgent 5` | Marks item 5 urgent |
| `move 5 to Fri` | Changes the due date |
| `backlog 12` | Moves item 12 to the backlog |
| `schedule 3 Tue` / `drop 4` | Backlog review actions |
| `show router` | Re-sends the screenshot and key facts |
| `/list` | Full numbered list: Now, then each project, one line per item |
| `/backlog` | Numbered backlog list |
| `/recurring` | Recurring items and their next dates |
| `/done` | Recently completed items (archive) |

### Functional requirements

1. One line per item in lists: number, "!" if urgent, title, short date, "waiting" or "(us)" where relevant.
2. Numbers stay the same within a day so "done 14" is reliable.
3. No limit on list length; if a list is too long for one message, it splits into consecutive messages (the size limit is set per chat provider).
4. The bot always confirms what it understood in one line, so I can correct it ("no, #job").
5. If the bot can't tell which item I mean, it asks, rather than guessing.
6. Only my husband's and my Matrix accounts (a fixed list of Matrix IDs) can use the bot; anyone else gets no reply.
7. Commands (`/list`, `done 3` etc.) work without the AI. Once the AI is added, the rules parser handles free text whenever the AI is slow or down.

### Swappable parts (modularity)

1. All Matrix-specific code lives in one adapter. The adapter turns incoming messages into one standard shape (who sent it, which conversation, text, attachments) and sends replies.
2. Rules, data, digests and the text reader never use Matrix code directly. Switching provider means writing a new adapter, not changing the rules.
3. User identity is mapped from the provider's ID (e.g. `@name:matrix.org`) to our own user ID, so stored items don't depend on the provider.
4. Replies use plain text plus attachments. Provider-only features (reactions, edits, threads) are optional extras inside the adapter.
5. The server address and bot account are settings, so moving from matrix.org to our own Matrix server needs no code change.
6. The text reader (rules parser, Claude or Ollama) sits behind one slot and always returns the same item format. It is chosen with one setting.
7. The host (Mac Mini, Miki's laptop, later a server) is swappable: the same code runs on each, all data lives in one folder, and moving means copying that folder. Only one host runs the bot at a time.

### Acceptance criteria

- [ ] Adding `#job review contract by Fri` creates a to-do in Job offer, due this Friday.
- [ ] Adding `#home frame prints` with no date puts it in the backlog.
- [ ] Ticking an item off deletes its screenshot.
- [ ] The digest arrives at 08:00 Mon–Fri, never at weekends, with a one-liner when nothing is due.
- [ ] Friday's digest includes the numbered backlog.
- [ ] An item tagged #us appears for my husband and he can tick it off.
- [ ] Dated items appear in the "Life admin" calendar on both phones.
- [ ] Forwarding the router email screenshot fills in the due date and reference; `show router` sends it back.
- [ ] A voice note is turned into an item.
- [ ] `done 3` ticks off item 3 and removes it from lists.
- [ ] `/list` with 30+ items shows every item, grouped and numbered.
- [ ] A stranger messaging the bot gets nothing back.
- [ ] The core rules pass their tests with a fake chat provider and a fake text reader, without Matrix or a real model.
- [ ] Moving the bot from one host to another by copying the data folder keeps all items and needs no new login.

### Non-functional requirements

How well it must work, and the limits it must work within.

| Area | Requirement |
|---|---|
| Users | Two: me and my husband. No sign-up for anyone else. |
| Platform | iPhone for both of us, Element app. |
| Privacy | Store only what's needed: title, dates, status, short key facts. No Meta products. Health items kept vague by default ("Annual check-up"). |
| Encryption | Screenshots and data stored encrypted on the host ("encrypted storage"). End-to-end encryption in chat is not required for the first increments; Matrix supports it and it is planned as its own increment. |
| Modularity | Chat provider, text reader and host swappable (see above). |
| Cost | Under £5/month in total (hosting, AI use, any accounts). Flag anything that breaks this. |
| Reliability | The 08:00 digest arrives every weekday; a missed digest is a bug. |
| Speed | Adding an item, including the AI reading it, takes under 10 seconds. |
| Dates | UK format (DD/MM/YYYY), UK time zone, handles clock changes. |
| Language | Plain English replies, short, no chatty filler. |
| Data | Nothing lost if the bot restarts. A way to export everything (e.g. to a file) if we ever move it. |
| Testing | Every increment ships with automated tests, written before the code. |

## v1 scope and definition of done

v1 is the first version we build and start using.

Definition of done for v1: my husband and I use it daily for two weeks without falling back to another list, the 08:00 digest arrives every weekday, and all acceptance criteria pass. New ideas during those two weeks go on a v2 list, not into the build.

## Open questions to discuss

| # | Question | Why it matters | Status |
|---|---|---|---|
| 1 | Chat bot or app? | Decides what we build | Decided: Matrix chat bot, provider swappable |
| 2 | Where does the data live and where does the bot run: laptop, home computer or small rented server (VPS)? | Digest needs a machine that is always on | Decided: Mac Mini or Miki's laptop first (swappable, one at a time), VPS as the last step; which VPS still open |
| 3 | What do we actually need to pay for, and does it fit under £5/month? | Hosting, AI use | Open |
| 4 | How do we set up encrypted storage for screenshots and data, and is it on by default? | My privacy requirement | Open |
| 5 | Matrix privacy: matrix.org sees who talks to whom and when; screenshots stay in chat history. Do we turn on end-to-end encryption, self-host later, or set message retention? | I want exposure kept small | Open: encryption planned as its own increment |
| 6 | Calendar: how do we create a shared "Life admin" calendar both phones see, and how can the bot add to it safely (subscription link vs storing an iCloud login)? | The bot can't reach my phone's calendar directly | Proposed: read-only subscription feed (.ics), reached via Tailscale until the VPS; confirm after testing how fast iPhones refresh it |
| 7 | Which AI service reads screenshots, voice notes and turns my text into items, and what does it cost? | Core feature; affects cost and privacy | Proposed: rules parser first; then Claude API or Ollama, swappable; cost to check |
| 8 | Login for two people | Only we two should get in | Decided: fixed list of Matrix IDs |
| 9 | Voice commands to change items ("move router to Friday"): v1 or later? | Voice notes that add items are already in | Open |
| 10 | How do we test the 08:00 digest without waiting for 08:00? | Reliability is a requirement | Decided: the clock is passed into the code, so tests can set any day and time |
| 11 | How do I build this myself in Claude Code: what order, and where do you want to review? | It's a learning project | Decided: see [build-plan.md](build-plan.md) |
| 12 | Missed digest when the machine was asleep: send late (until 12:00) or skip? | Laptop hosting makes this likely | Open (proposal: send late) |

## Assumptions to confirm

I filled these gaps myself; each is a one-line check.

1. My husband's digest also arrives at 08:00 Mon–Fri and contains shared items plus his own items.
2. Items he adds without #us are private to him, the same as mine.
3. Recurring items are added in plain words ("#health annual check-up every December"), and ticking one off sets the next date.
4. A waiting-on item with no deadline stays in the waiting state until I add a date; nudges continue weekly.
5. "Daily until done" reminders are a per-item override I ask for; the default for to-dos is the 3-day digest window.
6. Health items are not hidden from the shared calendar, since only we two see it.
