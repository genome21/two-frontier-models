---
name: lan-app
description: Building, extending or debugging a self-hosted app on the home LAN server — anything in ~/workspace with a docker-compose.yml, a FastAPI api/, a React web/, a Caddy front door, or a mobile/ WebView shell. Use when adding a feature to StormScope, the kanban board or the Kimi client; when starting a new app of that shape; when a deploy "worked" but the change is not visible; or when verifying UI or streaming behaviour without a browser to click in. Covers the stack conventions, the failures that have already cost time here, and how to check the running artifact rather than the source.
---

# Self-hosted apps on this server

Three apps share one shape. Follow it unless there is a reason not to, and
write the reason down when there is.

```
web (React + Vite)  ─┐
                     ├─ caddy ── :PORT   (LAN only)
api (FastAPI+SQLite) ┘
mobile/ ── a WebView shell, built in a container
```

| App | Port | What it is |
|---|---|---|
| stormscope | 8100 | severe-weather workstation (`/opt/stormscope`, symlinked) |
| kanban | 8200 | personal board |
| kimi | 8300 | Kimi K3 chat client with sandboxed file access |

New app: take the next free port, `cp -r` the compose/Caddy/mobile scaffolding
from the newest existing one, and check the port really is free
(`ss -tlnp | grep :PORT`) — Plex, Jellyfin and Tunarr also live here.

**`~/workspace` is never itself a git repo.** Each project directory is one, is
initialised on `main`, and gets its own private remote
(`gh repo create <name> --private --source=. --push`). Confirm `.gitignore`
covers `.env` before the first commit, not after.

## The failures that have already happened

Each of these cost real time at least once. Several cost it twice, in two
different apps, because they were not written down the first time.

**A cached `index.html` makes a deploy invisible.** Vite hashes every asset
filename, so the shell is the one file whose name never changes. Serve it
`no-cache` and the hashed assets `immutable` — otherwise a browser tab or the
Android WebView keeps loading a previous build's bundle while its API calls go
to the live server. The app works perfectly; it is just the old app, and
nothing on screen says so. Diagnose it by checking whether nginx logged a
request for the shell at all — if the device is calling `/api/*` but never
fetched `/`, it is serving from cache. The APK also clears its WebView cache
once per `versionCode` for the entries cached before the header existed.

**Caddy buffers SSE unless told not to.** `reverse_proxy` needs
`flush_interval -1` or a streaming response arrives as one block at the end,
which looks exactly like a slow model rather than a proxy setting. Verify by
timing arrivals: N events at N distinct times, not all at one.

**Docker creates a missing bind-mount source as root.** `docker compose up`
before `mobile/build.sh` has ever run leaves `mobile/dist` root-owned, and the
build then fails at the final `cp` with a bare permission error long after a
successful compile. `build.sh` repairs it via a throwaway container rather than
asking for sudo.

**A banner in a `FrameLayout` covers the page.** Gravity TOP or BOTTOM overlays
the WebView — it hid the board's search box in one app and the chat composer in
another. Use a vertical `LinearLayout` root so chrome takes space instead.

**An offer with one action cannot be declined.** A tap-to-enable banner whose
only button enables it is permanent for anyone who does not want it. Give it a
separate dismiss, persist the refusal with a timestamp, and re-offer much
later. Treat a denied permission as a decline, not a reason to ask again.

**XML comments cannot contain `--`.** It fails the Android resource parse with
a message that does not mention comments.

**`navigator.clipboard` does not exist on plain HTTP.** It is gated on a secure
context. Use `document.execCommand("copy")` — deprecated but unrestricted and
working in the Android WebView — and only claim "copied" if something did.

**`<input type="file">` silently does nothing in a bare WebView** until the
shell implements `WebChromeClient.onShowFileChooser`.

## Verify the running artifact, not the source

The rule is in CLAUDE.md; these are the techniques.

```bash
docker compose exec -T web sh -c 'grep -c "SOME NEW STRING" /usr/share/nginx/html/assets/*.js'
docker compose exec -T api grep -c "new_function" /app/app/thing.py
```

Two rebuilds have been lost to skipping this. Note also that
`docker compose up -d web` may recreate `api` if `web` depends on it —
env vars passed inline to an earlier command will be gone.

### Seeing the UI

