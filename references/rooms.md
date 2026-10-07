# Rooms: realtime state inside a hosted service

Rooms give a hosted service what a database cannot: live state shared by
several open connections at once. A chat room, a presence list, a shared
whiteboard, a multiplayer session, a collaborative document: each is one
room, reached by name from the service's fetch handler, holding its own
WebSocket connections, its own storage and its own alarm. Requires the
`service_rooms` permission (Basic and Pro; check `yard me --json` →
`.team_permissions.service_rooms` before building). Without it the API
answers `upgrade_required` and `yard push` restates it with the upgrade link.

- [What a room is](#what-a-room-is)
- [Declaring classes](#declaring-classes)
- [The class contract](#the-class-contract)
- [Connections](#connections)
- [Storage](#storage)
- [Limits](#limits)
- [Allowance and overage](#allowance-and-overage)
- [Lifecycle](#lifecycle)
- [Local development](#local-development)
- [Debugging](#debugging)
- [Testing before users see it](#testing-before-users-see-it)

## What a room is

A room is a named instance of a room class, a class that `_service.js`
exports. Every request for one name reaches the same instance, wherever it
comes from, and the instance handles one event at a time, so there is nothing
to lock and no race between two messages. It wakes on the first request, goes
idle when nothing is happening (its held connections stay open), and wakes
again on the next message, request or alarm. Instance fields do not survive
idling; the room's storage and each connection's attachment do.

**One conversation, game or document is one room, and a service has as many
rooms as it has names.** Never funnel every conversation through one room,
and never split one conversation across several. The fetch handler picks the
room by name (`idFromName("lobby")`), forwards the request, and does nothing
else; the room owns the connections and the state. Rows shared by everything
and kept for good (accounts, history, billing state) still belong in
`env.DB`; the live state of one conversation belongs in its room.

## Declaring classes

Room classes are declared on the service's entry in `.yard/settings.json`,
next to `database_access`:

```json
{
  "version": 8,
  "services": [
    {
      "dir": "chat",
      "name": "chat",
      "url": "/chat",
      "access": "users",
      "rooms": [{ "class": "Room", "binding": "ROOMS" }]
    }
  ]
}
```

- `class`: the name of a class `_service.js` exports. Matches
  `^[A-Z][A-Za-z0-9_]{0,63}$`. At most 10 classes per service.
- `binding`: the name the fetch handler reaches the class through, as
  `env.<binding>`. Optional; defaults to the class name in upper snake case
  (`Room` becomes `ROOM`, `ChatRoom` becomes `CHAT_ROOM`). Follows the
  secret-name grammar (`^[A-Z][A-Z0-9_]{0,63}$`), may not be `DB` or
  `ASSETS`, and may not collide with a secret's name.

`yard service init <name> --realtime` writes this entry for you, with a
`Room` class bound as `ROOMS`, a working `_service.js` and a minimal
WebSocket client in `index.html`, and no migration. Declaring a class by hand
is an edit to settings.json plus a `yard push`, like every other service
setting.

**Every declared class must be exported by `_service.js`.** A declared class
that is not exported answers every request through its binding with a 500
naming the class. `yard service check` warns about it before you push.

## The class contract

A class is plain JavaScript with a fixed set of methods. Yard constructs it
and calls them; you never instantiate the class yourself.

| Method                                       | Called when                                                                                                                                                              |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `constructor(ctx, env)`                      | The room wakes (first request, or after idling). `ctx` is the room's context: connections, storage, alarm. `env` is the fetch handler's `env`, secrets and `DB` included. |
| `fetch(request)`                             | The fetch handler forwarded a request to this room. Return a `Response`; for a WebSocket upgrade, a 101 carrying the client end.                                         |
| `webSocketMessage(ws, message)`              | A held connection sent a message (`string` or `ArrayBuffer`).                                                                                                            |
| `webSocketClose(ws, code, reason, wasClean)` | A held connection closed, on either side.                                                                                                                                |
| `webSocketError(ws, error)`                  | A held connection failed.                                                                                                                                                |
| `alarm()`                                    | The time set with `ctx.storage.setAlarm(ts)` arrived. Runs with or without connections open; retried if it throws.                                                       |

A complete minimal `Room`: the fetch handler routes `/ws?room=<name>` to the
room of that name, the room accepts the connection, broadcasts every message
to everyone else, and keeps a presence count in its storage.

```js
// chat/_service.js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === "/ws") {
      const name = url.searchParams.get("room") ?? "lobby";
      const room = env.ROOMS.get(env.ROOMS.idFromName(name));
      return room.fetch(request);
    }
    return env.ASSETS.fetch(request);
  },
};

export class Room {
  constructor(ctx, env) {
    this.ctx = ctx;
    this.env = env;
  }

  async fetch(request) {
    if (request.headers.get("Upgrade") !== "websocket") {
      return new Response("Expected a WebSocket", { status: 426 });
    }
    const user = request.headers.get("X-Yard-User-Id") ?? "anonymous";
    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);
    this.ctx.acceptWebSocket(server, [user]);
    server.serializeAttachment({ user, joined: Date.now() });
    await this.presence();
    return new Response(null, { status: 101, webSocket: client });
  }

  async webSocketMessage(ws, message) {
    const { user } = ws.deserializeAttachment();
    this.broadcast({ type: "message", from: user, text: String(message) }, ws);
  }

  async webSocketClose(ws, code, reason, wasClean) {
    await this.presence(ws);
  }

  async webSocketError(ws, error) {
    await this.presence(ws);
  }

  // Count the held connections, store the count, and tell everyone.
  async presence(leaving) {
    const here = this.ctx.getWebSockets().filter((ws) => ws !== leaving);
    await this.ctx.storage.put("present", here.length);
    this.broadcast({ type: "presence", count: here.length }, leaving);
  }

  broadcast(payload, except) {
    const text = JSON.stringify(payload);
    for (const ws of this.ctx.getWebSockets()) {
      if (ws === except) continue;
      try {
        ws.send(text);
      } catch {}
    }
  }
}
```

The frontend connects with a URL built relative to the page, never
root-absolute, so it works under any mount path and inside a sandbox:

```js
// index.html, served from the service's mount
const url = new URL("ws?room=lobby", location.href);
url.protocol = url.protocol === "https:" ? "wss:" : "ws:";
let ws;
function connect() {
  ws = new WebSocket(url);
  ws.onmessage = (e) => render(JSON.parse(e.data));
  ws.onclose = () => setTimeout(connect, 1000); // every close is expected; reconnect
}
connect();
```

## Connections

- **`ctx.acceptWebSocket(ws, tags)`** hands the server end of the pair to
  Yard. The room may then go idle between messages while the connection
  stays open (hibernation), so an idle room costs nothing; the next message
  wakes it and arrives in `webSocketMessage`. Never attach
  `ws.addEventListener(...)` to a held connection: its events go to the class
  methods, not to listeners.
- **Tags** are a few short strings given at accept time. `ctx.getWebSockets()`
  returns every held connection, `ctx.getWebSockets(tag)` those carrying a
  tag, and `ctx.getTags(ws)` a connection's tags. Tag by user id and "send to
  this user" is a lookup, not a loop.
- **Attachments** carry per-connection state across idling:
  `ws.serializeAttachment(value)` stores a small structured value on the
  connection, `ws.deserializeAttachment()` reads it back. Tags are for
  lookup; attachments are for state (display name, cursor colour, last seen).
  Keep an attachment to a few hundred bytes; anything larger goes in storage.
- **Identity** arrives on the upgrade request like any request: `X-Yard-User-Id`,
  `X-Yard-Email`, `X-Yard-Entitlement`, `X-Yard-Tier`, `X-Yard-Tier-Key`,
  `X-Yard-Sandbox`. Read them in the room's `fetch` before accepting, and gate
  per-tier features on `X-Yard-Tier-Key`, not the display name. A socket
  opened from another origin (another project's page included) arrives signed
  out, with no `X-Yard-User-Id`. The service's `access` setting gates the upgrade like
  any other request, so `"access": "users"` keeps visitors who haven't
  bought out of every room with no code.
- **Every session ends after 24 hours.** Yard closes the connection with code
  1000 and reason `Session limit reached`; clients reconnect and carry on.
  Write the client so every close leads to a reconnect, as above, and it also
  covers restarts and network drops.
- **Messages** may be up to 32 MiB inbound. An inbound message counts as one
  twentieth of a request for billing (see below).

## Storage

Each room has its own storage, reachable only from inside that room. It
exists from the room's first request and needs no migration.

- **Key-value:** `await ctx.storage.get("present")`, `put(key, value)`,
  `delete(key)`, and `list({ prefix, limit })`, which returns a `Map`. Values
  are structured-clone-able (objects, arrays, Dates, Maps).
- **SQL:** `ctx.storage.sql.exec(sql, ...params)` runs SQLite inside the
  room and returns a cursor: `.toArray()` for all rows, `.one()` for
  exactly one, or iterate it. Create tables on first use; there is no
  migration stream for rooms:

  ```js
  this.ctx.storage.sql.exec(
    "CREATE TABLE IF NOT EXISTS messages (id INTEGER PRIMARY KEY, sender TEXT, body TEXT, at INTEGER)",
  );
  this.ctx.storage.sql.exec("INSERT INTO messages (sender, body, at) VALUES (?, ?, ?)", user, text, Date.now());
  const recent = this.ctx.storage.sql.exec("SELECT * FROM messages ORDER BY id DESC LIMIT 50").toArray();
  ```

- **Alarms:** `ctx.storage.setAlarm(Date.now() + 60_000)` schedules one call
  to `alarm()`; `getAlarm()` reads the pending time and `deleteAlarm()`
  cancels it. One alarm per room; setting again replaces it. Use it to
  flush a message buffer to `env.DB`, expire an idle room, or end a game
  turn. It fires with no connections open.
- **Cross-room data goes through `env.DB`.** A room cannot read another
  room's storage and the fetch handler cannot read any of it. The room's
  `env` is the fetch handler's, so a service with `database_access: true`
  reaches `env.DB` from inside a room too. Write lasting history there and
  keep the room's storage for what the room needs now.
- **Each sandbox has its own rooms**, the way it has its own database. The
  room named `lobby` in the project and `lobby` in the `preview` sandbox are
  two rooms that never see each other. `X-Yard-Sandbox` on the upgrade
  request says which is serving.

## Limits

| Limit                         | Value                                      |
| ----------------------------- | ------------------------------------------ |
| Storage per room              | 10 GB                                      |
| Inbound message               | 32 MiB                                     |
| CPU per request or message    | 50 ms (time spent awaiting does not count) |
| Requests per second, per room | 1,000, soft                                |
| Classes per service           | 10                                         |
| WebSocket session             | 24 hours, then closed with code 1000       |

The per-room rate is the reason each conversation gets its own room: one busy
room fills its own budget, and the next room has a budget of its own. A
design that sends every user through one name has one budget for everyone.

## Allowance and overage

Basic and Pro each include an allowance per calendar month (UTC). Past it,
each unit is billed on the monthly overage invoice; the rates below are
approximate, and developers see the live numbers on the Usage page of the
dashboard.

| Meter         | Included per month | Overage, about        |
| ------------- | ------------------ | --------------------- |
| Room requests | 1,000,000          | $0.30 per million     |
| Compute       | 400,000 GB-seconds | $25 per million GB-s  |
| Storage       | 2 GiB              | $0.40 per GiB-month   |

- A request is one call into a room: a forwarded HTTP request, a WebSocket
  upgrade, or an alarm. An inbound WebSocket message counts as one twentieth
  of a request, so 20 messages cost what one request does.
- Compute is memory held times the seconds the room is awake handling
  something. An idle room with connections held costs no compute, which is
  why connections must be accepted with `acceptWebSocket`, not kept awake
  with listeners.
- Storage is the total across every room, in the project and its sandboxes.
- Rows read and written are shown on the Usage page but are not billed.

## Lifecycle

- **Adding a class:** declare it under `rooms`, export it from
  `_service.js`, `yard push`, publish. Rooms are created on first use;
  there is nothing to migrate and nothing to provision.
- **Removing a class deletes its data.** Removing an entry from `rooms`
  deletes every room of that class, and all their storage, at the next
  deploy of a release without it. `yard push` prints
  `warning: class Old will be deleted with all its data on deploy` when the
  live deployment has a class the local settings no longer declare. Copy
  anything worth keeping into `env.DB` before that deploy goes live.
- **Renaming a class is a delete plus a create.** `Room` to `ChatRoom`
  deletes every `Room` with its data and starts `ChatRoom` empty. The class
  name is the identity; if only the name in `env` needs to change, change
  `binding` and keep `class`.
- **Removing or renaming a service** leaves its rooms behind: a renamed
  service starts with empty rooms. On the project the old rooms are kept
  for 30 days before deletion, so a service removed or renamed by mistake
  can be put back under its old name without losing them. In a sandbox they
  are deleted at once.
- **Deleting a sandbox or the project** deletes its rooms immediately.

## Local development

`yard dev` runs rooms the way Yard hosts them, with the same `ctx` contract,
identity headers from the chosen persona on the upgrade request, and alarms
that fire.

- A service with room classes ends its banner line with `rooms=Room`. In
  `--json` mode the `ready` event's `services[]` entries gain
  `"rooms": ["Room"]`.
- Room storage lives under `.yard/dev/rooms/<service>/` and survives
  restarts. `yard dev --reset-rooms` deletes `.yard/dev/rooms/` before
  starting; `--reset-db` does not touch rooms, and `--reset-rooms` does
  not touch the database.
- The control panel at `/__yard/dev/` gains a Rooms card: each class, its
  binding, the size stored on disk, and a "Clear stored rooms" button.
  `POST /__yard/dev/api/rooms/reset` does the same from a script.
- **Live connections end when the runtime restarts**, which happens on every
  save. Clients reconnect, exactly as they must for the 24 hour cap when
  hosted.
- To see two users in one room, open the page in two browsers (or one
  private window) and pick a different persona in each; `X-Yard-User-Id`
  on the upgrade request follows the persona cookie.

Full `yard dev` guide: [local-dev.md](local-dev.md).

## Debugging

`yard service logs [--sandbox <slug>] [--service <name>]` shows console output
from room handlers and alarms like any other service output, a few seconds
after the event. Locally, the control panel's logs (or
`GET /__yard/dev/api/logs`) show the same.

Common mistakes, in the order they usually happen:

- **The class is declared but not exported.** Every request through the
  binding answers 500 with a message naming the class. `yard service check`
  warns; the fix is `export class Room` in `_service.js`.
- **Using ports.** `new WebSocketServer({ port })`, the `ws` package,
  socket.io: none of them can run. A WebSocket is accepted by returning a
  101 from the room's `fetch`, nothing listens.
- **Polling the database for liveness.** Presence, typing indicators and
  cursors are room state, held in the room and pushed over the connections
  it holds. A table of "last seen" timestamps polled every second is the
  design rooms replace.
- **Fanning out from the fetch handler.** The fetch handler holds no
  connections and cannot see any. It forwards to the room; the room
  broadcasts. Any loop over connections that is not inside the class is a
  bug.
- **Keeping state in instance fields.** `this.users = []` is reset whenever
  the room wakes. Use storage for room state and attachments for
  per-connection state.
- **A root-absolute WebSocket URL.** `new WebSocket("wss://…/ws")` or
  `"/ws"` breaks under the mount path and in every sandbox. Build it relative
  to the page.
- **Writing `env.DB` on every message.** Each message has a 50 ms CPU budget
  and a row write is not free. Buffer in the room's storage and flush from
  `alarm()`.
- **One room for everything.** One name means one budget of about 1,000
  requests per second for all users. One room per conversation is the rule.

## Testing before users see it

Rooms follow the sandbox flow in
[service-and-database.md](service-and-database.md#testing-before-users-see-it):
`yard push`, `yard releases publish <tag>`, `yard sandbox pin <tag> --sandbox
preview`, then `yard service open --sandbox preview --service chat`. The
sandbox has its own rooms, so the rooms you fill while testing never appear
in the project's, and `yard sandbox visibility public --sandbox preview` lets
testers outside the team join a room from the same URL. Two team members in
the same sandbox room is the quickest end-to-end check; `yard service logs
--sandbox preview` shows what the room did.
