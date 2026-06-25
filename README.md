# Fantasy AI Desktop

The desktop shell for [Fantasy AI](https://fantasyai.cloud). It wraps the web app
in a [Tauri](https://tauri.app) window **and** runs a local agent so the chat page
can read/write files and run commands on your machine — powers the plain web build
can never have.

## Architecture

```
┌─────────────────────────── Tauri window (Rust shell) ───────────────────────────┐
│  loads  http://localhost:3000/chat   (the Next.js app in ../fantasy-ai)          │
│                                                                                  │
│  on startup the shell AUTO-SPAWNS ↓                                              │
│                                                                                  │
│   agent/server.mjs  ──  Node WebSocket server on ws://127.0.0.1:8787             │
│        └─ drives the Claude Agent SDK against the Fantasy gateway                │
│                                                                                  │
│  the web page connects to the sidecar via fantasy-ai/lib/agent-bridge.ts         │
│  (localhost WS is a trusted origin, so this works even from the https prod page) │
└──────────────────────────────────────────────────────────────────────────────────┘
```

- **`src-tauri/`** — the Rust shell. Hosts the window and the sidecar lifecycle
  (`src/lib.rs`).
- **`agent/`** — the agent sidecar (`server.mjs`) and a standalone CLI proof
  (`run.mjs`). Uses `@anthropic-ai/claude-agent-sdk` + `ws`.
- **`src/`** — a legacy standalone Vite agent UI. The production window loads the
  Fantasy web app instead, so this is kept only as a minimal fallback surface.

## Zero-pain key (sub-A ✅)

The user never pastes an API key. When logged in on the Fantasy web page, the
`get_agent_key` server action (`fantasy-ai/app/actions/agent.ts`) returns/mints a
Fantasy key for the session; the bridge passes it to the sidecar per prompt.

## Auto-spawned sidecar (sub-B ✅)

`src-tauri/src/lib.rs` spawns `node agent/server.mjs` on startup and kills it on
exit — **no more `npm run serve`.** Details:

- **Won't double-spawn.** If something is already listening on `127.0.0.1:8787`
  (e.g. you ran `npm run serve` yourself), the shell connects to that instead and
  leaves it alone.
- **Graceful if Node is missing.** Logs a warning and disables agent features
  rather than crashing the window.
- **Path resolution**, in order: `FANTASY_AGENT_ENTRY` → bundled
  `<resources>/agent/server.mjs` → the dev tree `../agent/server.mjs` (baked at
  compile time, so `npm run tauri dev` just works).
- Sidecar stdout/stderr is forwarded to the Tauri log (`[sidecar] …`).

### Env overrides (optional)

| Var                   | Default                       | Purpose                                  |
| --------------------- | ----------------------------- | ---------------------------------------- |
| `FANTASY_NODE_BIN`    | `node`                        | Node binary used to launch the sidecar.  |
| `FANTASY_AGENT_ENTRY` | _(auto-resolved)_             | Absolute path to a custom `server.mjs`.  |
| `FANTASY_BASE_URL`    | `http://localhost:3000/api`   | Gateway base URL (read by `server.mjs`). |

## Develop

```bash
# 1. Run the Fantasy web app on :3000 (separate repo ../fantasy-ai)
cd ../fantasy-ai && yarn dev

# 2. Install the sidecar deps once
cd ../fantasy-desktop/agent && npm install

# 3. Launch the desktop shell — it spawns the sidecar for you
cd .. && npm install && npm run tauri dev
```

## Production bundling (sub-C ✅)

The packaged app runs the agent **without any system Node install**:

1. **The agent ships as a resource.** `tauri.conf.json` bundles `../agent/`
   (`server.mjs` + `node_modules`, incl. the Agent SDK's `cli.js`) into
   `$RESOURCE/agent/`. The shell resolves it via `resource_dir()` — see
   `agent_entry()` in `src-tauri/src/lib.rs`.
2. **A Node runtime ships inside it.** `npm run desktop:fetch-node` downloads the
   official Node for your platform and drops just the binary into
   `agent/node-runtime/` (gitignored). Because that folder lives inside the
   bundled `agent/`, it ships automatically — no extra resource entry.
3. **The shell launches the sidecar with that Node** (`resolve_node()`), and the
   sidecar pins `executable: process.execPath` so the Agent SDK spawns its
   `cli.js` with the **same** bundled Node. Zero PATH dependency.

If `agent/node-runtime/` is absent (you skipped step 2), the shell falls back to
`node` on PATH — so dev and "user already has Node" both keep working.

```bash
# Package the desktop app (run fetch-node first so Node is bundled)
cd fantasy-desktop
npm run desktop:fetch-node     # one-time per platform; idempotent
npm run tauri build            # produces the installer in src-tauri/target/release/bundle
```

> Verified: the SDK spawns `…/agent/node-runtime/node.exe …/cli.js`, i.e. the
> bundled runtime, with no system Node on PATH. `node_modules` is installed for
> the host platform — cross-platform installers need a per-platform
> `npm install` + `desktop:fetch-node` (build on each target, or in CI).
> `agent/node_modules/@img` (sharp, ~19 MB) can be pruned if image tools are unused.
