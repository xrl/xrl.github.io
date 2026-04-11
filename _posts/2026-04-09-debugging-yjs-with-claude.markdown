---
layout: post
title:  "Debugging a CRDT Race Condition With Claude"
date:   2026-04-09 12:00:00 -0700
categories: debugging
---

A collaborative notebook platform was crashing in the browser with a cryptic Yjs error: `Unexpected case`. AI agents were adding cells to notebooks via MCP tool calls, and the browser would silently corrupt the document. This post is the story of how Claude and I spent three days chasing the bug through five layers of infrastructure, hitting dead ends, building a local dev environment from scratch, finding a race condition in the wrong library, deploying a fix that exposed a *second* bug in a *different* library, and then building an executable proof that the two bugs were independent before writing the actual fix.

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
<li><a href="#the-first-real-fix">The First Real Fix</a></li>
<li><a href="#proving-the-first-fix">Proving the First Fix</a></li>
<li><a href="#deployed-to-production-still-broken">Deployed to Production: Still Broken</a></li>
<li><a href="#finding-the-second-bug">Finding the Second Bug</a></li>
<li><a href="#proving-both-bugs-are-independent">Proving Both Bugs Are Independent</a></li>
<li><a href="#the-actual-fix">The Actual Fix</a></li>
<li><a href="#epilogue-the-fix-that-wasnt-installed">Epilogue: The Fix That Wasn't Installed</a></li>
<li><a href="#what-i-learned-about-working-with-claude">What I Learned About Working With Claude</a></li>
<li><a href="#upstream-prs">Upstream PRs</a></li>
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

## The First Real Fix

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

## Proving the First Fix

We needed to prove this worked, not just hope. Three layers of testing:

**Code-level proof.** A script that creates a `YjsClientGroup` with one synced and one desynced client, then checks what `_broadcast_message` would do:

```
UNFIXED: _broadcast_message sends to 1 client. Client B received: 0 messages. DROPPED.
FIXED:   _broadcast_message iterates 2 clients. Client B pending: 1 message. QUEUED.
```

**Unit tests.** 13 new tests, including one that simulates the exact production scenario --- 10 initial cells, 19 added during handshake, second client connects mid-stream, all 29 cells must arrive. On the unfixed code: 2 tests fail. On the fix branch: all 23 pass.

