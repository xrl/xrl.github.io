---
layout: post
title:  "Weekends With Claude: Forking Jellyfin for the LG C2"
date:   2026-05-10 21:00:00 -0700
categories: homelab debugging
---

A 1080p H.264 file with a DTS 5.1 audio track plays video on the LG C2 but emits silence. Eight hours later, `git tag v10.11.5-xrl.X && git push` is a six-minute pipeline that ships a patched Jellyfin server image to the Pi sitting under the TV. This post is the trip from one to the other --- a tour through six layers of stack (TV firmware, Jellyfin web JS, server policy, ffmpeg argv, k3s manifests, GitHub Actions) and one candid admission of a no-op patch I shipped to production before catching it.

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
<li><a href="#the-symptom">The Symptom</a></li>
<li><a href="#the-stack">The Stack</a></li>
<li><a href="#act-1-where-is-the-audio-going">Act 1: Where Is the Audio Going?</a></li>
<li><a href="#act-2-the-client-is-lying-to-the-server">Act 2: The Client Is Lying to the Server</a></li>
<li><a href="#act-3-runtime-sed-as-evidence">Act 3: Runtime sed as Evidence</a></li>
<li><a href="#act-4-the-second-cause">Act 4: The Second Cause</a></li>
<li><a href="#act-5-this-belongs-upstream-ish">Act 5: This Belongs Upstream(-ish)</a></li>
<li><a href="#act-6-the-fork">Act 6: The Fork</a></li>
<li><a href="#act-7-master-is-the-wrong-base">Act 7: master Is the Wrong Base</a></li>
<li><a href="#act-8-the-rebase-onto-v10115">Act 8: The Rebase Onto v10.11.5</a></li>
<li><a href="#act-9-a-tiny-overlay-image">Act 9: A Tiny Overlay Image</a></li>
<li><a href="#act-10-the-homelab-pr">Act 10: The Homelab PR</a></li>
<li><a href="#act-11-first-deploy">Act 11: First Deploy</a></li>
<li><a href="#act-12-the-stutter">Act 12: The Stutter</a></li>
<li><a href="#act-13-the-no-op-patch">Act 13: The No-Op Patch</a></li>
<li><a href="#act-14-pr-5-and-the-six-minute-pipeline">Act 14: PR #5 and the Six-Minute Pipeline</a></li>
<li><a href="#what-this-shows-about-claude-code">What This Shows About Claude Code</a></li>
<li><a href="#the-prs">The PRs</a></li>
</ul>
</details>
</div>

## The Symptom

Friday night. A 1080p H.264 file with a DTS 5.1 audio track. Picture is fine on the LG C2. The video bar shows playback advancing. The audio is dead silent. Sometimes a stuck frame from the previous app lingers behind the playback layer for a beat before Jellyfin draws over it. No error. No toast. Just the absence of sound.

The same Pi is otherwise happy. A 4K HEVC + Dolby Vision file with EAC3 5.1 plays beautifully via direct play. A 1080p H.264 with AAC plays fine. Only DTS sources go silent.

The shape of the bug suggests a codec negotiation issue. The shape of the fix --- by the end of the day --- is a custom server image and a GitOps reconcile loop, but I didn't know that yet.

## The Stack

For context, the homelab is the one I [stood up in March](/homelab/2026/03/09/weekends-with-claude-rpi-homelab.html): Raspberry Pi 5, k3s, Argo CD, Jellyfin from the official Helm chart, LG C2 OLED running the WebOS Jellyfin app as the only client. The relevant request path:

```
LG C2 (WebOS browser)
  Jellyfin Web (main.jellyfin.bundle.js)
    advertises codec capabilities to server
      |
      | HTTPS / HLS
      |
  Jellyfin Server pod on the Pi
    decides direct play vs transcode
      ffmpeg (libfdk_aac, hevc, etc.)
        produces .ts segments
```

The web client lives in a single minified bundle served by the server pod. The WebOS app on the C2 is essentially a Chrome-like browser pointed at that bundle, plus a few `webOS.*` JS hooks. Whatever the bundle says about what the TV can play, the server believes.

Files relevant to this post live in three repos, all under `xrl/`:

| Repo | Role |
|------|------|
| `xrl/jellyfin-web` | Fork of `jellyfin/jellyfin-web` --- the JS client bundle |
| `xrl/jellyfin-rpi` | Tiny overlay image: upstream server + my web bundle |
| `xrl/rpi-homelab` | Helm values + ConfigMaps that Argo CD reconciles |

## Act 1: Where Is the Audio Going?

First instinct: look at what the server is actually doing. The pod is a single container running the upstream `jellyfin/jellyfin:10.11.5` image, but `kubectl exec` is awkward because the image is minimal. `nsenter` from the host is the path of least resistance:

