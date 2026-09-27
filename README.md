# Real-Time Chat Application

A full-stack real-time chat app: React (Vite) frontend, Node/Express backend,
Socket.io for real-time messaging, MongoDB for persistent chat history.

## Features

- Username-based entry (no password, dummy auth), persisted in `localStorage`
- Instant messaging via Socket.io — no polling
- Persistent chat history in MongoDB, reloaded on refresh
- Server-generated timestamps on every message
- Online user count + list, live-updated on connect/disconnect
- Typing indicators ("Alice is typing…"), debounced, never shown for yourself
- Connection status indicator; input disabled while disconnected
- Loading / empty / error states for chat history
- Centralized backend error handling with consistent JSON responses
- Input validation, length limits, Helmet, scoped CORS, JSON body size limit
- Messages rendered as plain text only (no HTML injection)
- Responsive UI (mobile + desktop)





------
## Tech Stack

**Frontend:** React 18, Vite 5, socket.io-client
**Backend:** Node.js, Express 4, Socket.io 4, Mongoose 8, Helmet, CORS
**Database:** MongoDB

## Project Structure

```
chat-app/
├── backend/
│   ├── src/
│   │   ├── config/database.js        # Mongo connection, retry/backoff
│   │   ├── controllers/messageController.js
│   │   ├── models/Message.js         # Mongoose schema + validation
│   │   ├── routes/messageRoutes.js
│   │   ├── sockets/chatSocket.js     # all Socket.io event handlers
│   │   ├── services/messageService.js # single source of truth for message writes
│   │   ├── middleware/errorHandler.js
│   │   ├── app.js                    # Express app (middleware, routes)
│   │   └── server.js                 # HTTP + Socket.io server bootstrap
│   ├── tests/
│   │   ├── messageService.test.js    # unit tests (node --test)
│   │   └── socket.manual-test.js     # live Socket.io integration script
│   ├── package.json
│   ├── .env.example
│   └── README.md
├── frontend/
│   ├── src/
│   │   ├── components/               # Chat, MessageList, Message, MessageInput,
│   │   │                              # UsernameModal, TypingIndicator, OnlineUsers
│   │   ├── services/                 # api.js (REST), socket.js (Socket.io client)
│   │   ├── hooks/useChat.js          # all chat state + wiring, one hook
│   │   └── App.jsx, main.jsx, index.css
│   ├── package.json
│   ├── .env.example
│   └── README.md
├── README.md
└── .gitignore
```

## Prerequisites

- Node.js 18+
- A running MongoDB instance (local install, Docker, or MongoDB Atlas)

## MongoDB Setup

Any of these work — just point `MONGODB_URI` at it:

- **Local install:** install MongoDB Community Server, then run `mongod`
  (default URI `mongodb://127.0.0.1:27017/realtime-chat` already matches this)
- **Docker:** `docker run -d -p 27017:27017 --name chat-mongo mongo:7`
- **MongoDB Atlas (cloud, free tier):** create a cluster, get the connection
  string, and use that as `MONGODB_URI` (e.g.
  `mongodb+srv://user:pass@cluster.mongodb.net/realtime-chat`)

## Backend Setup

```bash
cd backend
npm install
cp .env.example .env
```

Edit `backend/.env`:

```
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://127.0.0.1:27017/realtime-chat
CLIENT_URL=http://localhost:5173
MAX_MESSAGE_LENGTH=1000
MAX_USERNAME_LENGTH=30
```

## Start Backend

```bash
npm run dev     # with nodemon (auto-restart)
# or
npm start
```

You should see:

```
[server] Listening on port 5000
[server] Allowed client origin (CORS): http://localhost:5173
[database] MongoDB connected
```

If MongoDB isn't reachable yet, the server still starts (it retries 5 times
with backoff) and `GET`/`POST /api/messages` will return `503` until the
database connects — it will not crash.

## Frontend Setup

```bash
cd frontend
npm install
cp .env.example .env
```

Edit `frontend/.env`:

```
VITE_API_URL=http://localhost:5000
VITE_SOCKET_URL=http://localhost:5000
```

## Start Frontend

```bash
npm run dev
```

Open http://localhost:5173.

## API Documentation

All responses use a consistent shape:

```json
{ "success": true, "data": { ... } }
{ "success": false, "message": "..." }
```

### `GET /api/messages`