**Automated E2E.** We built a Go CLI ([source](https://gist.github.com/xrl/b510c73f7d8363a040ef0731c3f681a7)) that distills all the fiddly Playwright-driven reproduction steps into a single command. Over several debugging sessions, Claude and I had spent hundreds of turns manually driving `playwright-cli` --- learning JupyterLab's DOM structure, finding the right element refs in snapshot YAMLs, handling dialogs, dealing with multiple notebook panels in a single tab, and getting the cell counting selectors right (`.jp-NotebookPanel.jp-mod-current` to avoid counting cells from background notebooks). The CLI captures all of that hard-won knowledge.

It works by opening two named Playwright sessions (`{uuid}-1` and `{uuid}-2`), each controlling a headless Chrome. Session 1 navigates to JupyterLab, creates a fresh notebook (named with the UUID for traceability), opens a new chat, selects the `@OpenCode` persona from the autocomplete dropdown, and sends a prompt that triggers ~60 cells of rapid AI-generated content. Once cells start appearing, session 2 opens the same notebook in a separate JupyterLab workspace. Three concurrent goroutines then poll both tabs for cell counts and console errors every 5 seconds, logging timestamped diffs to both stdout and a `{uuid}.log` file.

```bash
./trigger --token=$TOKEN --description="commit 5608f99 on fix-sync-handshake-race"
```

```
UNFIXED (main): tab1=63 tab2=63 -> tab1=63 tab2=68 -> tab1=63 tab2=77
                FAIL: tabs diverged, diff=-14

FIXED:          tab1=62 tab2=62 -> tab1=62 tab2=62 -> tab1=62 tab2=62
                PASS: tabs converged (62/62), 47 samples, diff=0
```

The unfixed run shows tab 1 freezing at 63 cells while tab 2 keeps growing --- the classic symptom. The fixed run stays locked at diff=0.

Having a tool like this matters because the bug is a race condition --- it depends on timing between WebSocket handshakes and document mutations. You can't reproduce it by clicking around manually. And if the bug regresses, you don't want to spend another afternoon re-learning which CSS selectors to use or how to handle JupyterLab's chat dialog flow. The CLI encodes all of that and produces a log file you can diff against previous runs.

## Deployed to Production: Still Broken

We deployed the queue+replay fix. Docker build, ECR push, ArgoCD sync. Opened two browser tabs while the AI agent was expanding a notebook.

Both tabs got the same `findIndexSS` "Unexpected case" crash and dropped out of the room. More stable than before --- it took longer to trigger --- but not fixed.

The server logs showed the queue+replay working correctly: messages were being queued and replayed. But the *replayed messages themselves* were crashing the browser. The updates that `_broadcast_message` captured as raw bytes and replayed to the client contained something that JS yjs couldn't digest.

Why didn't local E2E catch this? Because locally, the AI agent adds cells slowly (~5-10 seconds between tool calls, limited by Bedrock round-trip latency). The sync handshake usually completed before any mutations fired, so `pending_messages` was empty. In production with real concurrent load, the race window was hit reliably and the replayed messages crashed the client.

We were in a frustrating spot: the handshake race was definitely real (the logs proved it), the fix was conceptually correct (queue and replay), but the replayed updates were somehow malformed. We needed to reproduce this in a way that didn't require a full-stack deployment.

## Finding the Second Bug

I told Claude to focus on a pure Python stress test --- no Playwright, no Docker, no deployment. Reproduce it in the small.

Claude built a cross-runtime test: Python generates pycrdt updates, writes them to a temp file, Node.js applies them to a JS yjs `Y.Doc`. The test used the local yjs checkout we'd already built with diagnostic logging.

The first test (notebook cells as `Map` values with plain strings) passed fine. But then Claude tried `Text` CRDT operations with Unicode content --- simulating an AI agent typing code with emoji and CJK characters --- and the JS side exploded:

```
[yjs findIndexSS] FAILED — clock not found in structs {
  searchClock: 14,
  structsLength: 1,
  firstStruct: { client: 518772262, clock: 0, len: 14 },
  lastStruct: { client: 518772262, clock: 0, len: 14 },
  allRanges: '0..13'
}
```

The struct covers clocks 0--13. The update references clock 14. Off by one --- the same pattern we'd seen in the browser. And it had nothing to do with the handshake race.

The minimal reproduction was just three lines of Python:

```python
import pycrdt

doc = pycrdt.Doc()
doc['t'] = pycrdt.Text()
doc['t'] += 'A📊B'
doc['t'].insert(2, 'X')  # expects "A📊XB", gets "A📊BX"
```

pycrdt puts the `X` at the wrong position. The emoji `📊` is 1 Python character but takes up more space in pycrdt's internal encoding. After any multi-byte character, all subsequent insert positions are off.

This wasn't just a display bug. The wire-format updates that pycrdt generates encode these wrong positions. When JS yjs tries to apply them, the struct clock references don't match, and `findIndexSS` crashes.

We had been chasing **two independent bugs** that happened to produce the same error:

1. **Bug 1 (handshake race):** Server drops broadcasts for desynced clients during the sync handshake. Causes silent data loss --- missing cells, no error.

2. **Bug 2 (pycrdt offset encoding):** pycrdt encodes Text CRDT positions using UTF-8 byte offsets, but JS yjs expects UTF-16 code unit offsets. After multi-byte characters, incremental updates crash JS yjs with `findIndexSS`.

On the unfixed `main` branch, Bug 1 *masks* Bug 2. The bad updates are silently dropped (never sent to the desynced client), so the client never crashes --- it just silently loses data. When we fixed Bug 1 (queue+replay), the bad updates were finally *delivered*, and Bug 2's crash became visible.

Trading silent data loss for visible crashes was actually progress. But we needed to fix both.

## Proving Both Bugs Are Independent

I was tired of deploying fixes that didn't work. Before writing another line of fix code, I wanted proof that these were two independent bugs and that fixing both would actually work.

Claude built a single Python test with a companion Node.js script that demonstrates both bugs in isolation and in combination. The test has three parts:

**Part A --- Bug 1 in isolation (ASCII only, no Bug 2):**

Handshake race with ASCII-only text. Without the queue, gap data is lost. With the queue, it arrives. No JS involvement needed.

```
  bug1_unfixed:  has_gap_data=False  msgs=2
  bug1_fixed:    has_gap_data=True   msgs=6
```

**Part B --- Bug 2 in isolation (synced client, no handshake race):**

A fully synced client applies incremental Text updates with emoji. Individual updates crash JS yjs. A batched update (same content, one message) works fine.

```
  individual updates: js_errors=1  text_correct=False
    msg 2: Unexpected case
  batched update:     js_errors=0  text_correct=True
```

**Part C --- Both bugs combined (handshake race + Unicode content):**

Three handshake strategies applied to the same scenario: AI agent adds cells with emoji during the handshake gap.

```
  Config                  Gap updates  JS errors  JS has gap?
  combined_no_fix         0            0          NO
  combined_queue_replay   4            1          NO
  combined_queue_batch    1            0          YES
```

`no_fix`: Gap updates are dropped (Bug 1). The bad updates never reach JS, so no crash --- but the data is lost.

`queue_replay`: Gap updates are delivered as 4 individual messages (Bug 1 fixed). One of them crashes JS yjs (Bug 2 exposed). The client drops the update and loses the data anyway.

`queue_batch`: Gap updates are delivered as 1 batched diff computed from the pre-handshake state vector (both fixed). Zero errors. Gap data present.

The full attribution matrix:

```
  Config             Bug 1 (data loss)  Bug 2 (JS crash)
  A: no queue        PRESENT            N/A (ASCII)
  A: with queue      FIXED              N/A (ASCII)
  B: individual      N/A (synced)       PRESENT
  B: batched         N/A (synced)       FIXED
  C: no fix          PRESENT            DORMANT
  C: queue+replay    FIXED              PRESENT
  C: queue+batch     FIXED              FIXED
```

Each bug is independently demonstrable. Each fix addresses exactly one bug. Only fixing both produces correct behavior. This wasn't speculation or theory --- it was executable proof that ran in seconds.

## The Actual Fix

With the attribution matrix in hand, the fix was clear. Two changes, one in each library:

### Server side: batched catchup (jupyter-server-documents)

Replace the individual message replay with a single batched diff:

```python
def handle_sync_step1(self, client_id, message):
    # Save state BEFORE computing SYNC_STEP2
    pre_sync_sv = self._ydoc.get_state()

    # Compute and send SYNC_STEP2 as normal
    sync_step2_message = pycrdt.handle_sync_message(...)
    new_client.websocket.write_message(sync_step2_message, ...)

    # Mark synced
    self.clients.mark_synced(client_id)

    # Send ONE batched diff covering the handshake gap
    # (not individual queued messages that may have bad offsets)
    catchup = self._ydoc.get_update(pre_sync_sv)
    if catchup and len(catchup) > 2:
        catchup_msg = pycrdt.create_update_message(catchup)
        new_client.websocket.write_message(catchup_msg, ...)
    new_client.pending_messages.clear()
```

The key insight: `get_update(pre_sync_sv)` produces a single update containing all mutations since the saved state vector. All CRDT struct references within this single update are self-consistent, so JS yjs can resolve them without hitting the offset encoding bug. The individual queued messages (which have the bad offsets from Bug 2) are discarded --- the batched diff covers the same content.

The `_broadcast_message` queuing stays in place (Bug 1 fix). We just don't replay individual messages --- the batched diff makes them redundant.

### Upstream: UTF-16 offset encoding (pycrdt)

The fundamental fix belongs in pycrdt. The yrs Rust library has an `OffsetKind` option that defaults to `OffsetKind::Bytes` (UTF-8 byte offsets). JS yjs uses UTF-16. The fix is two parts:

**`src/doc.rs`** --- set `OffsetKind::Utf16` when creating a Doc:

```rust
options.offset_kind = OffsetKind::Utf16;
let doc = _Doc::with_options(options);
```

**`python/pycrdt/_text.py`** --- convert Python character indices to UTF-16 code unit indices:

```python
def _char_to_utf16(text: str, char_index: int) -> int:
    """Convert a Python character index to a UTF-16 code unit index."""
    prefix = text[:char_index]
    extra = sum(1 for ch in prefix if ord(ch) > 0xFFFF)
    return char_index + extra
```

This conversion is applied in `insert()`, `__setitem__`, `__delitem__`, `format()`, and `__len__()`. For ASCII and BMP text (most content), it's a no-op.

After the fix, `insert(2, 'X')` after `'A📊B'` correctly produces `'A📊XB'`. All 6 cross-runtime test cases with emoji/CJK/Cyrillic pass where they previously crashed. All 122 existing pycrdt tests pass.

With the pycrdt fix, even the simpler queue+replay approach (Approach B) would work --- the replayed individual updates would have correct offsets. But the batched catchup (Approach C) is the right server-side fix regardless: it's simpler, it sends less data over the wire, and it doesn't depend on the upstream pycrdt fix being merged.

## Epilogue: The Fix That Wasn't Installed

A day after deploying the fix, the bug was still happening in production. I asked Claude to exec into my JupyterHub pod and check whether the pycrdt fork was actually running. It wasn't.

The Dockerfile installed the fork correctly:

```dockerfile
RUN pip install maturin && \
    pip install "pycrdt @ git+https://github.com/xrl/pycrdt@308-fix-utf16-offset-encoding" && \
    rm -rf /home/jovyan/.cargo /home/jovyan/.rustup
```

The CI build log told the whole story:

```
#18 [10/17] RUN pip install maturin && pip install "pycrdt @ ..."
#18 1.566 Collecting pycrdt @ git+https://github.com/xrl/pycrdt@308-fix-utf16-offset-encoding
#18 2.599   Resolved https://github.com/xrl/pycrdt to commit 7cf5af4
#18 4.883 Requirement already satisfied: anyio<5.0.0,>=4.4.0 ...
#18 DONE 5.4s
```

5.4 seconds. pycrdt is a Rust extension --- building it from source with maturin takes minutes, not seconds. pip cloned the fork, read the metadata, saw that the version string was `0.12.50` --- the same as the upstream already installed by `mamba env update` earlier in the Dockerfile --- and decided the requirement was already satisfied. It never compiled anything.

The fork, the upstream, and every CI build all reported version `0.12.50`. pip doesn't compare git commit hashes. It compares version strings. Same version string = already installed = skip. The Rust build never ran.

The fix was one flag:

```dockerfile
RUN pip install maturin && \
    pip install --force-reinstall --no-deps \
    "pycrdt @ git+https://github.com/xrl/pycrdt@308-fix-utf16-offset-encoding" && \
    rm -rf /home/jovyan/.cargo /home/jovyan/.rustup
```

After the fix, the build log showed what should have been happening all along:

```
#18 6.182 Building wheels for collected packages: pycrdt
#18 6.184   Building wheel for pycrdt (pyproject.toml): started
#18 23.79   Building wheel for pycrdt (pyproject.toml): finished with status 'done'
#18 23.80   Attempting uninstall: pycrdt
#18 23.81     Found existing installation: pycrdt 0.12.50
#18 23.87       Successfully uninstalled pycrdt-0.12.50
#18 23.96 Successfully installed pycrdt-0.12.50
```

18 seconds of Rust compilation. Uninstall of the upstream. Install of the fork. This is what "successfully installed" is supposed to look like.

The irony is hard to miss. We spent days proving the fix was correct --- attribution matrices, cross-runtime tests, stress tests, executable proofs. The fix *was* correct. It just wasn't running. pip looked at two identical version strings, shrugged, and moved on. The most dangerous bugs are the ones where everything looks green.

## What I Learned About Working With Claude

**The right fix in the wrong library is the wrong fix.** We spent a day on pycrdt-websocket. The code change was correct. The tests passed. The upstream PR was clean. But `jupyter-server-documents` replaces that entire code path when `jupyter-ai` is installed. Understanding which library actually handles the WebSocket would have saved a full day.

**One bug can mask another.** Bug 1 (silent data loss) masked Bug 2 (JS crash) for months. The bad updates were being dropped before they reached the client, so nobody saw the crash --- just missing cells. Fixing Bug 1 made Bug 2 visible, which felt like a regression. Building the attribution matrix was the only way to untangle them.

**Prove your fix before deploying it.** We deployed queue+replay to production and it failed. If we'd had the cross-runtime Python+Node test earlier, we'd have caught the pycrdt offset bug in seconds instead of wasting another deploy cycle. The attribution test runs in under 3 seconds. A production deploy takes 45 minutes.

**The 45-minute iteration loop was the real enemy.** The bug itself was a three-file fix. We spent most of the first day waiting for Docker builds. Once we had a 15-second iteration loop, we found the root cause in about an hour.

**Claude is good at breadth, bad at knowing when to stop.** The first four attempts were all reasonable hypotheses, and Claude executed each one competently. But each one went through the full production pipeline before we discovered it didn't work. I should have pushed for a local dev environment earlier.

**Build automated reproduction tooling early.** We built a Go CLI ([source](https://gist.github.com/xrl/b510c73f7d8363a040ef0731c3f681a7)) that drives two Playwright sessions and monitors notebook convergence. We also built a pure Python+Node attribution test that proves both bugs independently in under 3 seconds. Both tools paid for themselves immediately.

**Verify the fix is actually running.** We had a correct fix, passing CI, and a green deploy. But pip silently skipped the install because the version strings matched. The pod ran upstream pycrdt for days. `exec` into the pod. `grep` for your patch. Check the build log timestamps. "Successfully installed" means nothing if the build step took 5 seconds instead of 5 minutes.

**Claude's best move was reading the source.** Not theorizing about Module Federation, not trying another fork strategy --- just reading `yroom.py` line by line, reading pycrdt's `text.rs` to find the missing offset conversion, and building executable proofs for each hypothesis.

## Upstream PRs

| PR | Status | What |
|----|--------|------|
| [y-crdt/pycrdt#379](https://github.com/y-crdt/pycrdt/pull/379) | Open | Fix UTF-16 offset encoding for Text operations (Bug 2 root cause) |
| [jupyter-ai-contrib/jupyter-server-documents#218](https://github.com/jupyter-ai-contrib/jupyter-server-documents/pull/218) | Open | Batched catchup during sync handshake (Bug 1 fix + Bug 2 workaround) |
| [jupyter-ai-contrib/jupyter-ai-tools#24](https://github.com/jupyter-ai-contrib/jupyter-ai-tools/pull/24) | Open | Fix `add_cell` creating code cells without `execution_count` |
| [y-crdt/pycrdt-websocket#138](https://github.com/y-crdt/pycrdt-websocket/pull/138) | Open | Fix sync handshake race in `YRoom.serve()` (correct fix, wrong library) |
