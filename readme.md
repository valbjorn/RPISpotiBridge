``` markdown
# Headless Spotify Connect to Bluetooth Speaker (Raspberry Pi Zero W)

This guide explains how to turn a Raspberry Pi Zero W into a lightweight, headless Spotify Connect receiver that routes audio directly to a Bluetooth speaker. 

By using `bluez-alsa` and a specific "slim" version of `spotifyd`, we bypass heavy audio servers like PulseAudio. This keeps CPU and RAM usage minimal, making it perfect for the Pi Zero W. It also includes an auto-connect service so the Pi automatically reconnects when you turn on your speaker.

## Prerequisites
* A Raspberry Pi Zero W running **Raspberry Pi OS Lite** (headless).
* A Bluetooth speaker.
* A Spotify Premium account (required by `librespot`/`spotifyd`).

---

## Step 1: Install ALSA Bluetooth Bridge
First, update your system and install the utility that bridges Linux ALSA audio with Bluetooth.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install bluez-alsa-utils -y

```

## Step 2: Pair Your Bluetooth Speaker

Put your Bluetooth speaker into pairing mode and connect it via the terminal.

1. Open the Bluetooth control tool:
```bash
sudo bluetoothctl

```


2. Run the following commands one by one:
```text
power on
agent on
scan on

```


3. Wait for your speaker to appear and note its MAC address (e.g., `XX:XX:XX:XX:XX:XX`).
4. Pair, trust, and connect:
```text
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX

```


5. Type `quit` to exit.

## Step 3: Route Default Audio to Bluetooth

We need to tell the ALSA audio system to route the default audio stream to your Bluetooth speaker.

1. Create or edit the ALSA configuration file:
```bash
sudo nano /etc/asound.conf

```


2. Paste the following configuration. **Important:** Replace `XX:XX:XX:XX:XX:XX` with your speaker's actual MAC address, and keep the quotes around it:
```text
defaults.bluealsa.interface "hci0"
defaults.bluealsa.device "XX:XX:XX:XX:XX:XX"
defaults.bluealsa.profile "a2dp"

pcm.!default {
    type plug
    slave.pcm "bluealsa"
}

```


3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

## Step 4: Install Spotifyd (v0.3.5 armv6-slim)

*Note: We must use v0.3.5 because newer versions dropped support for the ARMv6 architecture used in the Pi Zero W. The "slim" version is ideal as it relies purely on ALSA.*

1. Download and extract the binary:
```bash
wget [https://github.com/Spotifyd/spotifyd/releases/download/v0.3.5/spotifyd-linux-armv6-slim.tar.gz](https://github.com/Spotifyd/spotifyd/releases/download/v0.3.5/spotifyd-linux-armv6-slim.tar.gz)
tar -xvzf spotifyd-linux-armv6-slim.tar.gz

```


2. Move it to the system binaries folder:
```bash
sudo mv spotifyd /usr/local/bin/

```



## Step 5: Configure Spotifyd

Because a Bluetooth speaker usually lacks a hardware PCM mixer exposed to Linux, we must tell `spotifyd` to use software volume control (`softvol`). Otherwise, playback will crash immediately.

1. Create the configuration file:
```bash
sudo nano /etc/spotifyd.conf

```


2. Paste the following:
```ini
[global]
backend = "alsa"
device = "default" 
volume_controller = "softvol"
device_name = "Pi_Zero_Bluetooth"
bitrate = 160

```


3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

## Step 6: Create the Spotifyd Systemd Service

To ensure `spotifyd` runs automatically in the background on boot:

1. Create a service file:
```bash
sudo nano /etc/systemd/system/spotifyd.service

```


2. Paste the following:
```ini
[Unit]
Description=A spotify playing daemon
Documentation=[https://github.com/Spotifyd/spotifyd](https://github.com/Spotifyd/spotifyd)
Wants=sound.target
After=sound.target
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/spotifyd --no-daemon
Restart=always
RestartSec=12

[Install]
WantedBy=default.target

```


3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

## Step 7: Create the Bluetooth Auto-Connect Service

The Raspberry Pi will not aggressively seek out the Bluetooth speaker if it is turned off and on again. This service continuously checks the connection and reconnects automatically.

1. Create the auto-connect service file:
```bash
sudo nano /etc/systemd/system/bt-autoconnect.service

```


2. Paste the following. **Important:** Replace `XX:XX:XX:XX:XX:XX` with your speaker's MAC address (it appears **twice** in the `ExecStart` line):
```ini
[Unit]
Description=Bluetooth Auto-Connect Loop
After=bluetooth.service
Requires=bluetooth.service

[Service]
Type=simple
ExecStart=/bin/bash -c 'while true; do if ! bluetoothctl info XX:XX:XX:XX:XX:XX | grep -q "Connected: yes"; then bluetoothctl connect XX:XX:XX:XX:XX:XX; fi; sleep 15; done'
Restart=always

[Install]
WantedBy=multi-user.target

```


3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

## Step 8: Enable and Start Services

Finally, reload the system daemon and start up your new services.

```bash
sudo systemctl daemon-reload
sudo systemctl enable bt-autoconnect.service
sudo systemctl enable spotifyd.service
sudo systemctl start bt-autoconnect.service
sudo systemctl start spotifyd.service

```

**You are done!** Open Spotify on your phone or computer, select `Pi_Zero_Bluetooth` from your devices, and start streaming.

---

## Appendix: How to Change the Bluetooth Speaker

If you get a new speaker or want to connect the Pi to a different device, you need to update the MAC address in two places.

1. Pair the new speaker using `bluetoothctl` (See Step 2).
2. Update the ALSA bridge (`/etc/asound.conf`) with the new MAC address.
3. Update the Auto-Connect service (`/etc/systemd/system/bt-autoconnect.service`) with the new MAC address in both spots.
4. Restart the services to apply the changes:
```bash
sudo systemctl daemon-reload
sudo systemctl restart bt-autoconnect.service
sudo systemctl restart spotifyd.service

```



```

```
