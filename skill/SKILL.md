---
name: consolehub
description: Talk to the other AI agents running beside you inside ConsoleHub — list them, start new ones, send them work, collect their answers, and show files to the user at a given line. Use whenever the CONSOLEHUB_SESSION environment variable is set, or when the user asks an agent to hand work to another agent, to reach "the other session/tab/console", to open a file in front of them, or mentions ConsoleHub / консольхаб / соседнюю сессию / другого агента в соседней вкладке.
---

# ConsoleHub — working with the agents beside you

ConsoleHub is a Windows terminal that hosts several AI agents in tabs. Every
session gets an id and a local HTTP API, so agents can drive each other without
the user relaying messages by hand.

## Am I inside it?

Two environment variables are set for every session it starts:

```
CONSOLEHUB_SESSION   your own id, e.g. "consolehub-2"
CONSOLEHUB_API       base address, e.g. "http://127.0.0.1:7788"
```

If `CONSOLEHUB_SESSION` is empty you are not in ConsoleHub — nothing here
applies. Never hardcode the port; read it from `CONSOLEHUB_API`.

**This works over SSH too.** If you are an agent running on a server the user
opened as a remote project, the hub is reachable at the same `CONSOLEHUB_API` —
it is carried onto the host over the connection (a reverse tunnel), so
`127.0.0.1:7788` on the server reaches the hub on the user's machine. You list,
message and answer other agents, local and remote, exactly as below. Two things
to keep in mind on a server:

- Use `curl`, not PowerShell. The examples below give the `curl` form; it runs
  on the server and on Windows alike.
- `POST /api/open` needs your session id (or a host) so the file is read from
  the server, not looked for on the user's machine — see "Showing something to
  the user".

Every example here uses `curl` so it works wherever you are. The same calls work
from PowerShell with `Invoke-RestMethod` if you prefer, when you are local.

## Ids, and how sessions relate

An id is the slug of the folder name; a second session in the same folder gets
`-2`, a third `-3`, and so on:

```
tolmach       claude   F:\AISpeach\Tolmach
tolmach-2     codex    F:\AISpeach\Tolmach      <- same project, different agent
consolehub    claude   F:\AISpeach\ConsoleHub
```

Agents in the same project are the ones sharing your `Cwd`. Rather than listing
every session and filtering, ask for your own surroundings directly:

```bash
curl -s "$CONSOLEHUB_API/api/sessions/$CONSOLEHUB_SESSION/peers"
```

The reply is `{ project, me, peers, helpers, files }`:

- `project` — the folder you are working in
- `me` — your own entry
- `peers` — other agents in this same folder
- `helpers` — sessions you started yourself, wherever they are
- `files` — files the user has open under you (editor tabs, not terminals)

`helpers` only fills in when you name yourself at the time of opening, with
`"by": "<your id>"` — see below. Reach for `GET /api/sessions` when you genuinely
need the whole machine.

## The API

Base address is `$CONSOLEHUB_API`. Everything takes and returns JSON.

### Sessions

| Call | Body / query | What it does |
|---|---|---|
| `GET /api/sessions` | — | Every session on the machine: `{Id,Title,Cwd,Kind,Pid,Ready,Active,Transient,OpenedBy}`. |
| `GET /api/sessions/{id}/peers` | — | **Start here.** `{project, me, peers, helpers}` — your folder and who is in it. |
| `POST /api/sessions` | `{"cwd":"C:/dir","kind":"claude","by":"<your id>"}` | Starts an agent, returns once it accepts input. Opens in the background. |
| `POST /api/sessions/{id}/focus` | — | Brings that session to the front of the window. |
| `POST /api/sessions/{id}/signal` | `{"message":"..."}` | Pulses that session's project tab green until the user goes there. Attention without stealing focus. |
| `POST /api/sessions/{id}/send` | `{"text":"..."}` | Types into that session, exactly as the user would. |
| `POST /api/sessions/{id}/ask` | `{"text":"...","waitMs":30000}` | Sends, waits for the output to settle, returns what appeared. |
| `GET /api/sessions/{id}/ready` | `?waitMs=45000` | Blocks until that session accepts input. |
| `GET /api/sessions/{id}/output` | `?chars=4000` | That session's screen, ANSI stripped. |
| `DELETE /api/sessions/{id}` | — | Closes it and kills the process. |
| `GET /api/runner` | `?session={id}` (or `?cwd=&host=`) | The project's saved Run commands: `{commands:[...]}`. |
| `POST /api/runner` | `{"session":"{id}","command":{...}}` | Adds a saved command to that project's Run panel. `{"remove":"Name"}` deletes one. |
| `POST /api/run` | `{"session":"{id}","name":"Build"}` or `{...,"commands":[...]}` | Runs a sequence in the project — opens a session (the SSH console for a remote project) and types it in. Returns `{started:"<id>"}`. |

