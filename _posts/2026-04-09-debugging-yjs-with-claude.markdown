---
layout: post
title:  "Debugging a CRDT Race Condition With Claude"
date:   2026-04-09 12:00:00 -0700
categories: debugging
---

A collaborative notebook platform was crashing in the browser with a cryptic Yjs error: `Unexpected case`. AI agents were adding cells to notebooks via MCP tool calls, and the browser would silently corrupt the document. This post is the story of how Claude and I spent two days chasing the bug through five layers of infrastructure, hitting dead ends, building a local dev environment from scratch, finding a race condition in the wrong library, deploying a fix that didn't work, and then finding the real bug in a different library entirely.

<style>
.toc { background: #f8f8fc; border: 1px solid #e0e0e8; border-radius: 6px; padding: 1em 1.5em; margin: 1.5em 0; font-size: 0.9em; }
.toc summary { font-weight: 600; cursor: pointer; color: #333; }
.toc ul { margin: 0.5em 0 0 1.2em; padding: 0; list-style: none; }
.toc > ul { margin-left: 0; }
.toc li { margin: 0.25em 0; }
.toc a { text-decoration: none; color: #555; }
.toc a:hover { color: #111; }
</style>

<div class="toc">
<details open>
<summary>Table of Contents</summary>
<ul>
<li><a href="#the-bug">The Bug</a></li>
<li><a href="#the-stack">The Stack</a></li>
<li><a href="#dead-end-1-websocket-timeouts">Dead End 1: WebSocket Timeouts</a></li>
<li><a href="#dead-end-2-bumping-yjs-in-a-fork">Dead End 2: Bumping yjs in a Fork</a></li>
<li><a href="#dead-end-3-module-federation-bundled-true">Dead End 3: Module Federation bundled:true</a></li>
<li><a href="#dead-end-4-rebuilding-jupyterlab-core">Dead End 4: Rebuilding JupyterLab Core</a></li>
<li><a href="#the-turning-point-build-a-local-dev-environment">The Turning Point: Build a Local Dev Environment</a></li>
<li><a href="#finding-the-first-bug">Finding the First Bug</a></li>
<li><a href="#the-first-fix-that-didnt-work">The First Fix (That Didn't Work)</a></li>
<li><a href="#the-wrong-library">The Wrong Library</a></li>
<li><a href="#finding-the-real-bug">Finding the Real Bug</a></li>
<li><a href="#the-real-fix">The Real Fix</a></li>
<li><a href="#proving-it">Proving It</a></li>
<li><a href="#what-i-learned-about-working-with-claude">What I Learned About Working With Claude</a></li>
</ul>
</details>
</div>

## The Bug

Users of our JupyterHub platform were seeing this in the browser console whenever an AI agent added cells to a notebook:

```
Caught error while handling a Yjs update Error: Unexpected case
    at findIndexSS (yjs.mjs:4465)
    at findIndexCleanStart (yjs.mjs:4500)
    at getItemCleanStart (yjs.mjs:4521)
    at Item.getMissing (yjs.mjs:11445)
    at integrateStructs (yjs.mjs:3152)
    ...
```

The error would fire, cell content would get truncated or lost, and the notebook would show "Document session error --- Please refresh the browser tab." This happened most reliably when the AI chat panel (backed by OpenCode via ACP) was rapidly adding cells via MCP tool calls while the user had the notebook open.

Later, a second symptom appeared: opening a second browser tab to the same notebook while the AI was working would show the tab permanently behind --- 58 cells instead of 77, never catching up. No error in the console. Just silent data loss.

## The Stack

The request path has a lot of moving parts:

```
Browser (JupyterLab)
  yjs (JavaScript CRDT library)
    @jupyter/docprovider-extension (WebSocket sync)
      |
      | WebSocket
      |
  jupyter-server
    jupyter-server-documents (sync handler when jupyter-ai is installed)
      OR jupyter-server-ydoc + pycrdt-websocket (sync handler otherwise)
    pycrdt (Rust-based CRDT, Python bindings)
    jupyter-ai -> opencode acp -> Bedrock Claude
      MCP tools: add_cell, edit_cell, etc.
```

The AI agent sends chat messages through `jupyter-ai`, which spawns `opencode acp` as a subprocess. OpenCode calls Bedrock Claude, which responds with MCP tool calls (`add_cell`, `edit_cell`). Those tool calls hit the Jupyter REST API, which modifies the server-side Yjs document. The modification is then broadcast to all connected browser tabs via WebSocket.

Somewhere in this chain, the browser was choking on the updates.

A critical detail that would cost us a day: **when `jupyter-ai` is installed, it brings in `jupyter-server-documents`, which replaces the entire WebSocket sync handler.** The base `pycrdt-websocket` code path is never reached. We didn't know this at first.

## Dead End 1: WebSocket Timeouts

My first theory was that WebSocket connections were being dropped, causing reconnection storms that triggered the yjs error. I asked Claude to look at the ingress configuration. It found two problems immediately:

1. **Wrong annotation vendor.** The JupyterHub ingress used `nginx.org/websocket-services` --- an annotation for the NGINX Inc. controller. But we run the community `ingress-nginx` controller, which ignores `nginx.org/` annotations entirely. The WebSocket annotation was a no-op.

2. **Default 60-second proxy timeout.** Without explicit `proxy-read-timeout` and `proxy-send-timeout` annotations, the nginx ingress controller defaults to 60 seconds. Any WebSocket idle for more than a minute gets killed.

Claude fixed both and created a PR. This was a real improvement --- fewer reconnections means fewer opportunities to trigger the yjs bug --- but it didn't fix the root cause. The error still appeared.

**Time spent: ~45 minutes. Outcome: real fix for a separate problem, but the yjs bug persists.**

## Dead End 2: Bumping yjs in a Fork

The error traced to `sortAndMergeDeleteSet` in yjs, a known bug fixed in version 13.6.30 ([yjs/yjs#767](https://github.com/yjs/yjs/issues/767)). JupyterLab 4.5.6 bundles yjs 13.6.20. Simple: bump yjs in the jupyter-collaboration extension.

Claude forked `jupyter-collaboration` and bumped yjs in `yarn.lock`. But then:

- **The fork installed a metapackage.** `jupyter-collaboration` is a Python metapackage that depends on `jupyter-collaboration-ui` and `jupyter-docprovider`. Installing the metapackage from the fork still pulled the real packages from PyPI with old yjs.

- **The sub-packages need pre-built JS assets.** When we tried installing the correct sub-packages, `hatch-jupyter-builder` tried to rebuild the JavaScript, which failed because Node.js wasn't available in the Docker image.

- **We committed pre-built assets to the fork.** This got past the build, but the yjs version in those assets was still 13.6.20 because only `yarn.lock` was changed --- the JavaScript wasn't rebuilt.

Each iteration required a full Docker build (~25 min), merge, tag, ECR push (~25 min), ArgoCD sync, and pod restart. **Four iterations, ~3 hours, and the bug was still there.**

## Dead End 3: Module Federation bundled:true

JupyterLab uses webpack Module Federation to share dependencies between the core and extensions. yjs is declared as `bundled: false, singleton: true` in the collaboration extensions, meaning they consume whatever version the core provides.

Claude's theory: if we set `bundled: true` and rebuild the extension with yjs 13.6.30, the extension would *provide* the newer version to the singleton scope, and Module Federation would pick the higher version.

We rebuilt the JavaScript, committed the assets, pushed the fork. CI passed. The extension registered `provide shared module (default) yjs@13.6.30` in the webpack build output.

**It didn't work.** Module Federation still picked the core's 13.6.20. The `bundled: true` approach couldn't override the host's singleton.

**Time spent: ~1 hour. Another Docker build cycle. Bug persists.**

## Dead End 4: Rebuilding JupyterLab Core

If the extensions can't override the core, rebuild the core. Claude added a `jupyter lab build` step to the Dockerfile with a yarn resolution override:

```dockerfile
RUN jupyter lab build --dev-build=False --minimize=True && \
    cd /opt/conda/share/jupyter/lab/staging && \
    node -e "..." && \  # patch package.json with resolution
    jlpm install && \
    jlpm run build:prod:minimize
```

This successfully put yjs 13.6.30 into the core's webpack bundle. We verified with `grep`:

```
t[n-1]=new Tl(s.clock,B(s.len,r.clock+r.len-s.clock))
```

That's the fix --- `new Tl(...)` instead of mutating `s.len` in place.

**The error still appeared.** Using Playwright to capture the browser console, we saw the crash was coming from `lab/extensions/@jupyter/docprovider-extension/static/vendors-node_modules_yjs_*` --- the extension's **own** vendor chunk, not the core's. The production build only rebuilds the core; pre-built extensions keep their PyPI-shipped vendor chunks.

**Time spent: ~2 hours across two more Docker cycles. The `sortAndMergeDeleteSet` fix is in the core but the error comes from the extension's copy.**

## The Turning Point: Build a Local Dev Environment

At this point I was frustrated. Each iteration took 45-60 minutes through the Docker/CI/deploy pipeline. I told Claude: "This development workflow stinks. Can we plan to run this locally?"

We built a complete local dev environment in `~/code/jupyter/`:

```
~/code/jupyter/
  yjs/                     # local checkout, v13.6.30
  jupyter-collaboration/   # fork with all sub-packages
  jupyter-ai-tools/        # MCP tools fork
  jupyter_ydoc/            # document models
  pycrdt-websocket/        # sync protocol
  jupyter-server-documents/ # the ACTUAL sync handler (we didn't know yet)
  .venv/                   # uv-managed Python env
```

Key decisions:
- **`uv`** for Python package management (fast, isolated)
- **yarn `portal:` resolution** to link the local yjs checkout into the JupyterLab build
- **`jupyter lab build --dev-build=True --minimize=False`** for source maps and readable function names
- **`AWS_REGION=eu-central-1`** (OpenCode needs the env var explicitly, doesn't read `~/.aws/config`)
- **Backstage MCP** configured with the external URL for the AI agent to look up documentation

The iteration loop went from ~45 minutes to ~15 seconds:

```
Edit yjs source -> rollup build (~2s) -> jlpm build (~10s) -> refresh browser
```

Using Playwright, we automated the full reproduction: open JupyterLab, create a chat, select the OpenCode persona via the `@` mention dropdown, send the triggering prompt, and capture browser console errors. The Playwright snapshots were invaluable for navigating the JupyterLab UI programmatically --- finding the chat input combobox, the persona dropdown, the "Always Allow" button for MCP tool permissions.

## Finding the First Bug

With source maps enabled, the stack trace became readable:

```
findIndexSS          (yjs.mjs:4465)
  -> findIndexCleanStart (yjs.mjs:4500)
  -> getItemCleanStart   (yjs.mjs:4521)
  -> Item.getMissing      (yjs.mjs:11445)
  -> integrateStructs    (yjs.mjs:3152)
```

We added diagnostic logging directly to `findIndexSS` in our local yjs checkout:

```javascript
console.error('[yjs findIndexSS] FAILED', {
  searchClock: clock,
  structsLength: structs.length,
  allRanges: structs.map(s =>
    `${s.id.clock}..${s.id.clock + s.length - 1}`).join(', ')
})
```

The output:

```
searchClock: 1223
allRanges: "0..0, 1..8, ..., 1200..1222"
```

**The client has structs through clock 1222. The server sent an update referencing clock 1223. Off by exactly one.**

We saw the same pattern twice --- always exactly one tick past the end. This wasn't a yjs bug. yjs was correctly rejecting a bad update. The bug was server-side.

## The First Fix (That Didn't Work)

We traced the server-side code and found the race in `pycrdt-websocket`'s `YRoom.serve()`:

```python
async def serve(self, channel):
    async with create_task_group() as tg:
        self.clients.add(channel)          # client added to broadcast list
        sync_message = create_sync_message(self.ydoc)
        await channel.send(sync_message)   # SYNC_STEP1 sent
        async for message in channel:      # wait for client's reply
            ...
```

The client was added to `self.clients` before the sync handshake completed. When the AI agent mutated the doc, `_broadcast_updates()` sent the update to all clients --- including the one mid-handshake. The fix was to defer `self.clients.add(channel)` until after the first sync exchange.

We wrote regression tests, opened an upstream PR ([y-crdt/pycrdt-websocket#138](https://github.com/y-crdt/pycrdt-websocket/pull/138)), pinned the fork in our Docker image, and deployed.

**The crash stopped. But a new, worse problem appeared: silent cell loss.** Opening a second tab to a notebook while the AI was adding cells would show the tab permanently behind (58 cells vs 77). No error in the console. No crash. Just missing cells, forever.

## The Wrong Library

This is where we lost a day. The pycrdt-websocket fix was *correct code* --- it solved the race condition in `YRoom.serve()`. But it was fixing code that was never reached.

When `jupyter-ai` is installed, it depends on `jupyter-server-documents`, a separate package from `jupyter-ai-contrib`. This package **replaces the entire WebSocket sync handler.** It has its own `YRoom` class, its own `YjsClientGroup`, its own `_broadcast_message()`. The `pycrdt-websocket` `YRoom.serve()` method is never called.

We discovered this by tracing the actual WebSocket connection in the running server. The handler was `jupyter_server_documents.websockets.yroom_ws.YRoomWebsocket`, not anything from pycrdt-websocket. The fork we'd deployed was dead code.

## Finding the Real Bug

The bug in `jupyter-server-documents` was the same pattern, but in different code:

```python
# yroom_ws.py - WebSocket handler
def open(self, *_, **__):
    self.client_id = self.yroom.clients.add(self)  # added as "desynced"

# yroom.py - broadcast
def _broadcast_message(self, message, message_type):
    clients = self.clients.get_all()  # defaults to synced_only=True
    for client in clients:
        client.websocket.write_message(message, binary=True)

# yroom.py - sync handshake
def handle_sync_step1(self, client_id, message):
    sync_step2_message = pycrdt.handle_sync_message(...)
    new_client.websocket.write_message(sync_step2_message, ...)
    self.clients.mark_synced(client_id)  # NOW broadcasts reach this client
```

The `_broadcast_message()` method calls `get_all()` which defaults to `synced_only=True`. Any mutations between `open()` (client added as desynced) and `mark_synced()` in `handle_sync_step1()` are broadcast only to already-synced clients. The new desynced client silently misses them.

No crash. No error. Just permanently missing cells.

We tried a catchup approach: after `mark_synced()`, compute `ydoc.get_update(client_state)` using the client's original state vector and send it. But this produced the identical diff as `SYNC_STEP2` (which used the same state vector). Yjs deduplicated it on the client side, making the catchup a no-op.

## The Real Fix

Instead of a catchup, we went with **queue + replay**. Three files changed in `jupyter-server-documents`:

**`clients.py`** --- added a `pending_messages` list to each client:

```python
class YjsClient:
    pending_messages: list[bytes]

    def __init__(self, websocket):
        self.pending_messages = []
```

**`yroom.py` --- `_broadcast_message()`** --- now iterates all clients, queuing for desynced ones:

```python
def _broadcast_message(self, message, message_type):
    for client in self.clients.get_all(synced_only=False):
        if client.synced:
            client.websocket.write_message(message, binary=True)
        else:
            client.pending_messages.append(message)
```

**`yroom.py` --- `handle_sync_step1()`** --- replays queued messages after sync:

```python
self.clients.mark_synced(client_id)

for msg in new_client.pending_messages:
    new_client.websocket.write_message(msg, binary=True)
new_client.pending_messages.clear()
```

SYNC_STEP2 already covers the doc state at the time it was computed. The queued messages cover mutations that happened during the handshake gap. Yjs deduplicates any overlap automatically.

## Proving It

We needed to prove this worked, not just hope. Three layers of testing:

**Code-level proof.** A script that creates a `YjsClientGroup` with one synced and one desynced client, then checks what `_broadcast_message` would do:

```
UNFIXED: _broadcast_message sends to 1 client. Client B received: 0 messages. DROPPED.
FIXED:   _broadcast_message iterates 2 clients. Client B pending: 1 message. QUEUED.
```

**Unit tests.** 13 new tests, including one that simulates the exact production scenario --- 10 initial cells, 19 added during handshake, second client connects mid-stream, all 29 cells must arrive. On the unfixed code: 2 tests fail. On the fix branch: all 23 pass.

**Automated E2E.** We built a Go CLI (`trigger`) that drives two headless Playwright browser sessions against a local JupyterLab. It creates a fresh notebook, chats with `@OpenCode` to generate ~60 cells, opens a second tab mid-stream, and polls both tabs for cell counts every 5 seconds:

```
UNFIXED (main): tab1=63 tab2=63 -> tab1=63 tab2=68 -> tab1=63 tab2=77
                FAIL: tabs diverged, diff=-14

FIXED:          tab1=62 tab2=62 -> tab1=62 tab2=62 -> tab1=62 tab2=62
                PASS: tabs converged (62/62), 47 samples, diff=0
```

The unfixed run shows tab 1 freezing at 63 cells while tab 2 keeps growing --- the classic symptom. The fixed run stays locked at diff=0.

## What I Learned About Working With Claude

**The right fix in the wrong library is the wrong fix.** We spent a day on pycrdt-websocket. The code change was correct. The tests passed. The upstream PR was clean. But `jupyter-server-documents` replaces that entire code path when `jupyter-ai` is installed. Understanding which library actually handles the WebSocket would have saved a full day. When you're debugging through a deep dependency stack, trace the actual runtime code path before writing fixes.

**Claude is good at breadth, bad at knowing when to stop.** The first four attempts were all reasonable hypotheses, and Claude executed each one competently --- creating forks, writing PRs, updating Dockerfiles, cutting tags. But each one went through the full production pipeline before we discovered it didn't work. I should have pushed for a local dev environment earlier.

**The 45-minute iteration loop was the real enemy.** The bug itself was a three-file fix. We spent most of the first day waiting for Docker builds. Once we had a 15-second iteration loop, we found the root cause in about an hour.

**Build automated reproduction tooling early.** We built a Go CLI that drives two Playwright sessions, creates a notebook, triggers the AI agent, opens a second tab, and monitors convergence. This let us run the same test against the unfixed code (FAIL, diff=-14) and fixed code (PASS, diff=0) in minutes instead of hours. The tool paid for itself immediately and will catch regressions in the future.

**Playwright was essential for automated reproduction.** JupyterLab's UI is complex --- chat panels, persona dropdowns, MCP tool permission dialogs. Playwright let us script the full reproduction: navigate to the notebook, create a chat, `@` mention the AI persona, send the triggering prompt, open a second tab, and capture console errors. Without it, we would have been manually clicking through the UI for each test.

**Claude's best move was reading the source.** Not theorizing about Module Federation, not trying another fork strategy --- just reading `yroom.py` line by line and seeing that `get_all()` defaulted to `synced_only=True`. The instrumented `findIndexSS` gave us the exact data ("off by one, always"), and from there the source told the rest of the story.

**Plan for upstream contributions from the start.** We eventually needed forks of four repos (`jupyter-collaboration`, `jupyter-ai-tools`, `pycrdt-websocket`, `jupyter-server-documents`). Having them all checked out in `~/code/jupyter/` with editable installs made it trivial to patch, rebuild, and test locally before pushing upstream PRs with regression tests.
