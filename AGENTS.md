# AGENTS.md — information for AI agents working in this repo

Read this file fully before changing anything. `README.md` is the user documentation; keep both up to date when behaviour changes.

## What this project is

Talk Timer is a no-frills countdown timer for talks, in plain HTML files. A speaker sees a big countdown on a tablet or laptop. The background is green, turns to a warning colour at a configured threshold, and red after zero (the countdown continues in negative numbers).

Since November 2026 it also has a remote control: a control-room page changes the time on a tablet that the operator cannot see, and the tablet reports back.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The timer: configuration form, countdown, and the tablet side of the remote control. All CSS and JavaScript are inline |
| `admin.html` | The control-room page of the remote control. Self-contained, plain JavaScript, no library |
| `time.html` | A static green "you have time" screen (Bulgarian text). Uses Zepto only for sizing and fullscreen |
| `lib/vendor/zepto/zepto.min.js` | Zepto v1.2.0, default build (zepto, event, ajax, form, ie), from the npm package `zepto@1.2.0`. Zepto has had no release since 2016 |
| `timer.png` | Screenshot used by the README and the OpenGraph tags |
| `README.md`, `LICENSE.md` | User documentation and the MIT license |
| `.gitignore` | Ignores `CLAUDE.md` (a private, per-developer file) and macOS `.DS_Store` files |

## Publishing — read this before any commit

- There is **no CI and no build step**.
- The site is GitHub Pages. **Every push to the `gh-pages` branch of `mitio/talktimer` goes live** on <https://ddimitrov.name/talktimer/> within about 1–10 minutes. Pages are cached for 10 minutes.
- **Every committed file becomes public** at that address, including this one. Never commit secrets, private notes or personal paths. Add files by name (`git add <file>`), never `git add -A` or `git add .`.
- The custom domain is not configured in this repo (there is no `CNAME` file here).
- Push only when the repo owner or the user explicitly asks for it.
- To undo a release: `git revert <commit>` and push.

## Code rules

- **No build tools, no package manager, no framework.** A file must work when opened directly from disk.
- **The timer must work offline** when opened without `room`. Do not load anything from a CDN; vendor it under `lib/vendor/`.
- JavaScript is written in the old style used throughout: `var`, `function () {}`, no arrow functions, no `let`/`const`, no modules. Browser APIs such as `fetch`, `EventSource` and `Promise` are used only by the remote control, behind a feature check.
- `index.html` uses Zepto (`$`). `admin.html` uses no library.
- Two-space indentation. Keep it simple: the README promises "all contributions are welcome, provided we keep it simple".
- Put untrusted text into the page with `textContent` / `.text()`, never as HTML.

## Link parameters of `index.html`

| Link | Meaning |
| --- | --- |
| `?5` | Start 5 minutes at once |
| `?time=5&time-in=minutes&warn-at=1&warn-time-in=minutes&hide-seconds=false&font-height-percentage=70` | Full preset, starts at once |
| `?room=1234` | Remote-control mode: listen to room 1234. Can be combined with the preset parameters |

Form settings are stored in `localStorage` under keys starting with `_talk_timer_`. The remote control uses the separate keys `_talktimer_remote` (tablet) and `_talktimer_admin_room` (admin page); do not give them the `_talk_timer_` prefix, because `loadState` reads every key with that prefix as a form field.

## Remote control

### Transport