**Open straight into a pane.** Do NOT open sessions and then move them — pass the
placement at open time, so each lands where it belongs in one call:

```json
{"cwd":"F:/proj","kind":"far","force":"true","split":"right","near":"<a tab>"}
```

`split` is `left`|`right`|`top`|`bottom`; `near` is an existing session id (or a
file path) to split away from. Chain it to build a row: open the first normally,
then open each next one with `near` set to the one before.

```bash
# a shell, then Claude to its right, then Far to Claude's right
A=$(curl -s "$CONSOLEHUB_API/api/sessions" -H 'Content-Type: application/json' \
     -d '{"cwd":"F:/proj","kind":"shell","force":"true"}' | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
B=$(curl -s "$CONSOLEHUB_API/api/sessions" -H 'Content-Type: application/json' \
     -d "{\"cwd\":\"F:/proj\",\"kind\":\"claude\",\"force\":\"true\",\"split\":\"right\",\"near\":\"$A\"}" | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
curl -s "$CONSOLEHUB_API/api/sessions" -H 'Content-Type: application/json' \
     -d "{\"cwd\":\"F:/proj\",\"kind\":\"far\",\"force\":\"true\",\"split\":\"right\",\"near\":\"$B\"}"
```

`POST /api/sessions` on a folder that already has a session returns the existing
one with `"alreadyOpen": true` instead of starting a second. Pass
`"force": "true"` when you deliberately want another agent in the same folder —
that is how a project ends up with both Claude and Codex.

Add `"waitReady": "false"` if you only want it launched and do not intend to
talk to it straight away.

### Who holds the screen

A session you start stays **in the background**. The user keeps watching
whichever session they were on — normally the one that gave you the task — and
spawning three helpers no longer drags the window through three tab switches.

- `"activate": "true"` on `POST /api/sessions` opens the new session in front,
  for when the user is meant to go there and watch.
- `POST /api/sessions/{id}/focus` moves the view afterwards: on your own id when
  you have something worth looking at, on someone else's when you are handing
  the conversation over.
- `Active` in the session list tells you who is on screen right now — check it
  before pulling the view away from the user, and leave it alone if you are
  working in the background.

Grabbing focus interrupts someone. Do it when the user needs to see something,
not to announce that you finished.

- `POST /api/sessions/{id}/signal` is the quiet alternative: it makes the
  session's **project tab pulse green** and leaves the view where it is. Use it
  to say "I'm done, come look when you can" — the pulse stops the moment the
  user opens that project or clicks anything inside it. `focus` yanks the pane to
  the front; `signal` waits to be noticed. When a task finishes and nobody is
  watching that project, `signal` is almost always what you want.

```bash
# finished a build in the background — nudge the user without interrupting them
curl -s -X POST "$CONSOLEHUB_API/api/sessions/$CONSOLEHUB_SESSION/signal" \
  -H 'Content-Type: application/json' -d '{"message":"tests are green"}'
```

### Helpers that leave no trace

`"transient": "true"` on `POST /api/sessions` opens a session as a helper. This
is not a label — the session really behaves differently: Claude Code writes no
transcript for it, so it never appears in the user's history and `/resume`
cannot bring it back.

```json
{"cwd": "C:/dir", "kind": "claude", "transient": "true"}
```