`chromium` is installed. Headless screenshots work; the snap confinement blocks
`--remote-debugging-port`, and it cannot write to `/tmp`, so write to `$HOME`.

```bash
chromium --headless --disable-gpu --no-sandbox --hide-scrollbars \
  --window-size=414,896 --screenshot=$HOME/shots/phone.png \
  --virtual-time-budget=8000 "http://localhost:PORT/"
```

Then **look at it** — a screenshot has caught a missing stylesheet, a banner
covering a header, and a chart of dots that meant nothing, none of which
failed a build.

To reach a state that needs interaction, serve a **probe page from the app's
own origin**: copy the served `index.html`, append a script, `docker cp` it in
as `_probe.html`, screenshot it, delete it. React only sees a programmatic
input if you use the native setter:

```js
const set = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, "value").set;
set.call(el, "text"); el.dispatchEvent(new Event("input", { bubbles: true }));
```

Regenerate the probe after every rebuild — it embeds the asset hash, and a
stale one renders a blank page.

Deep links are worth adding for their own sake and make states reachable:
`#/c/12`, `#card=7`, `#settings`. They also make Android's Back button close a
sheet instead of the app.

### Testing an upstream you should not spend money on

Run a mock in a container on the compose network and point the service at it
for one command:

```bash
docker run -d --rm --name mock --network <project>_default -v /tmp/mock:/m -w /m \
  python:3.12-slim python mock.py
API_KEY=mock BASE_URL=http://mock:9999/v1 docker compose up -d api
```

Make the mock **stateless** — derive its behaviour from the request, not from a
counter. A stateful mock let separate test runs contaminate each other and
produced results that looked like app bugs. Have it assert on what it receives,
so the test also checks what is being sent.

Scope background helper threads to a single turn. A shared approver thread
answered a different turn's prompt and produced nonsense.

## Conventions worth keeping

**Comments explain why, not what.** Especially the alternative that was
rejected and the reason. The commit messages in these repos are the same:
what changed, and why the obvious approach was wrong.

**Say what was not done.** Every one of these apps shipped with a stated list
of what was unverified — usually "the APK compiles but has never been run".

**Verify, do not assume, for anything external.** Read the vendor's own docs
before building on OpenAI-compatibility assumptions. For Kimi K3 that turned up
fixed `temperature` (so a slider would be a lie), reasoning that cannot be
disabled and streams separately, and cached input at a tenth of the price.

**Measure, do not estimate.** Numbers in these READMEs are measured, and
several claims were corrected after checking.

**When two apps share a bug, check the third.** A `FrameLayout` overlay was
found in one and fixed in two.

## Android shell

Plain Java, no third-party dependencies, built in a container by
`mobile/build.sh` — no JDK or SDK on the host. Keep `targetSdk 34`: apps
targeting 35 are forced edge-to-edge with no opt-out and lay the WebView behind
the system bars. The debug keystore lives under `mobile/.home` and is
gitignored — losing it means no future build can install as an update.

`build.sh` publishes a version manifest read straight from `build.gradle`, and
the app polls it and offers a banner. It hands the APK to the browser rather
than holding `REQUEST_INSTALL_PACKAGES`.

Match the mechanism to the need: StormScope holds a wake lock in a foreground
service because a late tornado warning is useless, while the board uses a
15-minute `JobScheduler` job because a due date is not an emergency. Copying
the first design into the second would have been cargo cult.

## Security posture

LAN-only by deliberate choice: no TLS, no auth, subnet-scoped, never
port-forwarded. Say so in every README, and say what would have to change.

The blast radius differs per app and the README should be honest about it. A
read-only weather service risks nothing; a board accepts writes; the Kimi
client holds an API key that costs money and can write files in a workspace.

If an app gets filesystem access, containment is layered and the layers must be
independent: the API checks the path, a separate sandbox service re-checks it
against its own root *whatever the caller claimed*, and that service's
container has only the permitted tree mounted. Resolve with `realpath` before
comparing, or a symlink inside a permitted folder walks straight out. Refuse
credential-shaped filenames even inside a permitted folder. No shell.

Try to break it and record the attempts — traversal, absolute paths, symlinks,
a caller claiming `/` as its root.

## .env

Never overwrite it; it is gitignored in all three. `.env.example` carries every
key with a comment. Prices, thresholds and addresses belong there rather than
in source, because a hardcoded number quietly becomes a lie.
