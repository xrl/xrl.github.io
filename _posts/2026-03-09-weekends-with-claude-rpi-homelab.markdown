---
layout: post
title:  "Weekends With Claude: RPi Homelab"
date:   2026-03-09 12:00:00 -0700
categories: homelab
---

A few weekends ago I sat down with [Claude Code](https://claude.com/claude-code) and built a homelab from scratch on a Raspberry Pi 5. The goal: a self-hosted media server that streams 4K Dolby Vision to my LG TV, managed entirely through GitOps. This post walks through every layer of the stack, from flashing the SD card to watching movies.

<style>
.carousel { position: relative; max-width: 100%; overflow: hidden; border-radius: 8px; margin: 1.5em 0; background: #1a1a2e; }
.carousel input[type="radio"] { display: none; }
.carousel .slides { display: flex; transition: transform 0.4s ease; }
.carousel .slide { min-width: 100%; }
.carousel .slide img { width: 100%; display: block; }
.carousel .slide figcaption { text-align: center; padding: 0.5em; color: #aaa; font-size: 0.9em; }
#slide1:checked ~ .slides { transform: translateX(0); }
#slide2:checked ~ .slides { transform: translateX(-100%); }
#slide3:checked ~ .slides { transform: translateX(-200%); }
.carousel .nav { text-align: center; padding: 0.75em 0; }
.carousel .nav label { display: inline-block; width: 12px; height: 12px; border-radius: 50%; background: #555; margin: 0 6px; cursor: pointer; transition: background 0.2s; }
#slide1:checked ~ .nav label[for="slide1"],
#slide2:checked ~ .nav label[for="slide2"],
#slide3:checked ~ .nav label[for="slide3"] { background: #00b4d8; }
</style>

<div class="carousel">
  <input type="radio" name="carousel" id="slide1" checked>
  <input type="radio" name="carousel" id="slide2">
  <input type="radio" name="carousel" id="slide3">
  <div class="slides">
    <figure class="slide">
      <img src="/images/rpi-homelab/argocd.png" alt="Argo CD dashboard showing four healthy, synced applications">
      <figcaption>Argo CD --- four apps, all healthy and synced from Git</figcaption>
    </figure>
    <figure class="slide">
      <img src="/images/rpi-homelab/sabnzbd.png" alt="SABnzbd download client showing queue and history">
      <figcaption>SABnzbd --- Usenet downloads with post-processing history</figcaption>
    </figure>
    <figure class="slide">
      <img src="/images/rpi-homelab/jellyfin.png" alt="Jellyfin media server home screen with Movies and TV Shows">
      <figcaption>Jellyfin --- media library with Movies and TV Shows, ready to stream</figcaption>
    </figure>
  </div>
  <div class="nav">
    <label for="slide1"></label>
    <label for="slide2"></label>
    <label for="slide3"></label>
  </div>
</div>

## What is a homelab?

A homelab is a small server you run at home. It might be a repurposed desktop, a NAS, or in my case a single-board computer tucked behind a TV. People use homelabs to self-host services they'd otherwise rent from the cloud: media servers, ad blockers, home automation, file storage, game servers.

The appeal is ownership. Your data stays on your network. You choose the software. And you learn a lot about infrastructure along the way. The downside is that you're the sysadmin, the network engineer, and the on-call team all at once. A good homelab setup minimizes that operational burden.

## The Raspberry Pi 5

The [Raspberry Pi](https://www.raspberrypi.com/) has come a long way since the original Model B launched in February 2012 with a 700 MHz single-core ARM11 CPU, 512 MB of RAM, and a $35 price tag. The Pi 5 is roughly 600x faster in multicore workloads.

Here's what's inside the board:

| Component | Detail |
|-----------|--------|
| SoC | Broadcom BCM2712 --- quad-core ARM Cortex-A76 at 2.4 GHz, 16nm process |
| RAM | 8 GB LPDDR4X |
| I/O Controller | RP1 --- Raspberry Pi's in-house "southbridge" chip |
| GPU | VideoCore VII (12-core) with hardware HEVC decoder |
| Storage | 128 GB microSD |
| WiFi | Infineon CYW43455 --- dual-band 802.11ac (2.4/5 GHz) |
| Networking | Gigabit Ethernet (via RP1), but I use WiFi |

The **BCM2712** is designed by [Broadcom](https://www.broadcom.com/), the same company that's supplied the SoC for every Pi generation. Broadcom designs the ARM CPU cores under license from Arm Holdings and fabricates the chip at TSMC. The Cortex-A76 cores have 512 KB L2 caches each and share a 2 MB L3 cache.

The **RP1** is [Raspberry Pi's first custom silicon](https://www.raspberrypi.com/news/rp1-the-silicon-controlling-raspberry-pi-5-i-o-designed-here-at-raspberry-pi/), seven years and $15 million in the making. It offloads all peripheral I/O from the main SoC: USB 2.0/3.0, Gigabit Ethernet, GPIO, camera/display interfaces. By splitting I/O onto its own chip, the BCM2712 can focus on compute, and Raspberry Pi gains independence from Broadcom's peripheral IP. The RP1 contains two ARM Cortex-M3 cores, a DMA controller, and 64 KB of SRAM.

### Who makes the Pi?

The corporate structure is unusual. The **Raspberry Pi Foundation** is a UK educational charity, founded in 2008 to promote computer science education. In 2012 it spun out **Raspberry Pi Ltd** (originally Raspberry Pi Trading Ltd) as its commercial subsidiary to design and sell the hardware. Raspberry Pi Ltd has donated nearly $50 million of its profits back to the Foundation. In 2024, the commercial arm went public on the London Stock Exchange as [Raspberry Pi Holdings plc](https://en.wikipedia.org/wiki/Raspberry_Pi_Holdings). The Foundation remains a major shareholder.

### How much does it cost?

The 8 GB Pi 5 I'm using launched at $80. As of early 2026, memory cost inflation from AI infrastructure demand has pushed prices up --- the 8 GB model is around $95, and the 16 GB model has climbed to $205. A 1 GB model was introduced at $45 for folks who just need a lightweight Linux box.

I bought the [CanaKit Raspberry Pi 5 Starter Kit - Aluminum](https://www.canakit.com/canakit-raspberry-pi-5-starter-kit-aluminum.html) for $170 (+ $22 shipping). The kit includes the Pi 5 8 GB board, a 128 GB microSD card pre-loaded with Raspberry Pi OS, a USB-C power supply, an aluminum case that doubles as a passive heatsink (with an Arctic thermal pad transferring heat from the CPU to the case --- no fan needed), two micro-HDMI cables, and a USB card reader.

## Flashing the OS

Before any software, there's a hardware problem: getting the microSD card into your Mac. The M1 MacBook Pro has a full-size SD card slot, but the Pi uses microSD, and the CanaKit doesn't include a full-size SD adapter --- just a USB-A microSD reader. The ASUS monitor's USB hub refused to pass it through. What finally worked: the CanaKit's USB-A card reader plugged into a USB-C to USB-A dongle. Friday night was off to a great start.

I'm running **Debian 13 (Trixie)** via the official Raspberry Pi OS Lite image. "Lite" means no desktop environment --- just a headless Linux system, which is all you need for a server.

The Raspberry Pi OS is built on Debian and maintained by Raspberry Pi Ltd. Trixie is the current testing/stable release of Debian, running kernel 6.12. It ships with systemd, apt, and all the familiar Debian tooling.

To flash the image on macOS, you write it directly to the microSD card with `dd`:

```bash
# identify your SD card (be very careful to pick the right disk!)
diskutil list

# unmount the SD card partitions
diskutil unmountDisk /dev/diskN

# flash the image (bs=4m for reasonable write speed)
sudo dd if=raspios-trixie-arm64-lite.img of=/dev/rdiskN bs=4m status=progress

# eject
diskutil eject /dev/diskN
```

The `rdiskN` (with the `r` prefix) is the raw device, which bypasses macOS's buffer cache and writes significantly faster.

### cloud-init: headless first boot

Rather than plugging in a keyboard and monitor, I configured the Pi for headless access using [cloud-init](https://cloud-init.io/). Cloud-init is the same tool AWS, GCP, and Azure use to bootstrap VMs on first boot. Raspberry Pi OS supports it natively.

You create a `user-data` file on the boot partition of the SD card:

```yaml
#cloud-config
hostname: rpi
manage_etc_hosts: true
preserve_hostname: true

users:
  - name: your-username
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key-here

locale: en_US.UTF-8
timezone: America/New_York

packages:
  - avahi-daemon

runcmd:
  - systemctl enable --now avahi-daemon
```

A few things to note:
- `preserve_hostname: true` prevents cloud-init from reverting the hostname on reboot (I learned this the hard way when the Pi kept renaming itself to `raspberrypi`)
- I install `avahi-daemon` so the Pi announces itself on the local network as `rpi.local`
- SSH keys go here so you can log in immediately without a password

### WiFi credentials: the `network-config` file

Since I'm running the Pi over WiFi (no Ethernet cable), I needed to tell it how to connect to the network before first boot. After flashing the image, remount the SD card and you'll see a FAT32 boot partition that macOS can read. Create a `network-config` file alongside the `user-data` file:

```yaml
version: 2
renderer: NetworkManager
wifis:
  wlan0:
    dhcp4: true
    optional: true
    access-points:
      "Your-WiFi-SSID":
        password: "your-wifi-password"
    regulatory-domain: US
```

Cloud-init processes this on first boot using [Netplan](https://netplan.io/)-style syntax. The Pi's **Infineon CYW43455** WiFi chip supports dual-band 802.11ac, so it'll connect on 5 GHz if your router supports it. Set `regulatory-domain` to your country's [ISO 3166 code](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) so the radio uses the correct channels and power levels.

The Pi 5 also has Gigabit Ethernet via the RP1 chip, but WiFi means one less cable --- just power and you're done. The throughput tradeoff is real (WiFi tops out around 100-200 Mbps vs 1 Gbps wired), but 4K streams peak at 80-120 Mbps, so WiFi handles it fine.

Pop the SD card in, plug in power, and within a minute you can `ssh your-username@rpi.local`.

## mDNS: making `rpi.local` work

[mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) (multicast DNS) lets devices on a local network find each other by name without a DNS server. Apple calls their implementation **Bonjour**, and the open-source implementation is **Avahi**. When the Pi runs `avahi-daemon`, it responds to queries for `rpi.local` with its IP address.

macOS has Bonjour built in, so `rpi.local` works out of the box from your Mac. Linux systems with Avahi installed get it too. Most smartphones support it.

One notable exception: **LG's webOS does not implement mDNS**. The Jellyfin app on my LG C2 can't resolve `rpi.local`, so I have to point it at the Pi's IP address directly (`192.168.1.19`). This is why my Traefik routing for Jellyfin doesn't require a `Host` header match --- the TV connects by IP, and there's no hostname to match against.

## K3s: Kubernetes on the Pi

[K3s](https://k3s.io/) is a lightweight Kubernetes distribution built for edge and IoT. It strips out cloud-provider integrations and replaces etcd with SQLite, compiling down to a single ~70 MB binary. It's a perfect fit for a Pi.

Installation is one line:

```bash
curl -sfL https://get.k3s.io | sh -
```

K3s bundles several components out of the box: **Traefik** as the ingress controller, **CoreDNS**, a **local-path storage provisioner**, and **metrics-server**. On a single-node cluster this gives you a fully functional Kubernetes environment with no additional setup.

### k3s config

The config lives at `/etc/rancher/k3s/config.yaml`:

```yaml
tls-san:
  - rpi.local
  - 192.168.1.19
kubelet-arg:
  - eviction-hard=nodefs.available<5%,imagefs.available<5%,memory.available<100Mi
```

`tls-san` adds Subject Alternative Names to the API server's TLS certificate, so `kubectl` can connect via either `rpi.local` or the IP address without certificate errors. This lets you add the Pi as a kubectl context on your Mac and manage the cluster remotely.

The `kubelet-arg` line configures disk pressure eviction. On a 128 GB microSD that's also storing media downloads, the default 15% free threshold is too aggressive --- it would taint the node and prevent pods from scheduling when you still have 18 GB free. I dropped it to 5%.

### Remote kubectl

After copying the kubeconfig from the Pi, you can manage the cluster from your Mac:

```bash
kubectl --context rpi get nodes
kubectl --context rpi get pods -A
```

No SSH needed for day-to-day operations.

## Argo CD: GitOps for the homelab

[Argo CD](https://argo-cd.readthedocs.io/) is a Kubernetes-native continuous delivery tool. You declare your desired cluster state in a Git repository, and Argo CD continuously reconciles the cluster to match. Push a commit, and your cluster updates automatically.

For a homelab this is overkill in the best way. Every configuration change is version-controlled. If you break something, `git revert` fixes it. You never have to remember what you `kubectl apply`'d three weeks ago.

I install Argo CD via its official Helm chart, slimmed down for the Pi:

```yaml
# argocd/values.yaml - trimmed for single-node RPi

# Disable components I don't need
applicationSet:
  enabled: false
notifications:
  enabled: false
dex:
  enabled: false

# Single replica for everything
controller:
  replicas: 1
server:
  replicas: 1
repoServer:
  replicas: 1

configs:
  params:
    server.insecure: true      # Traefik handles TLS
    server.rootpath: /argocd   # path-based routing
```

ApplicationSet, notifications, and Dex (SSO) are all disabled --- I don't need them on a single-user homelab. Everything runs as a single replica.

### The app-of-apps pattern

Argo CD uses an [app-of-apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) to manage multiple applications from a single root. The bootstrap Application watches the `apps/` directory in the repo:

```yaml
# apps/bootstrap/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:your-user/your-homelab.git
    targetRevision: main
    path: apps
    directory:
      exclude: bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Any YAML file dropped into `apps/` automatically becomes a managed Application. `prune: true` means if you delete a file, Argo deletes the corresponding resources. `selfHeal: true` means if someone manually edits something in the cluster, Argo reverts it.

### How Argo CD pulls from GitHub

Since this is a private repo, Argo CD needs credentials. I use a **GitHub deploy key** --- an ED25519 SSH keypair where the public key is added to the GitHub repo's deploy keys (read-only), and the private key is stored as a Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: homelab-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:your-user/your-homelab.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    <your-deploy-key-here>
    -----END OPENSSH PRIVATE KEY-----
```

This is preferable to adding keys to your GitHub user account because deploy keys are scoped to a single repository. If the Pi is compromised, the blast radius is limited to this one repo, and the key is read-only.

### Repo structure

The homelab repo (private for now, but here's the layout):

```
rpi-homelab/
  argocd/
    install.sh              # bootstrap script
    values.yaml             # Argo CD helm values
    ingressroute.yaml       # Traefik routing for Argo CD UI
    repo-secret.yaml        # deploy key for GitHub access
  apps/
    bootstrap/
      app-of-apps.yaml      # root Application
    sabnzbd.yaml            # SABnzbd Argo Application
    jellyfin.yaml           # Jellyfin Argo Application
    jellyfin-ingress.yaml   # Jellyfin IngressRoute + ConfigMap
  manifests/
    sabnzbd/
      namespace.yaml
      deployment.yaml
      service.yaml
      pvc.yaml
      configmap.yaml         # SABnzbd seed config
      scripts-configmap.yaml # post-processing scripts
      ingressroute.yaml
  helm/
    jellyfin/
      values.yaml            # Jellyfin helm overrides
      configmap.yaml         # network.xml + first-boot setup script
      ingressroute.yaml
  k3s/
    README.md
```

The bootstrap flow: run `argocd/install.sh` once. It installs the Argo CD Helm release, applies the IngressRoute, creates the repo secret, and applies the app-of-apps. From that point on, everything is managed by Git.

## SABnzbd: the stateful config problem

[SABnzbd](https://sabnzbd.org/) is a Usenet download client. It's a Python application with a web UI where users tweak dozens of settings: server credentials, download categories, RSS feeds, bandwidth limits, post-processing options. The config is a single `sabnzbd.ini` file that SABnzbd reads on startup and writes to continuously as you change settings in the UI.

This is at odds with GitOps, where configuration should be declarative and immutable. If Argo CD overwrites `sabnzbd.ini` on every sync, any setting you changed in the UI gets blown away.

My solution: **seed on first boot, then hands off.**

SABnzbd's config is baked into a ConfigMap. An init container copies it to the PVC --- but only if the PVC is empty:

```yaml
initContainers:
  - name: seed-config
    image: busybox:1.36
    command: ["sh", "-c"]
    args:
      - |
        if [ ! -f /config/sabnzbd.ini ]; then
          cp /seed/sabnzbd.ini /config/sabnzbd.ini
          echo "Seeded sabnzbd.ini"
        else
          echo "sabnzbd.ini already exists, skipping"
        fi
    volumeMounts:
      - name: config
        mountPath: /config
      - name: seed
        mountPath: /seed
```

The first time the pod starts, it gets the seed config. After that, SABnzbd owns the file. If you need to reset to defaults, delete the PVC and let the init container re-seed.

The ConfigMap in Git serves as documentation of your baseline config and as disaster recovery. Just remember: don't put your Usenet provider password in a public repo.

### Post-processing: wiring SABnzbd to Jellyfin

When SABnzbd finishes downloading a movie or TV episode, I want Jellyfin to pick it up immediately. SABnzbd supports post-processing scripts per category. Mine authenticates against the Jellyfin API and triggers a library refresh:

```bash
#!/bin/sh
# notify-jellyfin.sh - SABnzbd post-processing script
JELLYFIN="http://jellyfin.jellyfin.svc.cluster.local:8096/jellyfin"

# Authenticate and get token
TOKEN=$(wget -qO- --post-data='{"Username":"your-user","Pw":"your-password"}' \
  --header="Content-Type: application/json" \
  --header="X-Emby-Authorization: MediaBrowser Client=\"sabnzbd\", Device=\"k8s\", DeviceId=\"sabnzbd1\", Version=\"1.0\"" \
  "$JELLYFIN/Users/AuthenticateByName" | sed -n 's/.*"AccessToken":"\([^"]*\)".*/\1/p')

if [ -n "$TOKEN" ]; then
  wget -qO- --post-data='' \
    --header="X-Emby-Authorization: MediaBrowser Token=\"$TOKEN\"" \
    "$JELLYFIN/Library/Refresh"
  echo "Jellyfin library refresh triggered"
fi
exit 0
```

Note the Kubernetes service DNS: `jellyfin.jellyfin.svc.cluster.local`. SABnzbd and Jellyfin are in different namespaces but can reach each other through cluster DNS. No external networking required.

## Jellyfin + LG TV: the direct play sweet spot

[Jellyfin](https://jellyfin.org/) is an open-source media server, a free alternative to Plex and Emby. It organizes your media library, fetches metadata and artwork, and streams to client apps on TVs, phones, and browsers.

The Raspberry Pi 5 **cannot hardware-transcode video**. Its VideoCore VII GPU has a hardware HEVC decoder but no encoder, and Jellyfin doesn't support it for transcoding anyway. If Jellyfin needs to transcode a 4K HEVC stream on the Pi, it falls back to software encoding on the CPU, which the Cortex-A76 cores simply cannot do in real time.

This means **direct play is essential**. The server sends the file as-is to the client, which handles all the decoding. For this to work, the client needs to support every codec in the file natively.

### The LG C2 and Jellyfin's WebOS app

This is where the LG C2 OLED shines. LG paid the licensing fees to Dolby, so the TV natively decodes:

- **HEVC** (H.265) up to 4K@120fps
- **Dolby Vision** Profile 8 (the streaming profile)
- **HDR10** and **HLG**
- **Dolby Digital Plus** and **Dolby Atmos** (via eARC or the built-in decoder)

The [Jellyfin webOS app](https://github.com/jellyfin/jellyfin-webos) is a lightweight wrapper around Jellyfin's web interface. It implements a `NativeShell` bridge that reports the TV's codec capabilities back to the Jellyfin server. When the server sees that the client supports HEVC, Dolby Vision, and DD+ natively, it skips transcoding entirely and sends the raw file.

Under the hood, the app uses the TV's HTML5 video element for playback. The webOS platform routes the video bitstream to the TV's hardware decoder --- the same silicon that decodes Netflix and Disney+. The result is bit-perfect playback with full HDR metadata, exactly as the content was mastered.

### Why this matters: the macOS HDR problem

If you've ever tried to play a 4K Dolby Vision file in VLC on a Mac, you've probably seen the infamous purple-and-green tint. VLC can tone-map HDR10 to SDR for display, but it cannot interpret Dolby Vision metadata --- the proprietary enhancement layer that Dolby licenses to hardware vendors. Without the Dolby license, the player falls back to a broken color mapping that produces psychedelic garbage.

There's no software fix. Dolby Vision is a closed ecosystem: you need licensed silicon to decode it. Apple's own apps (Apple TV, Safari) handle it because Apple pays Dolby, but third-party players like VLC and mpv are out of luck.

With the LG + Jellyfin setup, this is a non-issue. The TV has the Dolby hardware. Jellyfin just serves the file. You get perfect 4K Dolby Vision playback from a $170 server.

### Performance: what the Pi actually does during playback

With Jellyfin direct-playing a 4K HEVC Dolby Vision stream to the LG C2, the Pi is barely working:

| Metric | Idle | Streaming 4K |
|--------|------|-------------|
| CPU idle | ~95% | ~93% |
| Load avg | <1.0 | ~1.1 |
| Jellyfin RAM | 465 MB | 465 MB |
| Available RAM | 6.0 GB | 6.0 GB |

The Pi is essentially acting as a wireless NAS --- reading a file from the SD card and pushing it over WiFi. The CPU barely registers the load. 4K streams typically peak around 80-120 Mbps (10-15 MB/s), well within the CYW43455's 802.11ac throughput. The bottleneck, if anything, is the microSD's sequential read speed, which at ~90 MB/s is still plenty.

## Ups and downs

The entire homelab was built in a single day --- a Friday night session and a Saturday afternoon. Here's the timeline, including the things that went wrong.

### Friday night: standing it up

**12:07 AM --- Argo CD boots.** First commit. K3s is running, Argo CD is installed, the app-of-apps pattern is wired up. GitOps is live.

**12:22 AM --- SABnzbd deployed.** Raw Kubernetes manifests for SABnzbd land in the repo. Argo syncs them automatically. Downloads start flowing within minutes.

**12:26 AM --- RSS feeds wired up.** DogNZB bookmarks feed added so movies queue automatically.

**12:29 AM --- First bug.** SABnzbd's INI format is picky about quoting. The RSS feed URIs needed quotes escaped a specific way. Quick fix, push, Argo syncs.

**12:42 AM --- Jellyfin joins the party.** Initial deployment shares SABnzbd's download directory via a hostPath volume. Both services can see the same files.

**12:44 AM --- Pivot to Helm.** Switched Jellyfin from raw manifests to the official Helm chart. Multi-source Argo Application pulls the chart from Jellyfin's Helm repo and values from the Git repo.

**12:52 AM --- RSS feed tuning.** Bumped results to 250, immediately dropped back to 100. On a 128 GB SD card, you don't want to queue 250 movies at once.

**1:01 AM --- Jellyfin can't route.** Jellyfin's web UI expects to live at a subpath (`/jellyfin`), but the default config puts it at root. Created a ConfigMap with `network.xml` to set the `BaseUrl`. Needed a few iterations to get the seeding approach right (1:08, 1:11).

**1:25 AM --- Downloads land in the wrong place.** Jellyfin expects movies and TV in separate directories. Updated SABnzbd categories to sort completed downloads into `complete/movies/` and `complete/tv/`.

**1:32 AM --- Automated setup wizard.** Wrote a shell script that waits for Jellyfin to boot, completes the setup wizard via the API, creates a user, adds Movies and TV libraries, and triggers the first scan. All of this runs in the background on first boot.

**1:37 AM --- LG TV can't connect.** The IngressRoute required `Host(rpi.local)`, but the LG C2 connects by IP address because webOS doesn't support mDNS. Dropped the Host match so Jellyfin accepts connections from any hostname.

**1:51 AM --- First movie plays.** README written with performance baselines. 4K Dolby Vision streaming to the LG C2, CPU at 93% idle. Time for bed.

### Saturday morning: the wheels come off

Woke up to find **Jellyfin unreachable**. The Pi's disk was 94% full --- downloads had been running all night. Kubernetes had applied a `disk-pressure` taint to the node, evicting pods.

The default kubelet eviction threshold is 15% free space, which on a 128 GB card means pods get evicted when you still have 18 GB free. With 4K movies at 15-50 GB each, that threshold hits fast. Lowered it to 5% and restarted K3s. Pods rescheduled.

Then: **95% I/O wait.** SABnzbd was running two `unrar` processes simultaneously, each hammering the microSD with random reads and writes. The Pi was essentially frozen --- SSH took 30 seconds to respond.

**3:17 AM --- Permission denied.** SABnzbd couldn't write to `complete/tv/` and `complete/movies/`. The directories didn't exist yet, and the container's user didn't have permission to create them. Added an init container to pre-create the directories with correct ownership.

**3:24 AM --- Jellyfin doesn't see new files.** Downloads complete but don't appear in the library. Enabled realtime filesystem monitoring in Jellyfin's library options.

**3:30 AM --- Obfuscated filenames.** Usenet downloads often have randomized filenames to evade DMCA takedowns. SABnzbd can deobfuscate them post-download, but the option was off. Enabled `deobfuscate_final_filenames` --- but discovered the UI checkbox didn't reflect the config file setting. Had to toggle it in the UI directly.

### Saturday afternoon: taming the I/O beast

**12:12 PM --- Disable direct unpack.** SABnzbd's "direct unpack" feature starts extracting files while still downloading. On a microSD card this creates a devastating read/write storm. Disabled it and dropped unpack threads to 1.

**12:24 PM --- SABnzbd OOM killed.** The Python process plus `unrar` exceeded the 1 GB memory limit. Kubernetes killed the container. Bumped the limit to 2 GB.

**1:22 PM --- ionice for the unrar process.** Set SABnzbd's post-processing to `ionice -c3` (idle I/O class). This means unrar only gets disk time when nothing else needs it.

**1:32 PM --- Priority for Jellyfin.** Wrapped Jellyfin's entrypoint with `ionice -c2 -n0` (best-effort, highest priority). Now when you're watching a movie and SABnzbd is unpacking in the background, the stream gets first dibs on the SD card.

**1:59 PM --- _UNPACK_ folders showing in Jellyfin.** SABnzbd creates temporary `_UNPACK_` directories during extraction. Jellyfin saw them and added half-extracted movies to the library. Set SABnzbd's `nomedia_marker` to create `.nomedia` files in those directories.

**2:08 PM --- Wrong marker file.** Jellyfin doesn't honor `.nomedia` --- that's an Android convention. Jellyfin uses `.ignore` files. Changed the marker, backfilled `.ignore` into existing `_UNPACK_` directories. The phantom movies disappeared from the library.

**2:51 PM --- Jellyfin ffmpeg OOM.** Started streaming and ffmpeg (which Jellyfin uses even for direct play, to probe file metadata and remux to HLS) got killed by the OOM killer. Jellyfin was at 1.4 GB of its 2 GB limit when ffmpeg launched. Bumped the limit to 3 GB.

**Post-commit:** A reboot revealed that cloud-init was resetting the hostname to `raspberrypi` on every boot. Services came up but Traefik had a stale pod from before the reboot stuck in a crash loop. Deleted it, set `preserve_hostname: true` in cloud-init, and everything stabilized.

### The pattern

Nearly every problem followed the same arc: deploy something, discover a resource constraint or compatibility issue unique to the Pi's hardware, fix it in Git, push, let Argo sync. The microSD card was the root cause of most pain --- it's the one component you'd upgrade first if doing this again.

## Lessons learned

**microSD is the weakest link.** When SABnzbd is downloading, unpacking, and Jellyfin is streaming simultaneously, the SD card becomes the bottleneck. I set up `ionice` to give Jellyfin I/O priority (`ionice -c2 -n0`, best-effort highest) and demote SABnzbd's post-processing to idle (`ionice -c3`). This keeps streams smooth even during heavy downloads.

**128 GB is tight.** 4K movies are 15-50 GB each. With the OS, Kubernetes, and application data, you've got maybe 60 GB for media. An external USB drive is the obvious next step. The Pi 5's USB 3.0 ports (via RP1) can sustain over 400 MB/s with an SSD.

**Memory limits matter.** Both SABnzbd and Jellyfin hit OOM kills before I tuned their memory limits. SABnzbd's Python process plus `unrar` can spike past 1 GB; I set its limit to 2 GB. Jellyfin's ffmpeg probe buffers pushed it past 2 GB during library scans; I set its limit to 3 GB. On an 8 GB Pi with K3s overhead, you need to be deliberate about resource allocation.

**GitOps works for homelabs.** Having every config change in Git made debugging dramatically easier. When something broke after a reboot, I could diff against the last known-good state. When I needed to tune SABnzbd's I/O settings, I edited a YAML file, pushed, and Argo CD handled the rest.

## The stack

For anyone building something similar, here's the complete stack:

| Layer | Choice |
|-------|--------|
| Hardware | Raspberry Pi 5, 8 GB |
| OS | Raspberry Pi OS Lite (Debian Trixie) |
| Init | cloud-init (headless SSH, avahi, hostname) |
| Container orchestration | K3s |
| Ingress | Traefik (bundled with K3s) |
| GitOps | Argo CD (Helm, app-of-apps pattern) |
| Downloads | SABnzbd |
| Media server | Jellyfin |
| Client | Jellyfin webOS app on LG C2 OLED |

Total cost: ~$170 for the hardware (CanaKit starter kit). Everything else is open source.

---

*This post was co-authored with [Claude Code](https://claude.com/claude-code), which also helped build and debug the homelab itself. The repo is private for now but the structure and approach are fully described above.*
