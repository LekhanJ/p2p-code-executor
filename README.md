# P2P Code Executor

> A peer-to-peer code execution desktop app built with Electron, WebRTC, and Socket.IO — write code on one machine and execute it on another in real time, with no server in the middle handling your code.

![Electron](https://img.shields.io/badge/Electron_28-47848F?style=for-the-badge&logo=electron&logoColor=white)
![WebRTC](https://img.shields.io/badge/WebRTC-333333?style=for-the-badge&logo=webrtc&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js_18-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Socket.IO](https://img.shields.io/badge/Socket.IO_4-010101?style=for-the-badge&logo=socketdotio&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

---

## Overview

P2P Code Executor lets two machines connect directly to each other and execute code across the connection. Once the WebRTC peer channel is established, code travels directly between clients — the signaling server is only used for the initial handshake and drops out of the picture entirely after that.

**The core idea:** Peer A writes a JavaScript or Python snippet, clicks "Execute on Remote", and the code runs on Peer B's machine. The output is sent back over the same P2P channel and displayed in Peer A's UI. No cloud execution environment, no shared server processing code — just a direct encrypted data channel between two Electron apps.

**Key design decisions:**
- **WebRTC data channel** (via `simple-peer`) for all post-handshake communication — code and results never touch the signaling server
- **Room-based pairing** — peers share a 6-character room ID to find each other; the signaling server only brokers the SDP offer/answer exchange
- **Dual execution engines** — JavaScript runs in a sandboxed Node.js `vm` context with a 10s timeout; Python spawns a new process on the remote machine
- **Electron desktop shell** — full Node.js access in the renderer, enabling real local process spawning for remote Python execution

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PEER A (Initiator)                              │
│                        Electron Desktop App                             │
│                                                                         │
│  ┌──────────────┐    ┌───────────────┐    ┌─────────────────────────┐   │
│  │  index.html  │    │  renderer.js  │    │    code-executor.js     │   │
│  │  Code Editor │───▶│  UI + Events  │───▶│  JS: vm.runInContext()  │   │
│  │  Lang Select │    │  P2P Events   │    │  PY: spawn python proc  │   │
│  │  Output Pane │◀───│  Exec Router  │    └─────────────────────────┘   │
│  └──────────────┘    └──────┬────────┘                                  │
│                             │                                           │
│                      ┌──────▼────────┐                                  │
│                      │  p2p-manager  │                                  │
│                      │  simple-peer  │                                  │
│                      │  (WebRTC DC)  │                                  │
│                      └──────┬────────┘                                  │
└─────────────────────────────┼───────────────────────────────────────────┘
                              │
              ① Signaling only (SDP offer/answer, ICE)
              ② After handshake: direct WebRTC data channel
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼          ③ Signal relay           ④ Direct P2P
┌─────────────────┐           │            (no server involved)
│ Signaling Server│           │
│ signaling-      │    ┌──────▼──────┐
│  server.js      │    │  Socket.IO  │
│                 │    │  rooms Map  │
│ :3000 (HTTP)    │    │  max 2/room │
│ Socket.IO 4     │    └─────────────┘
│ Docker ready    │
└─────────────────┘
         │
         │ ① Signaling only
         │
┌─────────────────────────────┼───────────────────────────────────────────┐
│                      ┌──────▼────────┐          PEER B (Joiner)         │
│                      │  p2p-manager  │         Electron Desktop App     │
│                      │  simple-peer  │                                  │
│                      │  (WebRTC DC)  │                                  │
│                      └──────┬────────┘                                  │
│                             │                                           │
│                      ┌──────▼────────┐    ┌─────────────────────────┐   │
│  ┌──────────────┐    │  renderer.js  │    │    code-executor.js     │   │
│  │  index.html  │◀───│  UI + Events  │◀───│  JS: vm.runInContext()  │   │
│  │  Output Pane │    │  Exec Handler │    │  PY: spawn python proc  │   │
│  └──────────────┘    └───────────────┘    └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Connection Handshake Sequence

```
Peer A                  Signaling Server               Peer B
  │                           │                           │
  │  socket.emit('create',    │                           │
  │    roomId)                │                           │
  │──────────────────────────▶│                           │
  │◀─── emit('created') ──────│                           │
  │                           │                           │
  │                           │◀── socket.emit('join', ───│
  │                           │      roomId)              │
  │                           │──── emit('joined') ──────▶│
  │◀─── emit('ready') ────────│──── emit('ready') ───────▶│
  │                           │                           │
  │  initPeer(initiator=true) │        initPeer(false)    │
  │  SimplePeer generates     │                           │
  │  SDP offer                │                           │
  │──── emit('signal', offer)─▶── relay to Peer B ───────▶│
  │                           │                           │  SimplePeer
  │                           │                           │  generates answer
  │◀─── relay to Peer A ──────◀── emit('signal', answer) ─│
  │                           │                           │
  │◀══════════ WebRTC Data Channel Established ══════════▶│
  │         (signaling server no longer involved)         │
```

### Code Execution Flow

```
Peer A writes code + clicks "Execute on Remote"
  │
  ▼
renderer.js:executeRemotely()
  │  p2pManager.send({ type: 'execute', language, code })
  │  → serialized as JSON over WebRTC data channel
  │
  ▼
Peer B: p2pManager emits 'data' event
  │
  ▼
renderer.js:handlePeerData()
  │  data.type === 'execute'
  │
  ├─ language: 'javascript'
  │     └─ CodeExecutor.executeJavaScript(code)
  │           vm.createContext(sandbox)        ← isolated sandbox
  │           vm.runInContext(code, sandbox,   ← 10s timeout
  │             { timeout: 10000 })
  │           captures console.log output
  │
  └─ language: 'python'
        └─ CodeExecutor.executePython(code)
              fs.writeFileSync('temp_script.py')
              exec('start cmd /c "python temp_script.py"')
              ← opens new CMD window on Peer B's machine
  │
  ▼
result / error sent back:
  p2pManager.send({ type: 'result', result })
  │
  ▼
Peer A: handlePeerData() → displayOutput(result)
```

---

## Project Structure

```
p2p-code-executor/
│
├── main.js                # Electron entry point — creates BrowserWindow
├── renderer.js            # UI logic, P2P event handling, execution routing
├── p2p-manager.js         # WebRTC peer abstraction via simple-peer + Socket.IO
├── code-executor.js       # JavaScript (vm sandbox) + Python (child_process) execution
├── signaling-server.js    # Lightweight Socket.IO server — room creation & SDP relay
│
├── index.html             # App UI — editor, language selector, output panel, log
├── styles.css             # App styles
│
├── Dockerfile             # Containerised signaling server (Node 18, exposes :3000)
└── package.json           # Dependencies: electron, simple-peer, socket.io
```

---

## How It Works

### 1. Signaling Server — `signaling-server.js`

A minimal Socket.IO HTTP server whose only job is to broker the WebRTC handshake. It maintains a `rooms` Map of up to 2 sockets per room ID.

**Events handled:**

| Event | Direction | What it does |
|---|---|---|
| `create` | Client → Server | Creates a new room, emits `created` back |
| `join` | Client → Server | Adds client to room, emits `ready` to both peers |
| `signal` | Client → Server | Relays SDP offer/answer and ICE candidates to the other peer |
| `disconnect` | Client → Server | Removes client from room; deletes empty rooms |

Once both peers have exchanged SDP via `signal` events, the WebRTC connection is established and the signaling server becomes idle.

### 2. P2P Manager — `p2p-manager.js`

Wraps `simple-peer` (a WebRTC abstraction) and `socket.io-client` into an EventEmitter-based class.

- `createRoom(roomId)` — connects to the signaling server and registers as the room initiator
- `joinRoom(roomId)` — connects and triggers `initPeer(false)` on the joining side
- `initPeer(initiator)` — instantiates a `SimplePeer` with `trickle: false` (full SDP in one shot), wires up signal relay via Socket.IO
- `send(data)` — JSON-serializes and sends over the established WebRTC data channel
- Emits: `connected`, `disconnected`, `data`, `error`

### 3. Code Executor — `code-executor.js`

Handles local execution of the code received over the P2P channel.

**JavaScript** — runs inside a Node.js `vm` sandbox:
```js
const sandbox = { console: { log: (...args) => { output += args.join(' ') + '\n' } } };
vm.createContext(sandbox);
vm.runInContext(code, sandbox, { timeout: 10000 });
```
The sandbox only exposes `console.log` — no `require`, no `fs`, no `process`. Output is captured and returned as a string.

**Python** — writes code to a temp file and spawns a new CMD window:
```js
fs.writeFileSync('temp_script.py', code);
exec('start cmd /c "python temp_script.py & pause"');
```
The new window stays open after execution so the output is visible on the remote machine.

### 4. Renderer — `renderer.js`

Ties everything together inside the Electron renderer process (with `nodeIntegration: true`):

- Creates `P2PManager` and `CodeExecutor` instances on load
- Routes `executeLocally` → `CodeExecutor.execute()` directly
- Routes `executeRemotely` → `p2pManager.send({ type: 'execute', language, code })`
- On receiving `{ type: 'result' }` from the peer, displays output in the UI
- On receiving `{ type: 'execute' }`, runs the code locally and sends the result back

---

## Getting Started

### Prerequisites

- Node.js 18+
- npm
- Python 3 (for Python remote execution on the receiving machine)

### 1. Install dependencies

```bash
npm install
```

### 2. Start the signaling server

```bash
npm run server
# Signaling server running on port 3000
```

Or run it in Docker:

```bash
docker build -t p2p-signaling .
docker run -p 3000:3000 p2p-signaling
```

### 3. Update the signaling server URL

In `renderer.js`, update the IP to the machine running the signaling server:

```js
p2pManager = new P2PManager("http://<your-signaling-server-ip>:3000");
```

### 4. Launch the Electron app (on both machines)

```bash
npm start
```

### 5. Connect and execute

1. **Machine A** — click **Create Room**. A 6-character Room ID is generated.
2. **Machine B** — paste the Room ID and click **Join Room**.
3. Both apps show "Connected" once the WebRTC channel is open.
4. Write code in either machine and click **Execute Locally** or **Execute on Remote**.

---

## Tech Stack

| Technology | Version | Role |
|---|---|---|
| Electron | 28 | Desktop shell — Node.js access in renderer |
| simple-peer | 9.11 | WebRTC data channel abstraction |
| Socket.IO | 4.6 | Signaling server transport |
| socket.io-client | 4.6 | Client-side signaling connection |
| Node.js `vm` | built-in | Sandboxed JavaScript execution |
| Node.js `child_process` | built-in | Python process spawning |
| Docker | — | Containerised signaling server |

---

## Technical Highlights

**True P2P after handshake** — the signaling server only touches SDP and ICE data (a few hundred bytes). All code, results, and application messages flow through the WebRTC data channel directly between peers. If the signaling server goes down after both clients are connected, the session continues unaffected.

**Sandboxed JS execution** — JavaScript runs in a `vm.createContext` sandbox with a custom `console` object that captures output. There is no access to `require`, the file system, or the network from within executed code, preventing accidental damage from untrusted snippets.

**Room-based discovery without accounts** — peers share a randomly generated 6-character alphanumeric room ID out-of-band (e.g. via a message). No authentication, no accounts, no persistent state on the server.

**Electron's `nodeIntegration`** — by setting `nodeIntegration: true` in the BrowserWindow, the renderer process has direct access to Node.js APIs. This is what allows `code-executor.js` to use `vm`, `fs`, and `child_process` directly from the UI layer — an architectural choice that trades security isolation for simplicity in a local tool context.

**Stateless signaling server** — rooms are stored only in memory (`Map`). The server has no database, no authentication, and automatically cleans up rooms when clients disconnect. This makes it trivially deployable as a Docker container.

---

## Potential Extensions

- [ ] Add end-to-end encryption for code payloads using the WebCrypto API
- [ ] Support more languages (Go, Rust via Docker sandboxes on the remote machine)
- [ ] Replace `nodeIntegration: true` with a `contextBridge` preload script for better security isolation
- [ ] Add a shared collaborative editor (CRDT-based, e.g. Yjs over the data channel)
- [ ] Use TURN server configuration for NAT traversal in restrictive network environments
- [ ] Stream `stdout` line-by-line instead of waiting for full execution to complete

---

## License

MIT
