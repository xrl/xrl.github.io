---
layout: post
title:  "Debugging a CRDT Race Condition With Claude"
date:   2026-04-09 21:00:00 -0700
categories: debugging
---

A collaborative notebook platform was crashing in the browser with a cryptic Yjs error: `Unexpected case`. AI agents were adding cells to notebooks via MCP tool calls, and the browser would silently corrupt the document. This post is the story of how Claude and I spent a day chasing the bug through five layers of infrastructure, hitting dead ends, building a local dev environment from scratch, and ultimately finding a one-line race condition in the CRDT sync protocol.

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
<li><a href="#finding-the-real-bug">Finding the Real Bug</a></li>
<li><a href="#the-fix">The Fix</a></li>
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
    jupyter-server-ydoc (server-side sync)
      pycrdt-websocket (WebSocket ↔ CRDT bridge)
        pycrdt (Rust-based CRDT, Python bindings)
    jupyter-ai → opencode acp → Bedrock Claude
      MCP tools: add_cell, edit_cell, etc.
```

The AI agent sends chat messages through `jupyter-ai`, which spawns `opencode acp` as a subprocess. OpenCode calls Bedrock Claude, which responds with MCP tool calls (`add_cell`, `edit_cell`). Those tool calls hit the Jupyter REST API, which modifies the server-side Yjs document. The modification is then broadcast to all connected browser tabs via WebSocket.

Somewhere in this chain, the browser was choking on the updates.

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
  pycrdt-websocket/        # sync protocol (later)
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
Edit yjs source → rollup build (~2s) → jlpm build (~10s) → refresh browser
```

Using Playwright, we automated the full reproduction: open JupyterLab, create a chat, select the OpenCode persona via the `@` mention dropdown, send the triggering prompt, and capture browser console errors. The Playwright snapshots were invaluable for navigating the JupyterLab UI programmatically --- finding the chat input combobox, the persona dropdown, the "Always Allow" button for MCP tool permissions.

## Finding the Real Bug

With source maps enabled, the stack trace became readable:

```
findIndexSS          (yjs.mjs:4465)
  → findIndexCleanStart (yjs.mjs:4500)
  → getItemCleanStart   (yjs.mjs:4521)
  → Item.getMissing      (yjs.mjs:11445)
  → integrateStructs    (yjs.mjs:3152)
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

Rebuilt the collaboration extension with the local yjs linked via portal resolution, reinstalled, restarted. Triggered the bug. The output:

```
searchClock: 1223
allRanges: "0..0, 1..8, ..., 1200..1222"
```

**The client has structs through clock 1222. The server sent an update referencing clock 1223. Off by exactly one.**

We saw the same pattern twice --- first `searchClock: 242` with structs through `240..241`, then `searchClock: 1223` with structs through `1200..1222`. Always exactly one tick past the end.

This wasn't a yjs bug. yjs was correctly rejecting a bad update. The bug was server-side.

## The Fix

We added logging to the Python sync protocol (`pycrdt._sync` and `pycrdt.websocket.yroom`). Then we read the `YRoom.serve()` method:

```python
async def serve(self, channel):
    async with create_task_group() as tg:
        self.clients.add(channel)          # client added to broadcast list
        sync_message = create_sync_message(self.ydoc)
        await channel.send(sync_message)   # SYNC_STEP1 sent
        async for message in channel:      # wait for client's reply
            ...
```

Meanwhile, `_broadcast_updates()` runs concurrently:

```python
async def _broadcast_updates(self):
    async for update in self._update_receive_stream:
        for client in self.clients:        # includes the new client!
            self._task_group.start_soon(client.send, message)
```

**The client is added to `self.clients` before the sync handshake completes.** When the AI agent adds a cell (mutating the server-side doc), `_broadcast_updates` sends the update to all clients --- including the one that hasn't finished its initial sync. The update references clock 1223, but the client only has through 1222.

The fix is two lines:

```python
# Before: self.clients.add(channel) at the top of serve()
# After: defer until the first sync exchange completes

synced = False
async for message in channel:
    ...
    reply = handle_sync_message(message[1:], self.ydoc)
    if reply is not None:
        tg.start_soon(channel.send, reply)
    if not synced:
        self.clients.add(channel)  # NOW safe
        synced = True
```

We wrote a regression test that creates a YRoom, connects a client, rapidly mutates the server-side doc 20 times during the handshake, and asserts the client receives all data without errors.

Upstream PR: [y-crdt/pycrdt-websocket#138](https://github.com/y-crdt/pycrdt-websocket/pull/138)

## What I Learned About Working With Claude

**Claude is good at breadth, bad at knowing when to stop.** The first four attempts were all reasonable hypotheses, and Claude executed each one competently --- creating forks, writing PRs, updating Dockerfiles, cutting tags. But each one went through the full production pipeline before we discovered it didn't work. I should have pushed for a local dev environment earlier.

**The 45-minute iteration loop was the real enemy.** The bug itself was a one-line fix. We spent most of the day waiting for Docker builds. Once we had a 15-second iteration loop, we found the root cause in about an hour.

**Playwright was essential for automated reproduction.** JupyterLab's UI is complex --- chat panels, persona dropdowns, MCP tool permission dialogs. Playwright let us script the full reproduction: navigate to the notebook, create a chat, `@` mention the AI persona, send the triggering prompt, open a second tab, and capture console errors. Without it, we would have been manually clicking through the UI for each test.

**Claude's best move was reading the source.** Not theorizing about Module Federation, not trying another fork strategy --- just reading `yroom.py` line by line and seeing that `self.clients.add(channel)` was in the wrong place. The instrumented `findIndexSS` gave us the exact data ("off by one, always"), and from there the source told the rest of the story.

**Plan for upstream contributions from the start.** We eventually needed forks of three repos (`jupyter-collaboration`, `jupyter-ai-tools`, `pycrdt-websocket`). Having them all checked out in `~/code/jupyter/` with editable installs made it trivial to patch, rebuild, and test locally before pushing upstream PRs with regression tests.
