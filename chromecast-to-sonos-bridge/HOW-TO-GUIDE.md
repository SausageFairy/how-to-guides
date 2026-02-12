# How-To Guide: Chromecast to Sonos Bridge

A step-by-step guide to stream audio from any Google Cast app to Sonos speakers with near-instant latency. This guide assumes basic homelab knowledge but explains every step along the way.

**Time estimate:** 1-2 hours (excluding shipping time for hardware)
**Difficulty:** Intermediate
**Cost:** ~EUR 53 if you already own a Raspberry Pi, Chromecast, and Sonos speakers
**Source:** [github.com/SausageFairy/how-to-guides](https://github.com/SausageFairy/how-to-guides)

---

## Table of Contents

1. [What You're Building](#1-what-youre-building)
2. [Shopping List](#2-shopping-list)
3. [Prepare the Raspberry Pi](#3-prepare-the-raspberry-pi)
4. [Connect the Hardware](#4-connect-the-hardware)
5. [Verify the USB Audio Device](#5-verify-the-usb-audio-device)
6. [Install OwnTone](#6-install-owntone)
7. [Configure OwnTone](#7-configure-owntone)
8. [Install the Bridge Script](#8-install-the-bridge-script)
9. [Set Up Auto-Start](#9-set-up-auto-start)
10. [One-Time OwnTone Setup](#10-one-time-owntone-setup)
11. [Test It](#11-test-it)
12. [Home Assistant Automation (Optional)](#12-home-assistant-automation-optional)
13. [What to Do If Something Goes Wrong](#13-what-to-do-if-something-goes-wrong)

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
USB SPDIF Adapter (converts optical to USB for the Pi)
         ↓
Raspberry Pi (captures audio, runs OwnTone server)
         ↓
Sonos Speakers (receive audio via AirPlay 2 over WiFi)
```

The entire audio chain stays digital -- no analog conversion, no quality loss.

---

## 2. Shopping List

### Required Hardware

| Item | What it does | Approx. cost | Where to buy |
|------|-------------|-------------|-------------|
| **HDMI audio extractor** with optical/TOSLINK output | Splits audio out of the HDMI signal | ~EUR 16 | Amazon, AliExpress |
| **Optical TOSLINK cable** | Connects the extractor to the USB adapter | ~EUR 8 | Amazon, any electronics store |
| **Cubilux USB SPDIF Receiver** | Converts optical audio to USB so the Pi can read it | ~EUR 29 | Amazon, cubilux.com |
| **Raspberry Pi** (any model with USB) | The brain that captures and re-streams the audio | Varies (see below) | Official resellers |
| **Chromecast** (any model with HDMI) | Receives Google Cast audio | Varies | Used market |
| **AirPlay 2 speakers** (Sonos, HomePod, etc.) | Plays the audio | Varies | - |
| **MicroSD card** (16GB+) | Stores the Pi's operating system | ~EUR 8 | Any electronics store |
| **USB-C or Micro-USB power supply** for the Pi | Powers the Pi | ~EUR 10 | Official resellers |

### Hardware Alternatives

**Raspberry Pi options:**
- **Pi 3 Model B** (~EUR 35 used) -- Works, but needs the real-time scheduling tweaks in this guide to prevent audio dropouts
- **Pi 4** (~EUR 50-60) -- More headroom, recommended if buying new
- **Pi 5 1GB** (~EUR 45) -- Latest model, plenty of power for this task
- **Pi Zero 2 W** (~EUR 20) -- Cheapest option, but only one USB port so you'll need a micro-USB OTG adapter

**Non-Raspberry Pi alternatives:** Any Linux single-board computer with a USB port works. Popular options:
- **Orange Pi 5** (~EUR 45) -- Good performance, runs Armbian Linux
- **Libre Computer Le Potato** (~EUR 35) -- Budget-friendly, good Linux support
- **Old laptop or thin client** -- Any Linux box with USB works, just more power-hungry

**Instead of the Cubilux adapter:** Any USB audio device with SPDIF/TOSLINK **input** (not output!) that's Linux-compatible. The key is that it must show up as an ALSA capture device. USB Audio Class 1 devices work out of the box on Linux.

**Instead of Chromecast:** Any Google Cast device with HDMI output (Chromecast 3rd gen, Chromecast with Google TV, etc.)

**Instead of Sonos:** Any speaker that supports AirPlay 2 (or even AirPlay 1). OwnTone supports both. This includes HomePod Mini, various Sonos models, and some third-party AirPlay speakers.

---

## 3. Prepare the Raspberry Pi

### 3.1 Flash the operating system

1. Download **Raspberry Pi Imager** from [raspberrypi.com/software](https://www.raspberrypi.com/software/) on your PC/Mac
2. Insert your microSD card into your computer
3. Open Raspberry Pi Imager and click **Choose OS > Raspberry Pi OS (64-bit)**
4. Click **Choose Storage** and select your microSD card
5. Click the **gear icon** (or press Ctrl+Shift+X) to open advanced options:
   - **Set hostname:** `sonos-bridge-pi`
   - **Enable SSH:** Yes, use password authentication
   - **Set username:** `pi` (or whatever you prefer)
   - **Set password:** choose something secure
   - **Configure WiFi:** enter your WiFi network name and password
   - **Set locale:** your timezone (e.g., Europe/Amsterdam)
6. Click **Write** and wait for it to finish
7. Put the microSD card in your Raspberry Pi and power it on

### 3.2 Connect to your Pi

Wait about 2 minutes for the Pi to boot up, then connect to it from your computer.

**SSH** (Secure Shell) lets you control the Pi remotely from a terminal on your computer. You type commands on your computer, and they run on the Pi.

Open a terminal on your computer (PowerShell on Windows, Terminal on Mac/Linux) and type:
```bash
ssh pi@sonos-bridge-pi.local
```

> **Note:** `pi` is the username you chose in step 3.1. If you picked a different username, use that instead (e.g., `ssh yourname@sonos-bridge-pi.local`). This applies to all commands in this guide that reference the `pi` user.

If `sonos-bridge-pi.local` doesn't work, find the Pi's IP address in your router's admin page and use that instead (e.g., `ssh pi@192.168.1.100`).

Type `yes` when asked about the fingerprint, then enter your password.

### 3.3 Update the system

Once connected, run these commands to make sure everything is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

`sudo` runs a command with administrator privileges. `apt` is the package manager that installs and updates software.

---

## 4. Connect the Hardware

Plug everything in following this chain:

```
Chromecast ──HDMI cable──> HDMI Audio Extractor ──TOSLINK cable──> USB SPDIF Adapter ──USB──> Raspberry Pi
```

1. Plug the Chromecast into the **HDMI input** of the audio extractor
2. Connect a **TOSLINK cable** from the extractor's optical output to the USB SPDIF adapter's optical input
3. Plug the **USB SPDIF adapter** into a USB port on the Pi
4. Power on the Chromecast (it needs its own power supply) and the HDMI extractor (if it has one)

The HDMI extractor also has an HDMI output -- you can connect that to a TV if you want video, or just leave it disconnected if you only care about audio.

---

## 5. Verify the USB Audio Device

Back in your SSH session on the Pi, check that the USB SPDIF adapter is detected:

```bash
arecord -l
```

This lists all audio recording devices. You should see your USB SPDIF adapter listed, something like:

```
card 3: ReceiverSolid [Cubilux SPDIF ReceiverSolid], device 0: USB Audio [USB Audio]
```

**Write down the card number** (in this example, `3`). You'll need it later. The device address is `plughw:3,0` (replace `3` with your card number).

Test that audio is actually coming through by recording 5 seconds:

```bash
arecord -D plughw:3,0 -f S16_LE -r 48000 -c 2 -d 5 test.wav
```

Cast something to your Chromecast while this runs, then check the file size:

```bash
ls -la test.wav
```

If it's larger than a few KB, audio is flowing through the chain. You can delete the test file:

```bash
rm test.wav
```

---

## 6. Install OwnTone

OwnTone is a music server that can stream audio to AirPlay speakers (including Sonos). We need to build it from source because the packaged version in Debian doesn't always include all features we need.

### 6.1 Install build dependencies

This installs everything needed to compile OwnTone. Copy and paste this entire block:

```bash
sudo apt install build-essential git autotools-dev autoconf libtool gettext gawk \
  gperf libantlr3c-dev libconfuse-dev libunistring-dev libsqlite3-dev \
  libavcodec-dev libavformat-dev libavfilter-dev libswscale-dev libavutil-dev \
  libasound2-dev libmxml-dev libgcrypt20-dev libavahi-client-dev zlib1g-dev \
  libevent-dev libplist-dev libsodium-dev libjson-c-dev libwebsockets-dev \
  libcurl4-openssl-dev libprotobuf-c-dev libxml2-dev flex bison ffmpeg -y
```

This will take a few minutes. These are libraries OwnTone needs for audio processing, networking, and AirPlay support.

### 6.2 Download and build OwnTone

```bash
cd ~
git clone https://github.com/owntone/owntone-server.git
cd owntone-server
autoreconf -i
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-chromecast
make
sudo make install
```

`git clone` downloads the source code. `autoreconf`, `configure`, and `make` compile it. `make install` copies the compiled program to the right system directories. This process takes 10-20 minutes on a Pi 3 (faster on newer models).

### 6.3 Create the OwnTone system user

OwnTone runs as its own user for security:

```bash
sudo useradd -r -s /bin/false owntone
sudo mkdir -p /var/cache/owntone
sudo chown -R owntone:owntone /var/cache/owntone
```

---

## 7. Configure OwnTone

### 7.1 Edit the configuration file

Open the OwnTone config file in a text editor:

```bash
sudo nano /etc/owntone.conf
```

`nano` is a simple text editor that runs in the terminal. Use arrow keys to move around. When done, press **Ctrl+O** and **Enter** to save and **Ctrl+X** to exit.

Find and change these settings (the file is long -- use **Ctrl+W** to search):

Under the `library` section:
```
directories = { "/srv/music" }
pipe_autostart = true
```

Under the `audio` section:
```
type = "disabled"
```

Here's a full example config for reference:

```
# OwnTone Configuration Example
# Full documentation: https://owntone.github.io/owntone-server/

general {
    uid = "owntone"
    logfile = "/var/log/owntone.log"
    loglevel = "log"
    db_path = "/var/cache/owntone/songs3.db"
    cache_path = "/var/cache/owntone/cache.db"
    cache_daap_threshold = 1000
    ipv6 = no
}

library {
    name = "My Music"
    port = 3689
    # Directory containing the named pipes
    directories = { "/srv/music" }
    # Auto-detect and start playing pipe sources
    pipe_autostart = true
    follow_symlinks = true
    hide_singles = false
    rating_updates = false
}

# Disable local audio output - we use AirPlay 2 only
audio {
    type = "disabled"
}

# Streaming settings for HTTP output (not used for AirPlay but good to have)
streaming {
    sample_rate = 44100
    bit_rate = 192
}
```

### 7.2 Create the music directory

This is where the audio pipes will live:

```bash
sudo mkdir -p /srv/music
```

---

## 8. Install the Bridge Script

The bridge script is the core of this project. It captures audio from the USB adapter and feeds it to OwnTone through named pipes.

### 8.1 Create the script

Create the file:

```bash
sudo nano /usr/local/bin/sonos-bridge.sh
```

Paste the following contents, then save with **Ctrl+O** and **Enter** and exit with **Ctrl+X**:

```bash
#!/bin/bash
# Sonos Bridge - Routes Chromecast audio to Sonos via OwnTone AirPlay 2
#
# This script:
# 1. Creates named pipes for OwnTone's pipe input feature
# 2. Streams USB SPDIF audio via FFmpeg with real-time priority
# 3. Continuously feeds metadata to keep OwnTone happy

# 1. Kill any ghost processes
killall ffmpeg 2>/dev/null

# 2. Clean and create the pipes OwnTone expects
rm -f /srv/music/livestream.wav*
mkfifo /srv/music/livestream.wav
mkfifo /srv/music/livestream.wav.metadata

# 3. Set permissions so OwnTone can read the pipes
chown owntone:owntone /srv/music/livestream.wav*
chmod 666 /srv/music/livestream.wav*

# 4. Metadata pipe loop - OwnTone needs this to be a FIFO, not a regular file!
# The loop keeps writing so OwnTone can always read fresh metadata
(while true; do
  echo -e "artist=Live Stream\ntitle=Chromecast Audio\nalbum=Live" > /srv/music/livestream.wav.metadata
  sleep 10
done) &

# 5. FFmpeg audio capture with skip-prevention settings:
# -thread_queue_size 16384  : Large buffer prevents underruns under CPU load
# chrt -f 10                : Real-time scheduling priority prevents dropouts
# -ar 44100                 : Sample rate (change to 48000 if you experience issues)
# plughw:3,0                : Your USB SPDIF adapter (verify with: arecord -l)
chrt -f 10 ffmpeg -y \
  -f alsa \
  -thread_queue_size 16384 \
  -i plughw:3,0 \
  -acodec pcm_s16le \
  -f wav \
  -ar 44100 \
  -ac 2 \
  /srv/music/livestream.wav
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/sonos-bridge.sh
```

### 8.2 Update the audio device

Open the script and verify the audio device matches your setup:

```bash
sudo nano /usr/local/bin/sonos-bridge.sh
```

Find the line with `plughw:3,0` near the bottom. Replace `3` with the card number you found in Step 5.

---

## 9. Set Up Auto-Start

**Systemd** is the system that manages services (background programs) on Linux. We'll create a service so the bridge starts automatically when the Pi boots.

### 9.1 Create the service file

```bash
sudo nano /etc/systemd/system/sonos-bridge.service
```

Paste the following contents:

```ini
[Unit]
Description=Sonos Chromecast Bridge
# Start BEFORE OwnTone so pipes exist when OwnTone scans the library
After=network.target
Before=owntone.service

[Service]
Type=simple
# Must run as root for chrt real-time scheduling and ALSA device access
User=root
ExecStart=/usr/local/bin/sonos-bridge.sh
# Restart automatically if the script crashes
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 9.2 Enable and start the services

Tell systemd to load the new service and enable both services to start on boot:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sonos-bridge.service
sudo systemctl enable owntone
```

**Important:** The sonos-bridge must start **before** OwnTone so the audio pipes exist when OwnTone scans for music. The service file is already configured for this.

Start everything:

```bash
sudo systemctl start sonos-bridge.service
sudo systemctl start owntone
```

Check that both are running:

```bash
sudo systemctl status sonos-bridge.service
sudo systemctl status owntone
```

Both should show `active (running)`.

---

## 10. One-Time OwnTone Setup

OwnTone has a web interface for managing speakers and playback.

1. Open a browser on your computer and go to: `http://sonos-bridge-pi.local:3689`
   (or use the Pi's IP address, e.g., `http://192.168.1.100:3689`)

2. Go to **Settings > Remotes and Outputs** -- you should see all your Sonos speakers (or other AirPlay speakers) listed here

3. Go to **Files** and find `livestream.wav`

4. Click on `livestream.wav` and press **Play**

5. Enable your Sonos speaker as the output (toggle it on in the Outputs section)

6. Cast something to your Chromecast -- you should hear it on your Sonos speakers!

If your speakers don't appear, check that the Avahi service is running (it discovers AirPlay devices on the network):

```bash
sudo systemctl status avahi-daemon
```

If it's not running:

```bash
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
sudo systemctl restart owntone
```

---

## 11. Test It

1. Open any Cast-enabled app on your phone (YouTube Music, Spotify, Jellyfin, AudioBookShelf, etc.)
2. Tap the Cast icon and select your Chromecast
3. Play something
4. Audio should come out of your Sonos speakers with barely noticeable delay

**Test play/pause:** The latency should be low enough that pause feels responsive.

**Test skip:** Skipping to the next track should work smoothly.

---

## 12. Home Assistant Automation (Optional)

If you run Home Assistant, you can automate the playback so you don't have to manually start the OwnTone stream every time you cast.

### 12.1 Create the session helper

This toggle prevents the automation from re-triggering on every track change:

1. In Home Assistant, go to **Settings > Devices & Services > Helpers**
2. Click **Create Helper > Toggle**
3. Name it: `ChromecastUltra Session Active`
4. Entity ID: `input_boolean.chromecastultra_session_active`
5. Click **Create**

### 12.2 Add the automations

Add these two automations to your Home Assistant configuration. **Update the entity IDs** to match your setup -- find them in **Developer Tools > States** and search for your devices.

**Automation 1: Start Session** -- starts OwnTone when casting begins:

```yaml
alias: "Sonos Bridge: Start Session"
description: >
  Triggers OwnTone playback handshake only at the START of a new casting session.
  Uses a session lock to prevent re-triggering on every track change.
  Requires helper: input_boolean.chromecastultra_session_active

triggers:
  - entity_id: media_player.chromecastultra0361  # UPDATE: your Chromecast entity ID
    to: playing
    trigger: state

conditions:
  - condition: state
    entity_id: input_boolean.chromecastultra_session_active
    state: "off"

actions:
  # Stop any stale OwnTone playback to clear buffer
  - target:
      entity_id: media_player.owntone_server  # UPDATE: your OwnTone entity ID
    action: media_player.media_stop

  # Brief pause to allow buffer to clear
  - delay: "00:00:01"

  # Start the livestream pipe in OwnTone
  # Note: media_content_id "11" is the database ID for livestream.wav
  # If this changes after wiping the DB, check OwnTone settings or use the API
  - target:
      entity_id: media_player.owntone_server  # UPDATE: your OwnTone entity ID
    data:
      media:
        media_content_id: "11"
        media_content_type: music
        metadata: {}
    action: media_player.play_media

  # Activate session lock to prevent re-triggering during this session
  - target:
      entity_id: input_boolean.chromecastultra_session_active
    action: input_boolean.turn_on

mode: single
```

**Automation 2: Reset Session** -- resets the lock after 2 minutes of no casting:

```yaml
alias: "Sonos Bridge: Reset Session"
description: >
  Unlocks the session after 2 minutes of no casting.
  This allows the Start Session automation to trigger again
  the next time casting begins.
  Requires helper: input_boolean.chromecastultra_session_active

triggers:
  - entity_id: media_player.chromecastultra0361  # UPDATE: your Chromecast entity ID
    from: playing
    for:
      minutes: 2
    trigger: state

actions:
  - target:
      entity_id: input_boolean.chromecastultra_session_active
    action: input_boolean.turn_off

mode: single
```

### 12.3 About the media_content_id

The automation uses `media_content_id: "11"` to tell OwnTone which file to play. This is the database ID for `livestream.wav`.

If you ever wipe the OwnTone database (e.g., during troubleshooting), this number will change. To find the new ID:

```bash
sudo sqlite3 /var/cache/owntone/songs3.db \
  "SELECT id, path FROM files WHERE path LIKE '%livestream%';"
```

---

## 13. What to Do If Something Goes Wrong

The most common issues and their fixes:

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| No audio at all | FFmpeg not running | `sudo systemctl restart sonos-bridge.service` |
| "Device busy" error | Old FFmpeg process stuck | `sudo killall -9 ffmpeg && sudo systemctl restart sonos-bridge.service` |
| Audio skips/drops | Pi CPU overloaded | Check that `chrt -f 10` is in the script, reduce other Pi workloads |
| Sonos not in OwnTone | Avahi not running | `sudo systemctl start avahi-daemon && sudo systemctl restart owntone` |
| OwnTone won't start | Config error | `sudo /usr/sbin/owntone -t` to test config |
| "not a fifo" error | Metadata file wrong type | `sudo rm /srv/music/livestream.wav.metadata && sudo systemctl restart sonos-bridge.service` |
| Card number changed | USB reordering after reboot | Run `arecord -l`, update the number in `/usr/local/bin/sonos-bridge.sh` |

For a complete troubleshooting reference, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

For a detailed account of every approach we tried before finding this solution (and why they failed), see [WHAT_WE_TRIED.md](WHAT_WE_TRIED.md).

---

**You're done!** Your Chromecast audio now streams to your Sonos speakers with near-instant latency. The bridge starts automatically when the Pi boots and recovers from crashes on its own.