```bash
# On the Pi, find the pod's PID and step into its namespaces
PID=$(crictl inspect --output=json $(crictl ps --name jellyfin -q) \
  | jq '.info.pid')
sudo nsenter -t $PID -m -u -i -n -p
```

Inside the pod, `ffprobe` the source file:

```bash
ffprobe -v error -show_entries stream=index,codec_type,codec_name,channels,bit_rate \
  -of compact /media/movies/<file>
```

Output: `stream|index=0|codec_type=video|codec_name=h264 ...` and `stream|index=1|codec_type=audio|codec_name=dts|channels=6|bit_rate=1536000`. Standard 5.1 DTS at 1.5 Mbps. Nothing exotic.

Then the live `ffmpeg` argv from `/proc/<ffmpeg-pid>/cmdline`. The interesting fragment:

```
-codec:v:0 copy
-codec:a:0 copy
-f hls -hls_segment_type mpegts
```

Both streams `copy`. So the server has decided to direct-stream-copy the DTS audio bytes verbatim into HLS-TS segments. It's not transcoding. The audio stream that hits the C2's decoder is raw DTS. Which would be fine on a TV that decodes DTS. The C2 doesn't.

LG dropped DTS decoding around the C9/CX generation when they got tired of paying for the license. The C2 will happily play DTS over HDMI bitstream if you have an external receiver, but pulling DTS bytes off the wire and decoding them with the TV's built-in audio path? The chip just doesn't have the codec.

The server is making the wrong call. Why?

## Act 2: The Client Is Lying to the Server

Jellyfin's server picks `copy` for an audio stream when the device profile from the client says "yeah, I can play DTS." So the question becomes: what is the WebOS app advertising?

The JS client bundle builds a "device profile" at startup. It probes the browser via `MediaSource.isTypeSupported` and `HTMLMediaElement.canPlayType`, then assembles a JSON blob of supported codecs/containers/profiles and POSTs it to the server. The server uses that profile to decide direct play vs. transcode.