Use it for work you commissioned and will collect yourself: a second pair of
eyes, a build to run, a file to summarise. The user's own sessions, and anything
they may want to reopen and read later, must stay ordinary — a lost transcript
cannot be recovered afterwards.

`Transient` in the session list says which sessions are helpers.

**kinds**: `claude`, `codex`, `agy`, `opencode`, `kilo`, `qwen`, `grok`, `amp`,
`crush`, `goose`, `aider`, `cline`, `copilot`, `shell`, `cmdshell`, `gitbash`,
`far`, `yazi`, `mc`. Only the installed ones appear in `GET /api/projects`
under `agents`.

### Projects

| Call | Body | What it does |
|---|---|---|
| `GET /api/projects` | — | `{projects:[{name,path,open,sessionId,agent}], agents:[...]}` |
| `POST /api/projects` | `{"path":"C:/dir","name":"Label"}` | Adds a folder to the user's list. |
| `DELETE /api/projects/{name}` | — | Removes it from the list; nothing on disk is touched. |

`sessionId` here names only **one** session per folder. When several agents share
a project, enumerate `GET /api/sessions` and match on `Cwd` instead.

### The Run panel — saved command sequences

Each project has a **Run** tab with saved command sequences the user launches
with a click. You can read, add and remove those, and trigger one. A saved
command is `{"name","group?","shell?","commands":[...],"env?":{},"autoClose?":false,"urls?":[],"programs?":[]}`
— `commands` are shell lines, `group` nests with `/`.

Running one opens a session in the project and types the commands in. For a
**remote (SSH) project the commands run on the server**, in that connection.

```bash
# the caller's own project is found from its session id
S=$CONSOLEHUB_SESSION

# save a reusable command for the user
curl -s -X POST "$CONSOLEHUB_API/api/runner" -H 'Content-Type: application/json' \
  -d "{\"session\":\"$S\",\"command\":{\"name\":\"Build & test\",\"group\":\"Dev\",\"commands\":[\"npm install\",\"npm test\"]}}"

# run it now (returns {"started":"<session id>"})
curl -s -X POST "$CONSOLEHUB_API/api/run" -H 'Content-Type: application/json' \
  -d "{\"session\":\"$S\",\"name\":\"Build & test\"}"

# or run an ad-hoc sequence without saving it
curl -s -X POST "$CONSOLEHUB_API/api/run" -H 'Content-Type: application/json' \
  -d "{\"session\":\"$S\",\"commands\":[\"git pull\",\"docker compose up -d\"],\"autoClose\":true}"
```

Prefer saving a command the user will want again, and running it, over typing a
long sequence into a session by hand. `${input:Label}` placeholders in a saved
command are prompted for in the window (not over the API), so an API run leaves
them literal — pass a fully-resolved `commands` list instead when running headless.

### Files the user has open

Sessions are not the whole picture: the hub also has an editor, and those tabs
are invisible in the session list.

```
GET /api/files   ->  { files: [{ Path, Session, Dirty, Active }] }
```

`Session` is the session the tab hangs under, `Active` marks the one on screen,
and `Dirty` means there are unsaved edits — so the file on disk is **not** what
the user is reading. Check it before drawing conclusions from a file's contents,
and before rewriting a file someone is midway through editing.

### Showing something to the user

```
POST /api/open   {"path":"C:/dir/file.cs","line":42,"endLine":48,"column":1}
```

Opens the file in ConsoleHub's own editor, scrolls to the line and highlights it
briefly. Prefer this over describing a location in prose — when the user asks
what you changed, open the file at the line and say so in one sentence.

```bash
curl -s "$CONSOLEHUB_API/api/open" \
  -H 'Content-Type: application/json' \
  -d "{\"path\":\"$PWD/file.cs\",\"line\":42}"
```

**If you are running on a server (over SSH), add your own session id so the file
is read from the host you are on, not from the user's machine:**

```bash
curl -s "$CONSOLEHUB_API/api/open" \
  -H 'Content-Type: application/json' \
  -d "{\"path\":\"/opt/app/main.py\",\"line\":42,\"session\":\"$CONSOLEHUB_SESSION\"}"
```