Both pages talk to the public service [ntfy.sh](https://ntfy.sh) straight from the browser. There is no server, account or API key.

- A room is one ntfy topic: `talktimer-room-<room number>`. The room number is 1–12 digits.
- Publish: `POST https://ntfy.sh/<topic>` with the JSON message as the body.
- Catch-up: `GET https://ntfy.sh/<topic>/json?poll=1&since=<all | unix time>` returns stored messages, one JSON object per line.
- Live: `EventSource` on `https://ntfy.sh/<topic>/sse?since=…`. The connection is reopened after an error, after 120 seconds of silence, and when the browser comes back online.
- Quota: `GET https://ntfy.sh/v1/account` returns `stats.messages_remaining`, `limits.messages` and `stats.reset_at` for the caller's IP address.

### Messages

Both pages listen to the same topic and ignore their own kind of message.

| `type` | Sent by | Fields | Receiver's action |
| --- | --- | --- | --- |
| `set` | admin | `id`, `seconds`, `warn` (seconds), `hideSeconds` (`true` when the time was typed in minutes) | Tablet restarts the countdown from `seconds`, hides or shows the seconds, then sends `status` with `ack: id`. A `set` without `hideSeconds` leaves the seconds setting as it is |
| `ping` | admin | `id` | Tablet sends `status` with `ack: id` and changes nothing |
| `room` | admin | `id`, `room` | Tablet joins the new room, rewrites its own link, and sends `status` with `ack: id` **in the new room** |
| `status` | tablet | `ack` (command id or `null`), `room`, `state` (`waiting`, `running`, `warning`, `overtime`), `total`, `warn`, `hideSeconds`, `remaining` (seconds, `null` when waiting), `left` and `limit` (the tablet's ntfy quota) | Admin shows the confirmation and counts down locally from `remaining` |

The tablet also sends a `status` with `ack: null` once per page load and when someone presses `↺` or `⨉` on it.

### Rules — keep them when you change the code

1. **A new time restarts the countdown from that time.** The operator cannot see the tablet, so the value sent must be the value shown.
2. **Never apply a command twice.** The tablet stores the ntfy id and server time of the last command it handled (`lastId`, `lastTime`). The catch-up request and the stream can both deliver an old command again; both paths check against these values.
3. **Catch-up applies only the newest `set` or `room` command**, and ignores stored `ping` commands.
4. **A reload continues the countdown.** The tablet stores the start moment and total; after a reload it resumes if the countdown started less than 12 hours ago.
5. **The link is the truth for the room.** After a `room` command the tablet rewrites `?room=` with `history.replaceState`. A link to a different room than the stored one resets the stored state, so an old bookmark follows a `room` command that is still stored there. `lastTime` prevents bouncing between two rooms.
6. **No heartbeat and no periodic messages.** Every message counts against the daily limit. The admin page asks with `ping` only on a click and once per page load.
7. **The admin confirms only on a matching `ack`.** No answer within 5 seconds shows "NOT confirmed"; a later answer shows "confirmed (late)".
8. **Without `room`, nothing remote runs** and no request leaves the page.

### Limits of ntfy.sh (checked on 2026-11-19)

| Limit | Value |
| --- | --- |
| Messages per day per IP address | 250 (a rolling 24-hour period) |
| Requests | A burst of 60, refilled at one request per 5 seconds. Reading the quota, catch-up and opening the stream are requests |
| Message retention | 12 hours |
| Access control | None. Topics are public; the room number is the only secret |
| Storing delay | A message published a moment ago may be missing from a catch-up request for a short time |

The admin page shows its own remaining messages and the tablet's, in red below 30. When the limit is reached ntfy.sh answers HTTP 429.

## Testing

There is no test suite in the repo. Test every change by hand or with a browser-automation script before pushing:

1. Serve the folder locally, e.g. `python3 -m http.server 8765 --bind 127.0.0.1`.
2. Open `admin.html?room=<test number>` in one browser window and `index.html?room=<same number>` in another. Use a long random room number for tests.
3. Check at least these cases:

| # | Case | Expected |
| --- | --- | --- |
| 1 | `index.html` with no parameters; start, `↺`, `⨉` | As described in the README |
| 2 | `?5` and the long preset link | The timer starts at once |
| 3 | `?room=N`, nothing sent | "Waiting for the control room", room number shown |
| 4 | Admin sends a time in seconds with a warning | Tablet shows it with seconds within about a second; admin shows "Tablet confirmed" and the same countdown |
| 4b | Admin sends a time in minutes | Tablet and the admin preview show whole minutes only (rounded down); a reload of the tablet keeps the seconds hidden |
| 5 | Admin sends a new time while one runs | Tablet restarts from the new time |
| 6 | Reload the tablet mid-countdown | The countdown continues, it does not restart |
| 7 | Close the tablet, send a time, reopen the tablet | Admin shows "NOT confirmed", then "confirmed (late)" when the tablet applies it |
| 8 | "Check tablet" | Admin shows the tablet's real state |
| 9 | "Move the tablet to this room" | Tablet link is rewritten; confirmation arrives in the new room; a reload stays there |
| 10 | Open the old room link in a fresh browser profile | It follows to the new room |
| 11 | Warning colour and red after zero | On the tablet and in the admin page |
| 12 | `time.html` | Renders and fills the screen |
| 13 | Browser console | No JavaScript errors |

A full automated run sends about 17 messages and about 60 requests. Two runs in a row can hit the request limit; wait a few minutes between them.

## Git

- Set the author per repo (`git config user.name`, `git config user.email`), not globally.
- Small, focused commits with a plain message in the style of the history, e.g. "Add remote control of the timer from a control room".
- No AI attribution in commits, pull requests or code comments.