Drop into the WebOS browser dev tools (yes, you can --- LG's developer mode has a remote inspector that hooks WebInspector to your laptop):

```js
> document.createElement('video').canPlayType('audio/mp4; codecs="dts+"')
"probably"
```

There it is. The WebOS browser claims `dts+` is "probably" supported. It is not. There is no DTS decoder on the chip. This is the OEM lying to the web platform about what its media stack can do --- a known wart on WebOS for the last few generations.

So: the client's `canPlayDts()` helper trusts `canPlayType`, returns true, and the device profile gets `dts` in the audio codec list. Server reads "I can do DTS," picks `copy`, ffmpeg passes raw DTS through, C2's audio path stares blankly at it.

## Act 3: Runtime sed as Evidence

Before opening any PRs, I wanted a positive test that lying about DTS support was actually the cause. The fastest way: patch the live JS bundle inside the running pod.

The minified bundle has the DTS push compiled to something like:

```js
z&&(_.push("dca"),_.push("dts"))
```

Where `z` is a local that `canPlayDts()` returned. Force the branch dead with sed:

```bash
# inside the running pod, via nsenter
sed -i 's/z&&(_.push("dca"),_.push("dts"))/!1\&\&(_.push("dca"),_.push("dts"))/' \
  /jellyfin/jellyfin-web/main.jellyfin.bundle.js
```

Confirm the patch is live by curling the served bundle from another shell:

```bash
curl -s http://rpi.local/jellyfin/web/main.jellyfin.bundle.js \
  | grep -o '!1&&(_.push("dca"),_.push("dts"))' | head -1
```

Restart the WebOS app (force-close from the C2 home screen, relaunch) so it pulls the freshly mangled bundle.

Play the file again. **Still silent.** Damn.

## Act 4: The Second Cause

Back to the server. Dump the live ffmpeg argv again. Still `-codec:a:0 copy`. The server is still picking direct-copy on the audio stream even though the client no longer claims DTS support.

That's surprising. With DTS off the codec list, the server should fall back to a transcode --- DTS in, AAC out. Unless something else is preventing transcoding entirely.

The Jellyfin server logs the policy decision at debug level. Bumping the log level and replaying gave the answer:

```
TranscodingDecisionMade: AudioPlaybackTranscoding=false (user policy);
falling back to direct stream copy of DTS
```

The user policy on this Jellyfin install has `EnableAudioPlaybackTranscoding=false`. That's why audio transcoding is refused. With audio transcoding refused and no compatible audio codec, "direct stream copy of DTS" is the only path the server can choose --- it doesn't have permission to convert.

Where did that policy setting come from? My own setup script, two months ago.

The homelab repo has a `setup.sh` ConfigMap that runs once on first Jellyfin boot. Among other things, it disables transcoding via the user policy API:

```bash
# rpi-homelab/jellyfin/configmap-setup.sh (excerpt)
curl -s -X POST -H "X-Emby-Token: $TOKEN" \
  "${JF}/Users/${UID}/Policy" \
  --data '{"EnableVideoPlaybackTranscoding": false,
           "EnableAudioPlaybackTranscoding": false,
           "EnablePlaybackRemuxing": true,
           ...}'
```

The intent in March was correct: don't let the Pi 5 try to transcode HEVC --- the SoC doesn't have the encode silicon, libx265 on four A76 cores is below realtime, and a transcode attempt will pin all four cores at 100% and trip the thermal pad's budget. Disabling video transcoding was deliberate.

But the broad-brush version disabled audio transcoding too. Audio transcoding on a Pi 5 is essentially free --- DTS 5.1 at 1.5 Mbps to AAC 5.1 at 640 kbps is well under one core. I'd over-corrected.

Fix the policy, fix the bug. But the bug *also* lives in the client (because even if I let audio transcode, a more aggressive future client might still negotiate DTS direct copy for a TV that can't actually decode it). Both layers want fixes.

## Act 5: This Belongs Upstream(-ish)

The runtime sed is fine for evidence. It's a bad permanent solution.

The homelab already had one such hack: `patch-web.sh`, a sidecar-style script that ran on every pod start and sed'd the live bundle to flip `enableMkvProgressive` true and add WebOS to a Dolby-Vision-with-HDR10+ allowlist. Two earlier WebOS-on-Jellyfin issues that I'd patched in place rather than upstream. Adding a third sed would have worked, but each one drifts further from anything reproducible. The bundle is minified --- the matchers are fragile --- the next Jellyfin server bump rewrites the surrounding code and the sed silently fails.

The right fix is to ship a patched web bundle, not patch the upstream bundle at runtime. Which means a fork.

## Act 6: The Fork

Four PRs against `xrl/jellyfin-web`'s `master`, in parallel, ~3:24 UTC on Sunday morning:

**PR #1 --- [`disable-dts-direct-play`](https://github.com/xrl/jellyfin-web/pull/1)**

Rewrite `canPlayDts()` in `src/scripts/browserDeviceProfile.js` to return `false` unconditionally. Users with HDMI bitstream to an external receiver can opt back in via a new `appSettings.enableDts()` toggle. The default is the safer default --- decode-and-pray is worse than transcode-and-work.

```js
// before
function canPlayDts(videoTestElement) {
    return videoTestElement.canPlayType
        && (videoTestElement.canPlayType('audio/mp4; codecs="dts-"').replace(/no/, '')
            || videoTestElement.canPlayType('audio/mp4; codecs="dts+"').replace(/no/, ''));
}

// after
function canPlayDts() {
    // The browser's canPlayType lies on several platforms (notably WebOS).
    // Default to false; gate behind an explicit user setting for HDMI bitstream setups.
    return appSettings.enableDts();
}
```

**PR #2 --- [`fix/mkv-progressive-default`](https://github.com/xrl/jellyfin-web/pull/2)**

Flip `enableMkvProgressive` from `false` to `true` in `getBaseProfileOptions()` in `src/components/apphost.js`. This was the first runtime sed, finally turned into source. (Spoiler: this PR was a no-op; we'll get to that.)

**PR #3 --- [`fix/webos-dovi-hdr10plus`](https://github.com/xrl/jellyfin-web/pull/3)**

Add `|| browser.web0s` to the existing guard around `DOVIWithHDR10Plus`:

```js
// before
if (browser.tizenVersion >= 3 || browser.vidaa) { ... }

// after
if (browser.tizenVersion >= 3 || browser.vidaa || browser.web0s) { ... }
```

WebOS on the C2 handles DV+HDR10+ correctly; it just wasn't in the cohort.

**PR #4 --- [`ci/release-workflow`](https://github.com/xrl/jellyfin-web/pull/4)**

A `.github/workflows/release.yml` that triggers on `v*-xrl.*` tag pushes, runs `npm ci && npm run build:production`, tars `dist/`, and creates a GitHub Release with the tarball as the only asset:

```yaml
on:
  push:
    tags: ['v*-xrl.*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run build:production
      - run: tar -czf jellyfin-web-dist.tar.gz -C dist .
      - uses: softprops/action-gh-release@v2
        with:
          files: jellyfin-web-dist.tar.gz
```

Four PRs, four small surfaces. All four merged within ~6 minutes of each other.

Tag `v10.11.5-xrl.1` against master, watch the release workflow run [`25619024435`](https://github.com/xrl/jellyfin-web/actions/runs/25619024435), and... 35.7 MB of `jellyfin-web-dist.tar.gz` lands in the GitHub Release. So far, so good.

Then I tried to deploy it.

## Act 7: master Is the Wrong Base

Before bumping the homelab, I wanted to sanity-check what I'd just built. The fork's `master` branch reports `"version": "12.0.0"` in `package.json`. The running server was `jellyfin/jellyfin:10.11.5`. That mismatch deserved a closer look.

`git log --oneline v10.11.5..master | wc -l` against the upstream `jellyfin/jellyfin-web` checkout: **1115 commits**. `git diff --stat v10.11.5..master`: **399 files changed, +19,970 / −40,106 lines**.

The `@jellyfin/sdk` dependency had jumped from a stable `0.12.0` on the v10.11.5 tag to `0.0.0-unstable.<nightly>` on master --- a nightly built against the in-progress server master's OpenAPI spec. The codec-detection code I'd just patched in `browserDeviceProfile.js` had moved around, picked up new helpers, lost others. The bundle that fell out of `npm run build:production` against master would expect server APIs that the v10.11.5 server doesn't ship.

That's not a paranoid worry. Real upstream issue [`jellyfin/jellyfin#16092`](https://github.com/jellyfin/jellyfin/issues/16092) reported that even a same-minor `10.11.5 → 10.11.6` web client bump *broke DV playback on WebOS* in production. A single point release. Master is 1115 commits past that.

So: master is the wrong base for these patches. They need to land on a branch tracking the `v10.11.5` tag.

## Act 8: The Rebase Onto v10.11.5

New branch: `release-10.11.z-xrl`, branched off the upstream `v10.11.5` tag inside the fork. Cherry-pick the four PRs.

| PR | Cherry-pick result |
|----|--------------------|
| #1 (DTS default-off) | clean |
| #2 (MKV progressive) | clean |
| #4 (release workflow) | clean |
| #3 (WebOS DV+HDR10+) | conflict, hand-translate |

PR #3 needed hand-translation because the v10.11.5 source has a simpler 2-term guard:

```js
// v10.11.5 (the actual base I needed to patch)
if (browser.tizenVersion >= 3 || browser.vidaa) { ... }
```

Master had grown a third term `&& !isWebOsWithoutDolbyVision` that doesn't exist on v10.11.5 and shouldn't be invented for it. Hand-translation was just dropping the master-only term and adding `|| browser.web0s` to the 2-term form.

Fork housekeeping: reset the fork's `master` branch back to upstream master via `git push --force-with-lease`. The PRs against master had served their purpose (review surface, history) but the actual deployable work lives on `release-10.11.z-xrl`. Keeping the fork's master clean means I can re-track upstream cleanly when v10.12 ships.

Tag `v10.11.5-xrl.1` from `release-10.11.z-xrl`, push. Workflow run [`25619024435`](https://github.com/xrl/jellyfin-web/actions/runs/25619024435) runs `npm run build:production` against the rebased branch. Tarball published.

## Act 9: A Tiny Overlay Image

The Jellyfin server is a .NET app that serves the web bundle from a directory. Building the entire server from source on every bundle bump is wasteful --- the server hasn't changed. The right pattern is an overlay image: take the upstream `jellyfin/jellyfin:<tag>` and replace the `jellyfin-web` directory.

New repo: `xrl/jellyfin-rpi`. The Dockerfile is small enough to read in full:

```dockerfile
ARG UPSTREAM_TAG=10.11.5
ARG WEB_TAG=v10.11.5-xrl.1

FROM jellyfin/jellyfin:${UPSTREAM_TAG}
ARG WEB_TAG

USER root
RUN curl -fsSL "https://github.com/xrl/jellyfin-web/releases/download/${WEB_TAG}/jellyfin-web-dist.tar.gz" \
      -o /tmp/web.tar.gz && \
    rm -rf /jellyfin/jellyfin-web/* && \
    tar -xzf /tmp/web.tar.gz -C /jellyfin/jellyfin-web && \
    echo "${WEB_TAG}" > /jellyfin/jellyfin-web/.xrl-web-tag && \
    grep -q 'enableMkvProgressive:!1' /jellyfin/jellyfin-web/main.jellyfin.bundle.js && \
    grep -qE 'tizenVersion>=3\|\|[a-z]\.A\.vidaa\|\|[a-z]\.A\.web0s' \
      /jellyfin/jellyfin-web/main.jellyfin.bundle.js
```

Three things worth noting:

1. **Build-time `grep` of the minified bundle.** If a future tag accidentally drops one of my patches --- because someone reverts a PR upstream, or my rebase missed something --- the `docker build` fails loudly instead of producing a "looks fine" image that silently regresses. Cheap belt-and-suspenders.
2. **`.xrl-web-tag` breadcrumb.** A static file in the served root that I can `curl` to confirm which tag is live in a running pod.
3. **No multi-stage build.** Upstream is already multi-arch, so this is a single-FROM overlay --- the fastest possible Dockerfile.

The companion `.github/workflows/build-and-push.yml` triggers on `v*-xrl.*` tag pushes:

```yaml
on:
  push:
    tags: ['v*-xrl.*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ghcr.io/xrl/jellyfin-rpi:${{ github.ref_name }}
            ghcr.io/xrl/jellyfin-rpi:latest
```

Multi-arch matters because amd64 runners are GitHub-hosted but the Pi is arm64. QEMU emulates the missing arch. Because the only step that runs in emulation is the small overlay (curl, tar, grep, echo) --- the upstream image already has both arches baked in --- emulation is fast: the first build came in at about a minute, subsequent ones with layer caching at ~42 seconds. Pushing both tags (`<version>` and `latest`) means the homelab can pin `:latest` for adventurous reconciles or pin a specific tag for stability.

Public package, no pull secrets. k3s pulls anonymously from `ghcr.io/xrl/jellyfin-rpi`.

First run: [`25619188690`](https://github.com/xrl/jellyfin-rpi/actions/runs/25619188690). Image at `ghcr.io/xrl/jellyfin-rpi:10.11.5-xrl.1` ~03:54 UTC.

## Act 10: The Homelab PR

The homelab repo gets a single PR ([`xrl/rpi-homelab#1`](https://github.com/xrl/rpi-homelab/pull/1)). Three changes:

1. Set the Helm chart's `image.repository` to `ghcr.io/xrl/jellyfin-rpi` and `image.tag` to `10.11.5-xrl.1` in `values.yaml`.
2. Drop `patch-web.sh` from the entrypoint and delete it from the ConfigMap. The patches live in the bundle now.
3. Replace `patch-web.sh` with a new `enforce-policy.sh` that runs once per pod start.

`enforce-policy.sh` is the GitOps complement to having retired the runtime sed. The user policy in Jellyfin's database is a piece of state I want declarative and reconciled, not a one-shot manipulation from `setup.sh` that drifts the moment someone clicks something in the admin UI.

The script:

```bash
#!/bin/sh
# Wait for Jellyfin's public health endpoint
until curl -sf http://localhost:8096/System/Info/Public >/dev/null; do sleep 2; done

TOKEN=$(curl -s -X POST -H "Content-Type: application/json" \
  -H 'X-Emby-Authorization: MediaBrowser Client="enforce-policy", Device="setup", DeviceId="enforce-policy", Version="1"' \
  http://localhost:8096/Users/AuthenticateByName \
  --data '{"Username":"'"${JF_USER}"'","Pw":"'"${JF_PASS}"'"}' \
  | jq -r .AccessToken)

UID=$(curl -s -H "X-Emby-Token: $TOKEN" http://localhost:8096/Users \
  | jq -r ".[] | select(.Name==\"${JF_USER}\") | .Id")

CURRENT=$(curl -s -H "X-Emby-Token: $TOKEN" \
  http://localhost:8096/Users/${UID}/Policy)

WANT_VIDEO=false
WANT_AUDIO=true
WANT_REMUX=true

CUR_VIDEO=$(echo "$CURRENT" | jq -r '.EnableVideoPlaybackTranscoding')
CUR_AUDIO=$(echo "$CURRENT" | jq -r '.EnableAudioPlaybackTranscoding')
CUR_REMUX=$(echo "$CURRENT" | jq -r '.EnablePlaybackRemuxing')

if [ "$CUR_VIDEO" = "$WANT_VIDEO" ] \
   && [ "$CUR_AUDIO" = "$WANT_AUDIO" ] \
   && [ "$CUR_REMUX" = "$WANT_REMUX" ]; then
    echo "=== enforce-policy.sh: policy already correct ==="
    exit 0
fi

echo "=== enforce-policy.sh: policy drift detected ==="
echo "  EnableVideoPlaybackTranscoding: ${CUR_VIDEO} -> ${WANT_VIDEO}"
echo "  EnableAudioPlaybackTranscoding: ${CUR_AUDIO} -> ${WANT_AUDIO}"
echo "  EnablePlaybackRemuxing:         ${CUR_REMUX} -> ${WANT_REMUX}"

NEW=$(echo "$CURRENT" \
  | jq ".EnableVideoPlaybackTranscoding=${WANT_VIDEO} \
        | .EnableAudioPlaybackTranscoding=${WANT_AUDIO} \
        | .EnablePlaybackRemuxing=${WANT_REMUX}")

curl -s -X POST -H "Content-Type: application/json" -H "X-Emby-Token: $TOKEN" \
  http://localhost:8096/Users/${UID}/Policy --data "$NEW"

echo "=== policy corrected ==="
```

Single-shot per pod. Logs `=== enforce-policy.sh: policy already correct ===` on the steady state, `=== policy corrected ===` on drift. Both are easy to grep out of `kubectl logs`.

PR merged at 04:03 UTC. Argo CD picks it up.

## Act 11: First Deploy

Argo's default polling interval is 3 minutes. To skip the wait:

```bash
kubectl annotate app jellyfin -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

Pod rolls. New container image:

```bash
$ kubectl get pod -n media -l app=jellyfin \
    -o jsonpath='{.items[0].spec.containers[0].image}'
ghcr.io/xrl/jellyfin-rpi:10.11.5-xrl.1
```

Bundle markers from the build-time grep are also visible at runtime --- both patterns present in `main.jellyfin.bundle.js`. Good.

`enforce-policy.sh` log on first run:

```
=== enforce-policy.sh: policy drift detected ===
  EnableVideoPlaybackTranscoding: false -> false
  EnableAudioPlaybackTranscoding: false -> true
  EnablePlaybackRemuxing:         true  -> true
=== policy corrected ===
```

Audio transcoding gets flipped on. The original `setup.sh` from March was a one-shot guarded by a marker file on the persistent volume; the marker was still there, so `setup.sh` wouldn't re-run. `enforce-policy.sh` is the right tool for this --- it's stateless against the disk and idempotent against the API.

Replay the silent file. Audio works. ffmpeg argv:

```
-codec:v:0 copy
-codec:a:0 libfdk_aac -ac 6 -ab 640000
-f hls -hls_segment_type mpegts
```

5.1 DTS at 1.5 Mbps to 5.1 AAC at 640 kbps. CPU on the Pi: roughly one core total when streaming, well within budget. Job done.

Almost.

## Act 12: The Stutter

Try a different file. 4K HEVC, Dolby Vision profile 7, HDR10+ fallback, EAC3 5.1 audio. About 5.4 GB at ~14 Mbps. Stutters during playback --- micro-pauses every few seconds, dropped frames visible on motion.

Pi side is fine: ffmpeg running at ~23x realtime, no thermal throttle, CPU spare. It's not a transcode bottleneck.

Inspect the served HLS:

```
GET /jellyfin/videos/<id>/master.m3u8
GET /jellyfin/videos/<id>/hls1/.../<segment>.ts
```

`.ts` again --- MPEG-TS segments. That's HLS classic. For Dolby Vision and HDR10+, `mpegts` is a poor container choice: the segmenter strips DV RPU side data and HDR10+ metadata that lives in NALU SEI messages. fmp4 (CMAF) preserves both end-to-end because the muxer leaves the elementary stream NALUs intact. The TV is getting *something* DV-shaped but with metadata loss, and the renderer is doing partial fallback that costs frames.

The fix should be: tell the WebOS client to prefer fmp4 over mpegts for HLS. Find that decision in the bundle. Source-side it lives in `src/scripts/settings/userSettings.js`:

```js
// excerpt --- the cohort that defaults preferFmp4HlsContainer to true
function preferFmp4HlsContainerDefault() {
    return browser.safari || browser.tizen || browser.chromecast;
}
```

WebOS isn't in the cohort. Add it.

## Act 13: The No-Op Patch

While reading `browserDeviceProfile.js` for the fmp4 fix, I noticed something uncomfortable: PR #2 --- the `enableMkvProgressive` flip in `apphost.js` --- had no consumer in `browserDeviceProfile.js`. The option is *set* in `getBaseProfileOptions()`. It is never *read* anywhere that matters for the device profile produced for the server. It's a setting on a struct that nothing downstream looks at.

In other words: PR #2 had been a no-op since I shipped it. The build-time grep in the Dockerfile for `enableMkvProgressive:!1` is a check that I wrote the patch correctly --- but writing the patch correctly didn't make it do anything.

This is the kind of thing a careful first-pass code review should have caught and didn't. I traced `enableMkvProgressive` from the setter to its (non-existent) reader during the second-pass investigation only because I was already grepping `browserDeviceProfile.js` for the fmp4 work. If the second bug hadn't existed I might have gone months thinking PR #2 was doing something.

Worth admitting plainly: Claude's code-aware first-pass review (and mine) saw the patch land in the right place syntactically and didn't trace its consumers. The second-pass investigation, prompted by an unrelated symptom, is what surfaced it. Both passes were assisted by Claude; only the second one was thorough.

## Act 14: PR #5 and the Six-Minute Pipeline

[PR #5](https://github.com/xrl/jellyfin-web/pull/5) against `release-10.11.z-xrl`, two commits:

1. Add `|| browser.web0s` to `preferFmp4HlsContainerDefault()` in `src/scripts/settings/userSettings.js`.
2. Revert PR #2's `enableMkvProgressive` flip with a comment explaining it was a no-op.

```js
// userSettings.js after
function preferFmp4HlsContainerDefault() {
    return browser.safari || browser.tizen || browser.chromecast || browser.web0s;
}
```

```js
// apphost.js after the revert
// Note: previously flipped enableMkvProgressive to true here. That option is
// set on the profile options object but never read by browserDeviceProfile.js,
// so the change had no runtime effect. Reverting to upstream default and
// leaving the comment so we don't reintroduce the same no-op.
enableMkvProgressive: false,
```

Merged at 04:50 UTC. Tag `v10.11.5-xrl.2` against the branch, push. Release workflow run [`25620198177`](https://github.com/xrl/jellyfin-web/actions/runs/25620198177) takes 3m31s. New tarball published at 04:56 UTC.

In `xrl/jellyfin-rpi`, bump `WEB_TAG` to `v10.11.5-xrl.2` in the Dockerfile (commit `a83d8ce9a6`). Update the build-time grep to check `enableMkvProgressive:!1` instead of `:!0` (the revert flipped it back). Tag `v10.11.5-xrl.2`. Workflow run [`25620266259`](https://github.com/xrl/jellyfin-rpi/actions/runs/25620266259) takes 42 seconds with layer caching. Image at `ghcr.io/xrl/jellyfin-rpi:10.11.5-xrl.2` at 04:58 UTC.

In `xrl/rpi-homelab`, one-line bump of `image.tag` in `values.yaml` to `10.11.5-xrl.2` (commit `c02be7714a`). Push to main. Argo CD reconciles. Pod rolls.

End-to-end:

```
PR #5 merged           04:50 UTC
jellyfin-web release   04:56 UTC  (3m31s build)
jellyfin-rpi image     04:58 UTC  (42s build)
homelab values bumped  04:58 UTC
pod rolled             ~04:56 UTC (Argo sync)
```

Under six minutes from merging the source patch to the running pod.

Pipeline diagram:

```
git tag v10.11.5-xrl.N (xrl/jellyfin-web)
   |
   | release.yml (npm run build:production, ~3m30s)
   |
   v
GitHub Release: jellyfin-web-dist.tar.gz
   |
   | (humans bump WEB_TAG in xrl/jellyfin-rpi/Dockerfile)
   |
   v
git tag v10.11.5-xrl.N (xrl/jellyfin-rpi)
   |
   | build-and-push.yml (qemu + buildx, multi-arch, ~45s with cache)
   |
   v
ghcr.io/xrl/jellyfin-rpi:10.11.5-xrl.N
   |
   | (humans bump image.tag in xrl/rpi-homelab/values.yaml)
   |
   v
Argo CD reconciles, pod rolls (~30s)
   |
   v
enforce-policy.sh runs once: "policy already correct"
   |
   v
LG C2 plays the file
```

Verification on the new pod:

```bash
$ curl -s http://rpi.local/jellyfin/web/.xrl-web-tag
v10.11.5-xrl.2

$ kubectl logs -n media -l app=jellyfin --tail=50 | grep enforce-policy
=== enforce-policy.sh: policy already correct ===
```

Drift correction redundant on the second deploy because the previous pod's enforce-policy.sh had already corrected it. The breadcrumb chain is what matters: tag in fork → tarball in release → image in registry → tag in pod → marker file in bundle → log line.

WebOS app on the C2 caches the JS bundle aggressively, so the freshly-rolled pod isn't enough --- I had to fully restart the Jellyfin app on the TV to make it re-fetch. After that, the 4K HEVC + DV file plays end-to-end without stutter, and the server logs show **no `ffmpeg` invocation at all** for that session: it's pure direct play, the C2 pulling the original MKV bytes over progressive HTTP.

Both bugs cleared. One via transcoding the audio (the easy file). One via fmp4 segmentation that lets the TV direct-play (the harder file).

## What This Shows About Claude Code

I've been doing a "Weekends With Claude" series this year on the kinds of things Claude Code is unusually good at, and unusually bad at. This one is the cleanest illustration so far.

**Six layers of stack in eight hours.** The WebOS app's JS console, the JS device-profile builder, the C# server's policy module, the ffmpeg argv it shells out, the k3s manifests in Argo, the GitHub Actions workflow files. No one of these layers is mysterious; the *traversal* is what's expensive normally, because each layer wants different tooling and different mental context. Claude flips between layers without context switching cost. I asked questions about minified JS, .NET log output, Helm values, and `docker buildx` invocations in the same conversation and got grounded answers in each.

**Parallel agents for breadth, single agent for depth.** I used multiple Claude instances at once for the four-PR fan-out --- one PR per branch, each agent producing a clean diff against the same upstream tree. The rebase and the post-mortem (this post, in fact) are single-agent work because they want one coherent narrative. Knowing when to fan out and when not to is something I'm getting better at.

**The version-mismatch realization is the kind of thing Claude is good at noticing if you ask.** The fork's `master` says 12.0.0, the server is 10.11.5. The first PR set against master would have produced a bundle that requires server APIs that don't exist yet. I asked "is master actually the right base for this?" only because Claude flagged the `package.json` version delta when looking at the dependency tree. Without that nudge I would have shipped a tarball, deployed it, hit some SDK-mismatch runtime error in the C2's console, and burned an hour figuring it out.

**The no-op patch is the candid failure.** PR #2 landed cleanly. It compiled. The grep in the Dockerfile passed. It did nothing. First-pass review, by both of us, did not trace the option from setter to (absent) reader. Second-pass investigation caught it only because I was already grepping the file for an unrelated reason. I'm not going to pretend Claude is past this kind of mistake --- the fix is more disciplined consumer-tracing on patches like this one, and "did this patch actually take effect?" should be a checkbox on every PR with a runtime-grep verifier in the deploy.

**The right primitive for "I patched a thing in someone else's binary" is a fork plus an overlay image.** The runtime `sed` was a fine evidence tool. As a permanent solution it's brittle, undiscoverable, and can't be re-run safely against a future server bump. The fork lets me name the patches in source. The overlay image lets me ship them without rebuilding the .NET server. The release-tag-driven workflow lets me tag-and-forget. Each layer pays for itself the second time you need it, and there will always be a second time.

**Claude was the right primary author for the rebase and the recap.** The four cherry-picks plus one hand-translation off `release-10.11.z-xrl` was tedious, mechanical, but easy to get wrong. The recap (this post) needs a coherent narrative across hundreds of small artifacts --- PR numbers, run IDs, file paths, log lines. Both are tasks where the per-step cost of being a human is high and the cost of supervising Claude is low. I wrote my own paragraph at the end here, but the scaffolding is Claude's.

I started Friday night with a silent video file. I finished Sunday morning with a tagged release pipeline that ships a patched media server image to a Pi sitting under my TV in six minutes. The fixes are real, the failure is named, and the next bug --- there will be a next bug --- has a working pipeline to land in.

## The PRs

| Repo | PR | Title | Status |
|------|----|-------|--------|
| `xrl/jellyfin-web` | [#1](https://github.com/xrl/jellyfin-web/pull/1) | Disable DTS direct play; gate via setting | Merged |
| `xrl/jellyfin-web` | [#2](https://github.com/xrl/jellyfin-web/pull/2) | Flip `enableMkvProgressive` true (no-op, later reverted) | Merged |
| `xrl/jellyfin-web` | [#3](https://github.com/xrl/jellyfin-web/pull/3) | Allow WebOS into DOVIWithHDR10+ cohort | Merged |
| `xrl/jellyfin-web` | [#4](https://github.com/xrl/jellyfin-web/pull/4) | Tag-driven release.yml workflow | Merged |
| `xrl/jellyfin-web` | [#5](https://github.com/xrl/jellyfin-web/pull/5) | Prefer fmp4 HLS for WebOS; revert no-op `enableMkvProgressive` | Merged |
| `xrl/rpi-homelab` | [#1](https://github.com/xrl/rpi-homelab/pull/1) | Pin Jellyfin to `ghcr.io/xrl/jellyfin-rpi`; retire patch-web.sh; introduce enforce-policy.sh | Merged |

Tags: `v10.11.5-xrl.1`, `v10.11.5-xrl.2` on both `xrl/jellyfin-web` and `xrl/jellyfin-rpi`.
Image: `ghcr.io/xrl/jellyfin-rpi:10.11.5-xrl.2` (also tagged `:latest`).

---

*This post was co-authored with [Claude Code](https://claude.com/claude-code), which also did most of the patch authoring, the rebase, the Dockerfile, and the GitHub Actions workflows. The homelab repo is private; the `jellyfin-web` and `jellyfin-rpi` forks are public.*