Returns chat history, oldest first.

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "_id": "66f...",
      "username": "Alice",
      "message": "Hello Bob",
      "timestamp": "2026-09-27T16:58:21.871Z"
    }
  ]
}
```

**Response `503`** (database unavailable):
```json
{ "success": false, "message": "Database is currently unavailable. Please try again shortly." }
```

### `POST /api/messages`

Creates and persists a message. Also broadcasts it to all connected
Socket.io clients (`new_message`), so REST-created messages stay in sync
with the live chat.

**Request:**
```json
{ "username": "Alice", "message": "Hello Bob" }
```

**Response `201`:**
```json
{
  "success": true,
  "data": {
    "_id": "66f...",
    "username": "Alice",
    "message": "Hello Bob",
    "timestamp": "2026-09-27T16:58:21.871Z"
  }
}
```

**Response `400`** (validation error, e.g. empty message, empty username,
message over `MAX_MESSAGE_LENGTH`):
```json
{ "success": false, "message": "Message text is required" }
```

### `GET /health`

Basic liveness/readiness check, including current DB connection state.

## Socket.io Events

| Event | Direction | Payload | Purpose |
|---|---|---|---|
| `connection` | built-in | — | client connects |
| `join_chat` | client → server | `{ username }` | registers this socket's username, adds to online list |
| `send_message` | client → server | `{ message }` (+ ack callback) | validates, saves to MongoDB, broadcasts |
| `new_message` | server → all clients | saved message object | the single source of truth for new messages in the UI |
| `user_typing` | client → server | — | user started typing |
| `user_stop_typing` | client → server | — | user stopped typing (also auto-sent after a debounce timeout) |
| `typing` | server → other clients | `{ username, isTyping }` | relays typing state (never sent back to the typer) |
| `online_users` | server → all clients | `{ count, usernames }` | current online users, recalculated on join/leave |
| `user_disconnected` | server → all clients | `{ username }` | fired once a user's *last* open tab disconnects |
| `error` | server → client | `{ message }` | recoverable errors (bad username, validation failure, DB down) |
| `disconnect` | built-in | — | socket connection closed |

## Architecture

```
React (frontend)
   │  GET /api/messages (once, on load)      → loads history
   │  Socket.io "send_message"               → sends a new message
   ▼
Node.js + Express + Socket.io (backend)
   │  validates input
   ▼
MongoDB (Mongoose)
   │  persists the message
   ▼
Socket.io broadcasts "new_message"
   │
   ├──────────────→ Sender's own client
   ├──────────────→ All other connected clients
```

## Design Decisions

- **Why MongoDB:** flexible schema for chat messages, easy to host
  (Atlas/Docker/local), and Mongoose gives us schema-level validation for
  free (required fields, trimming, max length) in addition to the
  service-layer checks.
- **Why Socket.io:** required by the assignment; it also gives us automatic
  reconnection, acknowledgement callbacks (used for `send_message`), and
  broadcast primitives out of the box, which a raw WebSocket wouldn't.
- **Single message-write path (no duplicates):** both the REST
  `POST /api/messages` controller and the Socket.io `send_message` handler
  call through the *same* `messageService.createMessage()` function, which
  is the only place a `Message` document is ever saved. The primary chat UI
  sends messages over Socket.io only (never also fires the REST POST for
  the same message), so a normal chat message is written to MongoDB exactly
  once. The REST endpoint remains fully functional as its own entry point
  (e.g. for scripts, tests, or non-socket clients) and broadcasts via
  Socket.io too, so all clients still see it — but the frontend chat UI
  itself never calls both for the same user action.
- **Connection handling:** the backend does not crash if MongoDB is
  unreachable at boot or drops mid-session — it retries with backoff and
  returns `503` on data routes until reconnected. The frontend disables the
  message input and shows a "Reconnecting…"/"Disconnected" state whenever
  the Socket.io connection drops, and `reconnectionAttempts: Infinity` on
  the client means it keeps trying to reconnect automatically.
- **Online users are not persisted:** "online" is a property of currently
  open Socket.io connections, tracked in an in-memory `Map` on the server.
  It intentionally is *not* written to MongoDB, since it would go stale the
  moment a process restarts or a socket drops without a clean disconnect.
- **Message content is never rendered as HTML:** React renders `message.message`
  as text content (not `dangerouslySetInnerHTML`), so message text can't
  inject markup, in addition to a light server-side control-character strip.

## Assumptions

- Username is a lightweight, non-authenticated identity (as instructed) —
  a real deployment would replace this with proper auth.
- Two tabs open with the same username are treated as one "online" person;
  they only disappear from the online list once every tab for that username
  has disconnected.
- Chat history is capped to the most recent 200 messages per load to keep
  the initial payload bounded; older messages remain in MongoDB.
- A single global chat room is assumed (no separate rooms/channels), since
  none were requested.
- `MAX_MESSAGE_LENGTH` (1000) and `MAX_USERNAME_LENGTH` (30) are reasonable
  defaults, configurable via backend `.env`.

## Testing

### Automated tests included in this repo

```bash
cd backend
node --test tests/messageService.test.js   # unit tests for validation logic
```

With the backend running (`npm run dev` in one terminal):

```bash
node tests/socket.manual-test.js
```

This connects two real Socket.io clients and checks: connection, `join_chat`,
`online_users` counts/usernames, typing/stop-typing broadcast (and that a
user never receives their own typing event), graceful `send_message`
failure handling, and the `user_disconnected` + online-count-drop on
disconnect.

### Manual two-browser test (the mandatory scenario)

1. Start MongoDB, the backend (`npm run dev` in `backend/`), and the
   frontend (`npm run dev` in `frontend/`).
2. Open **Browser A** at http://localhost:5173, enter username `Alice`.
3. Open **Browser B** (or an incognito window) at the same URL, enter `Bob`.
4. From Alice, send: `Hello Bob`.
   - **Expected:** Bob sees it immediately, no refresh.
5. Refresh Browser B.
   - **Expected:** the message is still there (loaded from MongoDB via
     `GET /api/messages`).
6. Bob replies.
   - **Expected:** Alice receives it immediately.
7. Also check: the online count shows `2` in both windows; closing one
   browser drops it to `1` in the other; typing in one window shows
   "… is typing…" in the other but not in the window doing the typing.

## Deployment

### Backend (Render / Railway / similar Node host)

1. Push this repo to GitHub.
2. Create a new Web Service, root directory `backend/`.
3. Build command: `npm install`. Start command: `npm start`.
4. Set environment variables in the platform's dashboard:
   - `MONGODB_URI` — your MongoDB Atlas (or other hosted Mongo) connection string
   - `CLIENT_URL` — your deployed frontend URL (e.g. `https://your-app.vercel.app`)
   - `NODE_ENV=production`
   - `PORT` — most platforms set this automatically; the app reads `process.env.PORT`