Your id is in `CONSOLEHUB_SESSION`. Without it a path like `/opt/app/main.py`
would be looked for on the user's computer, where it does not exist. The hub
reads the file over the same connection you came in on and puts it in front of
the user — a file you just wrote on the server opens on their screen. (A `host`
field works too if you know the host name.)

## Arranging the panes

The second row of tabs is a grid: each project's consoles and files live in
panes you can split and rearrange. You can both see it and change it.

```
GET  /api/layout
```

Returns `{ projects: { "<cwd>": <tree>, ... }, active }`. A tree is nested:

- a **pane** — `{ type:"pane", tabs:[ ... ] }` — holds tabs, each either
  `{ type:"terminal", session, agent, cwd, host, active }` or
  `{ type:"file", path, host, active }`
- a **split** — `{ type:"split", dir:"horizontal"|"vertical", panes:[ ... ] }` —
  holds panes or further splits side by side

So you can tell which session sits where before touching anything.

```bash
curl -s "$CONSOLEHUB_API/api/layout"
```

To change it:

```
POST /api/layout   { "action": "...", ... }
```

| action | fields | effect |
|---|---|---|
| `focus` | `session` or `path` | bring that tab to the front of its pane |
| `split` | `session`/`path`, `side` | move it into a new pane on that side |
| `duplicate` | `session`/`path`, `side` | a second view of it in a new pane (a file shares its text; a console opens another of the same kind) |
| `move` | `session`/`path`, `into` | move it into the pane holding `into` |

`side` is `left`, `right`, `top` or `bottom` (default `right`). A tab is named by
its session id (for a console) or its file path.

Example — put a console and Far side by side. Open them first, then split:

```bash
# a shell and Far in the same project
curl -s "$CONSOLEHUB_API/api/sessions" -H 'Content-Type: application/json' \
  -d '{"cwd":"F:/proj","kind":"shell","force":"true"}'
curl -s "$CONSOLEHUB_API/api/sessions" -H 'Content-Type: application/json' \
  -d '{"cwd":"F:/proj","kind":"far","force":"true"}'
# they land in one pane; move Far into its own pane to the right
curl -s "$CONSOLEHUB_API/api/layout" -H 'Content-Type: application/json' \
  -d '{"action":"split","session":"far","side":"right"}'
```

A pane holding a single tab cannot be split off itself — open or move a second
tab in first, or it just stays put.

## Talking to another agent

`send` types into their prompt; they have no idea who you are unless you say so.
Always name yourself and give a reply address:

```bash
curl -s "$CONSOLEHUB_API/api/sessions/tolmach-2/send" \
  -H 'Content-Type: application/json' \
  -d "{\"text\":\"This is $CONSOLEHUB_SESSION. Please <task>. When done, POST your answer to /api/sessions/$CONSOLEHUB_SESSION/send starting with 'ANSWER from $CONSOLEHUB_SESSION:'\"}"
```

Then **carry on with your own work**. Do not poll and do not sleep waiting —
their reply arrives in your session as an ordinary user message.

When a message names a sender and a reply address, treat it as a request: do the
work and post the result back the same way. Answer even on failure — "could not
do it, because ..." beats silence.

### ask vs send

`ask` is convenient for one-liners, but the answer is scraped off a terminal
screen that redraws over itself, so it comes back rough and sometimes truncated.
For anything structured, use `send` and have the other agent reply into your
session.

## Manners

- Check `GET /api/sessions` before starting anything — the agent you need is
  often already running.
- Reach for other agents only when the task genuinely needs them: a second
  opinion, work in a repo you are not in, something long you want run in
  parallel. Otherwise just do the work.
- Do not close sessions you did not open.
- The live reference is always at `GET /api/help`, served by the running build —
  consult it if something here disagrees with reality.

## Maintenance

This skill mirrors ConsoleHub's API, which lives in `app/HubApi.cs`,
`app/Bootstrap.cs` and `app/SessionHost.cs` of the ConsoleHub repository. When
that API gains, loses or changes an endpoint, update this file in the same
change.
