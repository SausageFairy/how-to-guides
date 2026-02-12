# Troubleshooting Guide

## No Audio Playing

### Check FFmpeg is running
```bash
ps aux | grep ffmpeg | grep -v grep
```
If not running, start manually:
```bash
sudo systemctl restart sonos-bridge.service
```

### Check OwnTone is running
```bash
sudo systemctl status owntone
```

### Check for "Device busy" error
```bash
cat /tmp/ffmpeg.log
```
If you see "Device or resource busy":
```bash
sudo killall -9 ffmpeg
sudo systemctl restart sonos-bridge.service
```

## Audio Skipping or Dropping Out

### Verify real-time scheduling is active
```bash
ps aux | grep ffmpeg | grep -v grep
# Look for 'Sl' or 'S' status - 'T' means stopped/stuck
```

### Check for ALSA buffer underruns in FFmpeg log
```bash
cat /tmp/ffmpeg.log | grep xrun
```
If you see xrun errors, the Pi's CPU is too busy. Try:
```bash
# Reduce other Pi workloads, or increase thread_queue_size in sonos-bridge.sh
```

## OwnTone Can't Find livestream.wav

### Check pipes exist
```bash
ls -la /srv/music/
# Should show two 'p' entries (pipes):
# prw-r--r-- 1 owntone owntone 0 ... livestream.wav
# prw-r--r-- 1 owntone owntone 0 ... livestream.wav.metadata
```
If missing, restart sonos-bridge:
```bash
sudo systemctl restart sonos-bridge.service
```
Then rescan OwnTone library in the web UI.

## OwnTone Shows "Source type is pipe, but path is not a fifo" Error

This means the .metadata file is a regular file, not a pipe.

Fix:
```bash
sudo rm /srv/music/livestream.wav.metadata
sudo systemctl restart sonos-bridge.service
```

## OwnTone Web Interface Not Loading

Check if it's running:
```bash
sudo systemctl status owntone
```

If failed, check for config errors:
```bash
sudo /usr/sbin/owntone -t
```

If config error, check /etc/owntone.conf for syntax issues.

Reset and restart:
```bash
sudo systemctl reset-failed owntone
sudo systemctl start owntone
```

## OwnTone Database Corrupted / Wrong Files

Wipe the database and start fresh:
```bash
sudo systemctl stop owntone
sudo rm /var/cache/owntone/songs3.db
sudo systemctl start owntone
```
Then redo the one-time UI setup (see HOW-TO-GUIDE Step 9).

## Sonos Not Appearing in OwnTone Outputs

OwnTone uses mDNS/Avahi to discover AirPlay devices. Check Avahi is running:
```bash
systemctl status avahi-daemon
```

If not running:
```bash
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
sudo systemctl restart owntone
```

## Home Assistant Automation Not Triggering

### Check entity IDs are correct
In HA Developer Tools > States, search for:
- Your Chromecast entity (should be `media_player.something`)
- OwnTone entity (`media_player.owntone_server`)

Update the automation YAML with the correct entity IDs.

### Check media_content_id
The ID `11` for livestream.wav may change if you wipe the OwnTone database.

To find the correct ID:
```bash
sudo sqlite3 /var/cache/owntone/songs3.db \
  "SELECT id, path FROM files WHERE path LIKE '%livestream%';"
```

Update the automation with the correct ID.

### Check session lock state
In HA Developer Tools > States, check `input_boolean.chromecastultra_session_active`.
If stuck ON, turn it off manually and try casting again.

## USB SPDIF Card Number Changed After Reboot

ALSA card numbers can change between reboots. Verify current number:
```bash
arecord -l
```

Update `plughw:X,0` in `/usr/local/bin/sonos-bridge.sh` with the correct card number.

For a permanent fix, create a udev rule to assign a fixed card order.