5. Socket.io works over the same HTTP(S) port as the REST API — no separate
   configuration needed, as long as the platform supports WebSockets
   (Render and Railway both do).

### Frontend (Vercel / Netlify / similar)

1. Root directory `frontend/`. Build command: `npm run build`. Output
   directory: `dist`.
2. Set environment variables:
   - `VITE_API_URL` — your deployed backend URL
   - `VITE_SOCKET_URL` — same as above (Socket.io shares the HTTP origin)
3. Deploy. Confirm the deployed frontend can reach the deployed backend
   (check the browser console for CORS or connection errors — if you see
   any, double-check `CLIENT_URL` on the backend matches the frontend's
   actual deployed origin exactly).

> This application was **not** deployed as part of this build (no hosting
> credentials/target were provided). The steps above are what deployment
> requires; the code and configuration are deployment-ready.

## Environment Variables

**`backend/.env`**
| Variable | Purpose | Example |
|---|---|---|
| `PORT` | HTTP port | `5000` |
| `NODE_ENV` | `development` or `production` | `development` |
| `MONGODB_URI` | Mongo connection string | `mongodb://127.0.0.1:27017/realtime-chat` |
| `CLIENT_URL` | Allowed CORS/Socket.io origin | `http://localhost:5173` |
| `MAX_MESSAGE_LENGTH` | Max message chars | `1000` |
| `MAX_USERNAME_LENGTH` | Max username chars | `30` |

**`frontend/.env`**
| Variable | Purpose | Example |
|---|---|---|
| `VITE_API_URL` | Backend REST base URL | `http://localhost:5000` |
| `VITE_SOCKET_URL` | Backend Socket.io URL | `http://localhost:5000` |

Never commit `.env` — only `.env.example` is tracked (see `.gitignore`).

## Known Issues / Limitations

- **MongoDB was not available in the build/test sandbox** (no `mongod`
  binary installable, and the sandbox's network egress is restricted to
  package registries only — `fastdl.mongodb.org`, MongoDB's binary CDN, is
  blocked). As a result, actual message persistence (`POST`/`GET
  /api/messages` writing to and reading from a real database, and the full
  Socket.io `send_message` → save → broadcast path) could **not** be run
  end-to-end in this environment. What **was** verified here, against the
  real running server:
  - Server boots and stays up with MongoDB unreachable, retries with
    backoff, and returns `503` (not a crash or hang) from data routes.
  - `GET /health`, `404` handling, malformed-JSON `400`, oversized-body
    `413`, and CORS origin restriction — all verified via `curl` against
    the live server.
  - Full Socket.io event flow — connect, `join_chat`, `online_users`
    counts/usernames, `typing`/stop-typing broadcast (excluding the typer),
    `user_disconnected`, and graceful `send_message` failure when the DB is
    down — verified via a real two-client `socket.io-client` script
    (`backend/tests/socket.manual-test.js`), 13/13 checks passing.
  - `messageService`'s input-validation logic — verified via unit tests
    (`backend/tests/messageService.test.js`), 5/5 passing.
  - Frontend production build (`vite build`) succeeds; ESLint passes clean
    on all source files.
  - **Not run here:** the mandatory two-browser Alice/Bob scenario, and any
    check that a message actually lands in and reloads from MongoDB. Please
    run `mongod` (or point `MONGODB_URI` at Atlas/Docker) locally and follow
    the "Manual two-browser test" section above — with a real MongoDB
    connected, this is expected to work based on the verified request/response
    contracts and validation logic, but that specific scenario has not
    itself been executed.
- Chat is a single global room; there's no per-conversation/direct-message
  support (not requested).
- "Read receipts" / delivered-status (the optional bonus) was not
  implemented, per the instruction to not sacrifice core functionality for
  optional bonus features — typing indicators and online users (the
  higher-priority bonuses) were implemented and tested instead.
## 👨‍💻 Author

**Neha Gupta**
**AI Full Stack Developer**
