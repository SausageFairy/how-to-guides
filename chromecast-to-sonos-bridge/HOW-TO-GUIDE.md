# How-To Guide: Chromecast to Sonos Bridge

A step-by-step guide to stream audio from any Google Cast app to Sonos speakers with near-instant latency, running as a Linux container (LXC) on a Proxmox VE host. This guide assumes basic homelab knowledge (SSH, a working Proxmox host, ideally a Home Assistant instance) but explains every step along the way.

**Time estimate:** 2-3 hours (excluding shipping time for hardware)
**Difficulty:** Intermediate
**Cost:** ~EUR 53 in hardware if you already have a Proxmox host, Chromecast, and Sonos (or other AirPlay 2) speakers — no dedicated single-board computer, SD card, or power supply required, since it runs as a container on infrastructure you already have
**Source:** [github.com/SausageFairy/how-to-guides](https://github.com/SausageFairy/how-to-guides)

---

## Table of Contents

1. [What You're Building](#1-what-youre-building)
2. [What You Need](#2-what-you-need)
3. [Create the LXC Container](#3-create-the-lxc-container)
4. [Connect the Hardware and Pass It Through](#4-connect-the-hardware-and-pass-it-through)
5. [Verify the USB Audio Device](#5-verify-the-usb-audio-device)
6. [Install OwnTone](#6-install-owntone)
7. [Configure OwnTone](#7-configure-owntone)
8. [Set Up the Capture Service](#8-set-up-the-capture-service)
9. [One-Time OwnTone Setup](#9-one-time-owntone-setup)
10. [Test It](#10-test-it)
11. [Set Up Session-Gating (Bridge-Control API)](#11-set-up-session-gating-bridge-control-api)
12. [Home Assistant Automations](#12-home-assistant-automations)
13. [Full Soak Test](#13-full-soak-test)
14. [What to Do If Something Goes Wrong](#14-what-to-do-if-something-goes-wrong)

---

## 1. What You're Building

The signal chain looks like this:

```
Your phone/tablet/PC (casts audio via Google Cast)
         ↓
Chromecast (outputs HDMI)
         ↓
HDMI Audio Extractor (splits out the audio as optical/TOSLINK)
         ↓
USB SPDIF Adapter (converts optical to USB)
         ↓
Proxmox host (adapter plugs directly into the host)
         ↓
LXC container (captures audio, runs OwnTone server)
         ↓
Sonos Speakers (receive audio via AirPlay 2 over WiFi)
```

The entire audio chain stays digital — no analog conversion, no quality loss.

The USB SPDIF adapter plugs directly into the Proxmox host, and the host's own audio driver claims it — the container gets access to it via a device bind-mount rather than raw USB passthrough (more on why in step 4). The capture step also only needs to run while you're actually casting, controlled by a small API described in steps 11-12. That matters if you also play audio to the same speakers some other way — for example with Music Assistant over Sonos's native protocol: an always-on AirPlay session on the speaker will block a native-protocol app from taking control of it. If you don't have that concern, you can stop after step 10 and skip the session-gating entirely.

---

## 2. What You Need

### Hardware

| Item | What it does | Approx. cost | Notes |
|------|-------------|-------------|-------|
| **HDMI audio extractor** with optical/TOSLINK output | Splits audio out of the HDMI signal | ~EUR 16 | |
| **Optical TOSLINK cable** | Connects the extractor to the USB adapter | ~EUR 8 | |
| **Cubilux USB SPDIF Receiver** (or similar) | Converts optical audio to USB | ~EUR 29 | Must show up as an ALSA capture device on Linux; USB Audio Class 1 devices work out of the box |
| **Chromecast** (any model with HDMI) | Receives Google Cast audio | Varies | |
| **AirPlay 2 speakers** (Sonos, HomePod, etc.) | Plays the audio | Varies | |

### Software / access prerequisites

- A working Proxmox VE host with spare capacity for one more LXC (this guide assumes Proxmox VE 9.x)
- SSH or console access to the Proxmox host
- Optional but recommended: Home Assistant, for the session-gating automations in steps 11-12. Everything through step 10 works fine without it — OwnTone's own `pipe_autostart` will pick up audio whenever the capture service happens to be running.

---

## 3. Create the LXC Container

On the Proxmox host, confirm you have a free container ID and IP before creating anything:

```bash
for i in <every existing container ID>; do echo "=== $i ==="; pct config $i | grep -i "^net"; done
```

Don't assume the next sequential number is free — check first. This guide uses `113` as an example ID; use whatever's actually free on your host.

Get the exact Debian 13 template filename before creating the container — a wildcard won't expand server-side and produces a confusing storage-type error instead of "file not found":

```bash
pveam update
pveam list local | grep debian-13
```

Create a **privileged** container (needed for the audio device bind-mount in the next step):

```bash
pct create 113 local:vztmpl/debian-13-standard_<version>_amd64.tar.zst \
  --hostname chromecast-bridge \
  --cores 1 --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.50/24,gw=192.168.1.1 \
  --rootfs local-lvm:8 \
  --unprivileged 0 \
  --onboot 1
```

Adjust the storage pool names (`local`, `local-lvm`) and the IP/gateway to match your own network. Start it and confirm it boots:

```bash
pct start 113
pct enter 113
apt update && apt upgrade -y
```

---

## 4. Connect the Hardware and Pass It Through

Plug everything in following this chain — note the adapter goes into the **Proxmox host itself**, not the container directly:

```
Chromecast ──HDMI cable──> HDMI Audio Extractor ──TOSLINK cable──> USB SPDIF Adapter ──USB──> Proxmox host
```

1. Plug the Chromecast into the **HDMI input** of the audio extractor
2. Connect a **TOSLINK cable** from the extractor's optical output to the USB SPDIF adapter's optical input
3. Plug the **USB SPDIF adapter** into a USB port on the **Proxmox host**
4. Power on the Chromecast and the HDMI extractor (if it needs its own power)

On the Proxmox host itself (not inside the container), confirm the adapter was claimed by the host's own audio driver:

```bash
cat /proc/asound/cards
```

You should see it listed as an ALSA card, e.g. `card 1: ReceiverSolid`. **This is expected and correct** — you're not passing the raw USB device through to the container. Instead, the container gets access to the host's already-working ALSA device via a `/dev/snd` bind-mount, which is simpler and more reliable than raw USB-bus passthrough for a device the host can already drive itself.

Edit the container's config on the Proxmox host:

```bash
nano /etc/pve/lxc/113.conf
```

Add these two lines:

```
lxc.cgroup2.devices.allow: c 116:* rwm
lxc.mount.entry: /dev/snd dev/snd none bind,optional,create=dir
```

Reboot the container for this to take effect:

```bash
pct reboot 113
```

---

## 5. Verify the USB Audio Device

Back inside the container (`pct enter 113`), confirm `/dev/snd` made it through:

```bash
arecord -l
```

You should see the same card you saw on the host, now visible inside the container too. Check its actual supported format — many of these adapters are fixed to one rate and won't negotiate a different one:

```bash
cat /proc/asound/card1/stream0
```

(Replace `card1` if your card number differs.) Note whatever rate/depth/channel count it reports (a common one is 48000Hz, 16-bit, 2-channel) — you'll use this exact value everywhere else in this guide, in both the capture command and the OwnTone config. Any mismatch between what you configure and what the hardware actually does is the leading cause of stutter or slow audio drift over a long session.

---

## 6. Install OwnTone

OwnTone isn't in Debian's own apt repos, and the separate `owntone-apt` project's repo is Raspberry Pi-specific, so install it from a prebuilt `.deb`:

```bash
apt install curl -y
curl -LO https://github.com/owntone/owntone-debian/releases/latest/download/owntone_<version>+trixie_amd64.deb
apt install ./owntone_<version>+trixie_amd64.deb
```

Using `apt install ./<file>.deb` (rather than `dpkg -i`) lets apt resolve and pull in dependencies automatically — including `avahi-daemon`, which you'll need later for AirPlay discovery.

**Important:** this package runs `owntone.service` as **root**. There is no dedicated `owntone` system user on this install. Don't create one — it isn't used and won't fix permission issues (see the troubleshooting table at the end).

---

## 7. Configure OwnTone

Create the pipe directory:

```bash
mkdir -p /var/lib/owntone/pipes
```

Edit the config file:

```bash
nano /etc/owntone.conf
```

Set these values (adjust `pipe_sample_rate`/`pipe_bits_per_sample` to match what you found in step 5):

```
general {
    uid = "root"
}

library {
    directories = { "/var/lib/owntone/pipes" }
    pipe_autostart = true
    pipe_sample_rate = 48000
    pipe_bits_per_sample = 16
}

# Disable local ALSA output - AirPlay 2 to your speakers is the only output we want
audio {
    type = "disabled"
}
```

**Don't** add `sync_disable` or `adjust_period_seconds` to the `audio {}` block if you see them suggested elsewhere online — those are for local ALSA output, not a pipe-input/AirPlay-output setup like this one, and OwnTone will reject the config with a parse error.

---

## 8. Set Up the Capture Service

Create the named pipe OwnTone will read from:

```bash
mkfifo /var/lib/owntone/pipes/spdif-bridge
```

Create the systemd unit:

```bash
nano /etc/systemd/system/spdif-capture.service
```

```ini
[Unit]
Description=USB SPDIF capture -> OwnTone pipe (Chromecast bridge)
After=network.target sound.target
Before=owntone.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/ffmpeg -y -f alsa -i hw:1,0 -f s16le -ar 48000 -ac 2 /var/lib/owntone/pipes/spdif-bridge
SuccessExitStatus=255
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Adjust `hw:1,0` and `-ar`/`-ac` to match your card number and the rate you found in step 5. Two details that are easy to get wrong:

- **The `-y` flag is required.** Without it, ffmpeg treats the FIFO like a regular file, hits an interactive overwrite prompt with no TTY attached to answer it, and exits — which under `Restart=always` looks exactly like a crash loop, not a missing flag.
- **`SuccessExitStatus=255` is required.** ffmpeg capturing from ALSA always exits with code 255, even on a completely clean `SIGTERM` stop. Without this line, systemd reports every normal stop as a *failed* unit instead of an *inactive* one — confusing when you're trying to tell a real problem apart from a routine stop.

Reload systemd and start it manually for now (you'll wire up automatic start/stop in steps 11-12):

```bash
systemctl daemon-reload
systemctl start spdif-capture.service
systemctl status spdif-capture.service
```

Enable and start OwnTone itself, which runs continuously:

```bash
systemctl enable owntone
systemctl start owntone
```

---

## 9. One-Time OwnTone Setup

OwnTone has a web interface for managing speakers and playback.

1. Open a browser and go to `http://chromecast-bridge.local:3689` (or `http://192.168.1.50:3689`, using whatever IP you assigned the container, if mDNS doesn't resolve for you)
2. Go to **Settings → Remotes and Outputs** — you should see your Sonos (or other AirPlay 2) speakers listed
3. Toggle your speaker on as an output

If your speakers don't appear, confirm Avahi is running (it discovers AirPlay devices on the network):

```bash
systemctl status avahi-daemon
```

If it's not running:

```bash
systemctl enable avahi-daemon
systemctl start avahi-daemon
systemctl restart owntone
```

---

## 10. Test It

1. Cast something to your Chromecast from any Cast-enabled app
2. Confirm the capture service is actually running: `systemctl status spdif-capture.service`
3. Audio should come out of your speakers with barely noticeable delay

At this point you have a working bridge that runs continuously. If nothing else needs to control the same speakers, you're done — stop here. Steps 11-12 add session-gating specifically for the case where something else (like Music Assistant) also needs the speakers and would otherwise be blocked by an always-on AirPlay session.

---

## 11. Set Up Session-Gating (Bridge-Control API)

**Why this exists:** if you also use something like Music Assistant (or anything else using Sonos's native protocol) to play audio to the same speakers, an always-on OwnTone AirPlay session will block it — AirPlay holds the speaker, so native-protocol commands have nothing to take control of. The fix is to only run `spdif-capture.service` during an actual Chromecast session, controlled by a small HTTP API that Home Assistant calls.

Install Python and create a venv:

```bash
apt install python3-venv -y
mkdir -p /opt/bridge-control
cd /opt/bridge-control
python3 -m venv venv
./venv/bin/pip install flask
```

Create `/opt/bridge-control/app.py`:

```python
import os
import subprocess
from flask import Flask, jsonify, request

app = Flask(__name__)
SHARED_SECRET = os.environ["BRIDGE_CONTROL_SECRET"]
UNIT = "spdif-capture.service"

def _authorized():
    return request.headers.get("X-Secret") == SHARED_SECRET

@app.post("/start")
def start():
    if not _authorized():
        return jsonify(error="unauthorized"), 401
    subprocess.run(["systemctl", "start", UNIT], check=False)
    return jsonify(status="started")

@app.post("/stop")
def stop():
    if not _authorized():
        return jsonify(error="unauthorized"), 401
    subprocess.run(["systemctl", "stop", UNIT], check=False)
    return jsonify(status="stopped")

@app.get("/status")
def status():
    result = subprocess.run(["systemctl", "is-active", UNIT], capture_output=True, text=True)
    return jsonify(active=result.stdout.strip())

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5050)
```

`GET /status` is intentionally left unauthenticated — it only reports `systemctl is-active`, it can't change anything. `/start` and `/stop` both require a shared secret in the `X-Secret` header, since they can control a service.

Generate a secret and create the environment file:

```bash
openssl rand -hex 32   # save this value — you'll need it in step 12 too
nano /opt/bridge-control/.env
```

```bash
BRIDGE_CONTROL_SECRET=<your-generated-secret>
```

Store the secret in your password manager — don't leave it only in this file.

Create the systemd unit:

```bash
nano /etc/systemd/system/bridge-control.service
```

```ini
[Unit]
Description=HTTP control API for spdif-capture.service (session-gated Chromecast bridge)
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/bridge-control
EnvironmentFile=/opt/bridge-control/.env
ExecStart=/opt/bridge-control/venv/bin/python /opt/bridge-control/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Now that the API is in place, disable `spdif-capture.service` from starting automatically at boot — it'll be started on demand instead:

```bash
systemctl disable spdif-capture.service
systemctl daemon-reload
systemctl enable --now bridge-control.service
```

Test it manually before wiring up Home Assistant (replace the IP and secret with your own):

```bash
curl http://192.168.1.50:5050/status
curl -X POST -H "X-Secret: <your-generated-secret>" http://192.168.1.50:5050/start
curl http://192.168.1.50:5050/status
curl -X POST -H "X-Secret: <your-generated-secret>" http://192.168.1.50:5050/stop
```

Why an HTTP API instead of SSH from Home Assistant's `shell_command`? If you're running Home Assistant as HAOS, its Core container doesn't reliably have an SSH client available, so `shell_command` + SSH isn't a robust option. A plain HTTP API called via HA's built-in `rest_command` integration works regardless of how HA itself is installed.

---

## 12. Home Assistant Automations

Add these two entries to the existing `rest_command:` block in `configuration.yaml` (or create the block if you don't have one yet):

```yaml
rest_command:
  start_spdif_bridge:
    url: "http://192.168.1.50:5050/start"
    method: post
    headers:
      X-Secret: "<your-generated-secret>"

  stop_spdif_bridge:
    url: "http://192.168.1.50:5050/stop"
    method: post
    headers:
      X-Secret: "<your-generated-secret>"
```

Restart Home Assistant (or reload YAML configuration) to pick up the new `rest_command:` entries, then create two automations via the UI (**Settings → Automations & Scenes → Create Automation**). Update the entity ID to match your own Chromecast — find it in **Developer Tools → States**.

**Automation 1: start the bridge when casting begins**

- Trigger: your Chromecast's `media_player` entity state changes **to** `playing`
- Action: call service `rest_command.start_spdif_bridge`

**Automation 2: stop the bridge after 5 minutes idle**

- Trigger: the same entity state changes **from** `playing`, **for** 5 minutes
- Action: call service `rest_command.stop_spdif_bridge`

The 5-minute delay on the stop side is a grace period — without it, a normal pause between tracks would stop the capture service and cause an audible gap when playback resumes. Adjust it shorter or longer to taste; a longer mid-session silence gap than whatever you pick could in theory be read as the session ending and trigger a stop/restart blip.

---

## 13. Full Soak Test

Before trusting this day-to-day, run it for a couple of hours under real use — cast something long, let it run, pause and resume a few times, and confirm:

- Audio doesn't stutter or drift out of sync over time
- `spdif-capture.service` starts and stops correctly around your actual casting sessions (`systemctl status spdif-capture.service` should flip between `active` and `inactive`, never `failed`)
- If you use Music Assistant or another native-Sonos-protocol app on the same speakers, confirm it can take over the speaker once a Chromecast session ends

---

## 14. What to Do If Something Goes Wrong

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| No audio at all | Capture service not running | `systemctl start spdif-capture.service`, or check whether the bridge-control automations actually fired |
| `spdif-capture.service` crash-looping | Missing `-y` flag on ffmpeg | Check `journalctl -u spdif-capture.service` for an overwrite-confirmation prompt instead of capture output |
| `spdif-capture.service` shows "failed" after a normal stop | Missing `SuccessExitStatus=255` | Add it to the unit's `[Service]` block, `systemctl daemon-reload` |
| Speakers not showing up in OwnTone's outputs | Avahi not running | `systemctl start avahi-daemon && systemctl restart owntone` |
| Audio stutters or drifts out of sync over time | Sample rate mismatch somewhere in the chain | Re-check that the adapter's actual rate (step 5), the ffmpeg capture rate, and OwnTone's `pipe_sample_rate`/`pipe_bits_per_sample` all match exactly |
| Music Assistant (or similar) can't take over the speaker | `spdif-capture.service` still running when it shouldn't be | Check `curl http://<your-bridge-ip>:5050/status`, then check the stop automation's trace in Home Assistant |
| bridge-control API returns 401 | `X-Secret` header doesn't match `/opt/bridge-control/.env` | Compare the value in your `rest_command` config against the LXC's `.env` |
| Permission errors on the pipe | Something assumed a dedicated `owntone` user exists | It doesn't on this install — both services run as root, don't create one |
| Audio device not found inside the container | `/dev/snd` bind-mount missing, or container not rebooted after editing the LXC config | Re-check `/etc/pve/lxc/<id>.conf` on the **host**, then `pct reboot <id>` |

---

**You're done!** Your Chromecast audio now streams to your speakers with near-instant latency, running as a container on infrastructure you already have — no dedicated single-board computer needed, and (if you set up steps 11-12) it only runs while you're actually casting.
