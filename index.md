---
title: "Autonomous Sound Installations with Raspberry Pi and Csound"
subtitle: "A step-by-step guide for macOS, Windows and Linux"
authors:
  - name: Sem
exports:
  - format: pdf
---

# Autonomous Sound Installations with Raspberry Pi and Csound

:::{admonition} What you will build
:class: tip

A Raspberry Pi that, as soon as power is connected, boots without keyboard or
monitor and automatically starts a Csound file playing multichannel audio
through a class-compliant USB audio interface. It restarts itself if anything
crashes and survives having its plug pulled at the end of the day.

No desktop, no login, no clicking. Just power.
:::

This guide assumes no prior Linux experience. Every command is given in full.
Where the three operating systems differ, you will find tabs — read only the
one for your machine.

---

(hardware)=
## 1. Hardware

:::{list-table}
:header-rows: 1
:widths: 30 70

* - Component
  - Notes
* - Raspberry Pi 3 Model B (or newer)
  - A Pi 4 or 5 has more headroom, but a Pi 3 handles eight channels of
    playback comfortably.
* - **Official power supply**
  - 5.1 V / 2.5 A with micro-USB for the Pi 3. See the warning below — this is
    not a place to improvise.
* - microSD card, 32 GB, class A1 or A2
  - 8 channels at 48 kHz/24-bit is roughly 4 GB per hour of audio. Small old
    cards are also slow and fail sooner.
* - Class-compliant USB audio interface
  - "Class-compliant" means it works without a manufacturer driver. If the
    interface can be used on an iPad, it is almost certainly class-compliant.
* - Powered USB hub *(situational)*
  - Needed if your interface draws its power from the USB bus.
* - Heatsink
  - An installation Pi usually ends up in a closed box.
:::

:::{danger} The single most common cause of failure
An underpowered supply does not announce itself as "not enough power". It
appears as random clicks in the audio, a USB interface that does not show up,
drivers that hang, and a Pi that mysteriously slows down.

Phone chargers, laptop USB ports and thin or long micro-USB cables all cause
this. The Pi 3 wants **5.1 V**, slightly above 5 V, precisely to compensate for
losses in the cable. Use the official supply with its attached cable.

You will verify this in {ref}`power-check`. Do not skip it.
:::

---

(flashing)=
## 2. Preparing the SD card

Download **Raspberry Pi Imager** from <https://www.raspberrypi.com/software/>.
It exists for macOS, Windows and Linux and works the same way on all three.

### 2.1 Choose the image

- **Device:** your Pi model
- **Operating System:** `Raspberry Pi OS (other)` → **`Raspberry Pi OS Lite (64-bit)`**
- **Storage:** your SD card

:::{note} Why Lite, and why no desktop
The desktop version runs PipeWire or PulseAudio, a sound server that sits
between Csound and your interface. It will happily downmix your eight channels
to stereo or claim the device for itself. Without a desktop, Csound talks
directly to ALSA, the kernel's audio layer, and gets the hardware to itself.

On a Pi 3 the 32-bit (`armhf`) image also works and leaves slightly more of the
1 GB of RAM free. Either is fine; this guide assumes 64-bit.
:::

### 2.2 Apply OS customisation

When Imager asks whether to apply customisation settings, choose **Edit
settings**:

:::{list-table}
:header-rows: 1
:widths: 25 75

* - Field
  - Value
* - Hostname
  - `pi` — this becomes the network name `pi.local`
* - Username
  - `pi`
* - Password
  - If the Pi ends up on a venue or festival network you do not control, use a safe password. If you are using a local network that only you can access use a simple pw like: `pi`
* - Wireless LAN
  - Your network, country `DE`
* - Locale
  - Your timezone and keyboard layout
:::

On the **Services** tab, tick **Enable SSH** and choose **Use password
authentication**.

:::{tip} Why password authentication and not the key field
Imager can also take a public key directly, but it silently pre-fills any
existing key it finds on your machine, and a pasted key that is subtly wrong
locks you out with an unhelpful `Permission denied (publickey)`.

Starting with a password and installing the key afterwards over a working
connection (see {ref}`ssh-keys`) is less fragile.
:::

Write the image, then put the card in the Pi and connect the power.

:::{important}
The first boot takes several minutes. The Pi expands the filesystem, applies
your settings, and reboots once. **`Connection refused` in the first few
minutes means "not ready yet", not "broken".** Wait and try again.
:::

---

(connecting)=
## 3. Connecting to the Pi

You will do everything from now on over **SSH**, a text connection to the Pi's
command line from your own computer.

The Pi announces itself on the local network as `pi.local` via a protocol
called mDNS, so you do not need to know its IP address.

::::{tab-set}

:::{tab-item} macOS
:sync: mac

Open **Terminal** (Applications → Utilities, or ⌘-Space and type "Terminal").

```bash
ssh pi@pi.local
```

mDNS is built into macOS, so `.local` names work out of the box.
:::

:::{tab-item} Windows
:sync: win

Open **Windows Terminal** or **PowerShell** (Start menu, type "PowerShell").
The SSH client is built into Windows 10 and 11.

```powershell
ssh pi@pi.local
```

If this fails with a name-resolution error, Windows mDNS is not cooperating.
Two fixes:

1. Install **Bonjour Print Services** from Apple, which adds `.local` support.
2. Or find the Pi's IP address in your router's device list and use that
   instead: `ssh pi@192.168.1.42`

:::{note}
If you use the IP address, it may change when the router reassigns addresses.
Set a fixed address for the Pi in your router if you go this route.
:::
:::

:::{tab-item} Linux
:sync: linux

Open a terminal.

```bash
ssh pi@pi.local
```

If the name does not resolve, install the mDNS daemon:

```bash
sudo apt install avahi-daemon   # Debian, Ubuntu, Raspberry Pi OS
sudo dnf install avahi          # Fedora
```
:::

::::

### 3.1 The fingerprint question

On the first connection you will see:

```text
The authenticity of host 'pi.local' can't be established.
ED25519 key fingerprint is SHA256:bLIadtOFOhtvot4o8aXcK2B2eCbwJ7takiZorMNJHJ4.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes`. Your computer stores the Pi's identity so it can recognise it next
time.

### 3.2 After reflashing the card

A freshly flashed Pi generates a new identity, and SSH will refuse to connect
with a large warning about a possible attack. On your own network, right after
reflashing, this is expected. Remove the stored entry:

```bash
ssh-keygen -R pi.local
```

Then connect again and answer `yes` to the fingerprint question.

:::{warning}
On an unfamiliar network this warning is worth taking seriously — it is exactly
what an intercepted connection looks like. Only dismiss it when you know why
the key changed.
:::

---

(ssh-keys)=
## 4. Logging in without a password

An SSH key is a pair of files: a **private key** that stays on your computer
and is never shared, and a **public key** you install on the Pi. Logging in then
happens automatically.

`ed25519` names the cryptographic algorithm — it is the current default,
secure, fast, and produces short keys.

### 4.1 Create a key (skip if you already have one)

Run this **on your own computer**, not on the Pi:

```bash
ssh-keygen -t ed25519
```

Press Enter to accept the default location. The passphrase is optional; it
encrypts the private key on disk so that someone who copies the file cannot
use it.

### 4.2 Install it on the Pi

::::{tab-set}

:::{tab-item} macOS
:sync: mac

```bash
ssh-copy-id pi@pi.local
```

Enter your Pi password once. That is all.
:::

:::{tab-item} Windows
:sync: win

Windows has no `ssh-copy-id`, so send the key over the existing connection:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh pi@pi.local "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Enter your Pi password when asked.
:::

:::{tab-item} Linux
:sync: linux

```bash
ssh-copy-id pi@pi.local
```

Enter your Pi password once. That is all.
:::

::::

From now on `ssh pi@pi.local` logs you straight in.

:::{note}
You will still be asked for the password by `sudo`, which runs commands with
administrator rights. That is a separate confirmation and is normal.
:::

### 4.3 A shortcut worth setting up

Create or edit the file `~/.ssh/config` on your computer (on Windows:
`C:\Users\<you>\.ssh\config`):

```text
Host pi
    HostName pi.local
    User pi
```

Now `ssh pi` is enough.

---

(first-steps)=
## 5. First steps on the Pi

Everything from here runs **on the Pi**, in your SSH session.

### 5.1 Silence the locale warnings (macOS and some Linux users)

If every login is followed by a wall of
`setlocale: LC_CTYPE: cannot change locale (UTF-8)`, your computer is sending a
locale name the Pi does not understand. Tell the Pi to ignore it:

```bash
sudo sed -i 's/^AcceptEnv LANG LC_\*/#&/' /etc/ssh/sshd_config
sudo systemctl restart ssh
exit
```

Log back in and the warnings are gone.

### 5.2 Update and install

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y csound alsa-utils
```

On a Pi 3 the upgrade takes a while. Check the result:

```bash
csound --version
```

:::{note} A harmless warning
Csound on Debian prints
`could not open library ... liblinear_algebra.so: undefined symbol: dorgqr_`
on every start. It is a packaging fault in an optional plugin you will not use.
Ignore it.
:::

### 5.3 Create a working folder

```bash
mkdir -p ~/installation
```

(power-check)=
### 5.4 Check the power supply

This is the diagnostic that explains most later mysteries:

```bash
vcgencmd get_throttled
```

:::{list-table}
:header-rows: 1
:widths: 25 75

* - Result
  - Meaning
* - `throttled=0x0`
  - Everything fine.
* - `throttled=0x50000`
  - The voltage has dropped and the Pi has slowed itself down since boot.
* - `throttled=0x50005`
  - It is happening **right now**.
:::

Also look at the kernel log:

```bash
sudo dmesg | grep -i voltage
```

Any `Undervoltage detected!` line means the supply or cable is inadequate.
Replace it before doing any audio work. The flags reset on every reboot, so
check again after a few minutes of running.

---

(audio)=
## 6. Finding the audio interface

Plug the interface in, then:

```bash
cat /proc/asound/cards
aplay -l
```

You should see your interface alongside the Pi's built-in outputs:

```text
 0 [Headphones     ]: bcm2835_headpho - bcm2835 Headphones
 1 [vc4hdmi        ]: vc4-hdmi - vc4-hdmi
 2 [US16x08        ]: USB-Audio - Tascam US-16x08
```

The name in brackets is the **card ID**. Note it down.

:::{important} Never use `hw:1,0`
Card numbers can change between boots. If your installation refers to card 1
and the interface comes up as card 2 one morning, it plays into the void.

Always address the device by name: `hw:CARD=US16x08,DEV=0`.
:::

Check what the device actually supports (substitute your card number):

```bash
cat /proc/asound/card2/stream0
```

This lists the sample rates, bit depths and channel counts. **Your Csound `sr`
must match one of the rates listed here.**

### 6.1 If the interface does not appear

```bash
lsusb
sudo dmesg | tail -n 30
```

- **Missing from `lsusb` entirely:** it is not reaching the Pi. Check that it is
  switched on, that its own power supply is connected, and try another cable
  (some USB cables are charge-only). A bus-powered interface may simply be
  drawing more than the Pi can give — use a powered hub.
- **Present in `lsusb` but not in `aplay -l`:** the Pi sees the device but it is
  not presenting itself as class-compliant audio. Some interfaces have a switch
  or firmware setting for this.

### 6.2 Test before involving Csound

```bash
speaker-test -D hw:CARD=US16x08,DEV=0 -c 8 -t wav -l 1
```

You will hear a voice announcing each channel in turn. Note which physical
speaker responds to which announcement — this is how you verify your wiring.

:::{tip} Developing without the interface
You can build and test the entire autostart chain on the Pi's headphone jack
and swap in the interface later. Only three things change: the device name in
`-odac:`, `nchnls`, and the output opcode.

The onboard jack needs `plughw:` instead of `hw:` (it is fussy about sample
formats, and `plug` converts automatically), and it crackles slightly on its
own — it is PWM-based, not a real DAC.
:::

### 6.3 Output level

```bash
amixer -c Headphones sset PCM 80%,80%
```

USB interfaces often have no software gain at all, only hardware knobs. List
what yours offers:

```bash
amixer -c US16x08 scontrols
```

---

(csound)=
## 7. The Csound file

### 7.1 Buffer settings

Two options govern glitch-free playback:

- `-b` — the **software** buffer, how much Csound computes per block
- `-B` — the **hardware** buffer, how much ALSA holds

```text
-b256 -B1024
--realtime
```

Keep `-b` a multiple of `ksmps` and smaller than `-B`. Latency is irrelevant for
an installation, so if you hear dropouts, raise both. `--realtime` moves
initialisation work onto a separate thread so it cannot interrupt audio output.

:::{note} What "not allowed on this device" means
Csound may print

```text
ALSA: -b 512 not allowed on this device; using 1024 instead
```

This is not a syntax error. ALSA offered the nearest size the driver accepts.
The line that matters is `ALSA output: total buffer size: ...` — that is what
is actually running.
:::

:::{warning} Avoid `dmix`
`dmix` is an ALSA layer that mixes several programs onto one card. It resamples,
runs at a fixed rate, and can silently change your pitch. For an installation
you want one program owning the device: use `hw:` (or `plughw:` for the onboard
jack) and nothing else.
:::

### 7.2 A minimal 8-channel player

```{code-block} xml
:caption: play.csd
:linenos:

<CsoundSynthesizer>
<CsOptions>
-+rtaudio=alsa
-odac:hw:CARD=US16x08,DEV=0
-b256 -B1024
--realtime
-d -m7
</CsOptions>
<CsInstruments>
sr     = 48000
ksmps  = 128
nchnls = 8
0dbfs  = 1

instr 1
  prints "installation started\n"
  a1,a2,a3,a4,a5,a6,a7,a8 diskin2 "installation.wav", 1, 0, 1
  outc a1,a2,a3,a4,a5,a6,a7,a8
endin
</CsInstruments>
<CsScore>
i1 0 z
e
</CsScore>
</CsoundSynthesizer>
```

- `diskin2` arguments after the filename: **speed**, **start position**,
  **loop** (1 = on).
- `z` as the note duration means "indefinitely".
- `-m7` prints buffer-underrun warnings. Use it while testing;
  `-m0` silences them.
- The filename is relative to the service's working directory
  (see {ref}`autostart`), so a bare filename works.

:::{tip} Fade your loop points
If the file loops, or if the service restarts Csound, an abrupt start clicks.
Fade in and out at the seams, either in the audio file or with an envelope in
Csound.
:::

---

(transfer)=
## 8. Getting files onto the Pi

Run these **on your own computer**.

::::{tab-set}

:::{tab-item} macOS
:sync: mac

```bash
rsync -avh --progress ~/path/to/audio/ pi@pi.local:/home/pi/installation/
```

The trailing slash on the source copies the folder's *contents*; without it the
folder itself is nested inside. Running it again transfers only what changed.
:::

:::{tab-item} Windows
:sync: win

Windows has no `rsync`, but `scp` is built in:

```powershell
scp C:\path\to\audio\*.wav pi@pi.local:/home/pi/installation/
scp C:\path\to\play.csd pi@pi.local:/home/pi/installation/
```

For drag-and-drop, **WinSCP** (free) connects via SFTP to `pi.local` with user
`pi`. If you use WSL, `rsync` is available there and behaves as on Linux.
:::

:::{tab-item} Linux
:sync: linux

```bash
rsync -avh --progress ~/path/to/audio/ pi@pi.local:/home/pi/installation/
```

The trailing slash on the source copies the folder's *contents*; without it the
folder itself is nested inside. Running it again transfers only what changed.
:::

::::

Check the space you have left, on the Pi:

```bash
df -h /
```

:::{note}
Use uncompressed WAV or AIFF. Decoding MP3 or FLAC costs CPU you would rather
spend on audio. The file's sample rate must match your `sr`, or `diskin2` plays
it at the wrong speed.
:::

---

(autostart)=
## 9. Autostart

`systemd` is Linux's service manager. You describe what should run, and it
starts it at boot and restarts it if it dies.

### 9.1 Create the service

On the Pi, paste this as a single block:

```{code-block} bash
:caption: Creating /etc/systemd/system/installation.service

sudo tee /etc/systemd/system/installation.service << 'EOF'
[Unit]
Description=Csound sound installation
After=sound.target
Wants=sound.target

[Service]
Type=simple
User=pi
Group=audio
WorkingDirectory=/home/pi/installation
ExecStartPre=/bin/bash -c 'until [ -e /proc/asound/US16x08 ]; do sleep 1; done'
ExecStart=/usr/bin/csound /home/pi/installation/play.csd
Restart=always
RestartSec=5
TimeoutStopSec=5
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=80
LimitRTPRIO=95
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Replace `US16x08` and `play.csd` with your own card ID and filename.

:::{admonition} What each line does
:class: dropdown

`ExecStartPre` with the `until` loop
: Waits for the audio device to actually appear. USB enumeration is slower than
  systemd reaching the point where it starts services, and a fixed `sleep 10`
  is a guess. This waits exactly as long as needed.

`Restart=always` / `RestartSec=5`
: If Csound dies for any reason, it comes back five seconds later.

`TimeoutStopSec=5`
: Csound occasionally hangs while closing an audio device. This kills it after
  five seconds instead of waiting forever.

`CPUSchedulingPolicy=fifo` / `CPUSchedulingPriority=80`
: Real-time scheduling. Other processes — SSH, networking, logging — can no
  longer push Csound aside. This is often the difference between clean playback
  and occasional clicks.

`LimitRTPRIO` / `LimitMEMLOCK`
: Permission to use that priority. Note that `/etc/security/limits.conf` does
  **not** apply to systemd services, which is why it must be set here.

`User=pi` / `Group=audio`
: Runs as your normal user, in the group that has access to audio devices.
  Nothing here needs to run as root.
:::

If you need to set the output level at every start, add another line before
`ExecStart`:

```text
ExecStartPre=/usr/bin/amixer -c Headphones sset PCM 80%,80%
```

### 9.2 Activate

```bash
sudo systemctl daemon-reload
sudo systemctl enable installation
sudo systemctl start installation
```

:::{caution}
`enable` and `start` are different things. `start` runs it now; **`enable` is
what makes it start at boot.** Forgetting `enable` is the most common reason an
autostart "stops working" after reflashing.
:::

### 9.3 Watch and control it

```bash
systemctl status installation          # is it running?
systemctl is-enabled installation      # will it start at boot?
journalctl -u installation -f          # live log, Ctrl+C to leave
journalctl -u installation -b          # everything since this boot
```

Anything Csound prints — including your own `prints` and `printks` messages —
appears in the journal. Count underruns since boot with:

```bash
journalctl -u installation -b --no-pager | grep -c -i underrun
```

Stop and start it:

```bash
sudo systemctl stop installation
sudo systemctl restart installation
sudo systemctl disable installation    # also stop it starting at boot
```

:::{important}
The audio device can only be used by one program at a time. **Always
`sudo systemctl stop installation` before running Csound by hand**, or one of
the two will fail with "Device or resource busy".
:::

### 9.4 The real test

```bash
sudo reboot
```

Do not log in. Just wait. If the sound starts by itself after a minute, the
installation behaves correctly.

---

(hardening)=
## 10. Making it installation-proof

Do this **last**, after everything works and the piece is final.

### 10.1 System tuning

Edit `/boot/firmware/config.txt`:

```bash
sudo nano /boot/firmware/config.txt
```

Add at the end:

```{code-block} ini
:caption: /boot/firmware/config.txt

dtparam=audio=off
dtoverlay=disable-bt
dtoverlay=disable-wifi
dtparam=watchdog=on
arm_freq_min=1200
```

:::{list-table}
:header-rows: 1
:widths: 35 65

* - Setting
  - Why
* - `dtparam=audio=off`
  - Removes the onboard audio device so it cannot be picked up by mistake. Leave
    this out while you are still testing on the headphone jack.
* - `dtoverlay=disable-bt`
  - You do not need Bluetooth, and it costs power and interrupts.
* - `dtoverlay=disable-wifi`
  - Leave out if you want SSH access on site. Then disable Ethernet instead —
    there is no reason to run both.
* - `arm_freq_min=1200`
  - Stops the CPU clocking down when idle and failing to ramp back up in time.
    A classic source of "random" clicks. Use this rather than `force_turbo=1`,
    which sets a permanent hardware bit.
* - `dtparam=watchdog=on`
  - Hardware watchdog, see below.
:::

In `/boot/firmware/cmdline.txt` — this is **one single line**, append to the
end of it, do not add a new line:

```text
usbcore.autosuspend=-1 snd_usb_audio.nrpacks=1
```

The first stops the Pi putting your interface to sleep. The second makes the
USB audio driver submit one packet at a time instead of batching — slightly
more CPU, considerably more stability on a Pi 3.

### 10.2 Watchdog

If the Pi ever freezes, the watchdog reboots it. In
`/etc/systemd/system.conf`, find and set:

```text
RuntimeWatchdogSec=15
```

The Pi must tell the watchdog it is alive every 15 seconds. If it cannot, the
hardware resets it, and `Restart=always` brings your piece back.

### 10.3 Turn off background jobs

Automatic updates wake up, use CPU, and can change software under an
installation that is supposed to be frozen:

```bash
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer man-db.timer
```

### 10.4 Read-only filesystem

This is the step people skip and then regret. An installation gets switched off
at the wall every night, and an SD card written to mid-flush eventually
corrupts.

```bash
sudo raspi-config
```

→ **Performance Options** → **Overlay File System** → enable, and also set the
boot partition read-only.

The system then runs from a RAM overlay and the card is never written. Pulling
the plug becomes harmless.

:::{caution} Updating afterwards
With the overlay on, **anything you change is lost at the next reboot** —
including files you copy over. To make changes:

```bash
sudo raspi-config nonint do_overlayfs 1   # disable overlay
sudo reboot
# copy files, test
sudo raspi-config nonint do_overlayfs 0   # re-enable
sudo reboot
```
:::

### 10.5 Before you leave it alone

- Let it run for 24 hours, then check the log for underruns and the power flags
  with `vcgencmd get_throttled`.
- **Cut the power ten times in a row** and confirm it comes back every time.
  That, not a single successful boot, is the acceptance test for an
  installation.
- Write down the card ID, the filenames and the hostname, and tape them inside
  the box.

---

(troubleshooting)=
## 11. Troubleshooting

:::{admonition} `Connection refused`
:class: dropdown

The Pi is reachable but nothing is listening on the SSH port.

1. **Wait.** On first boot this is normal for several minutes.
2. If it persists, SSH was not enabled in Imager. You do not need to reflash:
   put the card in your computer, and on the `bootfs` volume create an empty
   file named exactly `ssh` (no extension). On the next boot SSH will be
   enabled.
:::

:::{admonition} `REMOTE HOST IDENTIFICATION HAS CHANGED!`
:class: dropdown

Expected after reflashing the card. Run `ssh-keygen -R pi.local` and reconnect.
See {ref}`connecting`.
:::

:::{admonition} `Permission denied (publickey)`
:class: dropdown

The Pi accepts only keys, and none of yours matches. Either reflash choosing
password authentication and then use {ref}`ssh-keys`, or plug in a monitor and
keyboard and fix `~/.ssh/authorized_keys` locally.
:::

:::{admonition} Clicks in the audio
:class: dropdown

First determine which kind:

```bash
journalctl -u installation -f
```

**With `Buffer underrun` warnings** → the audio chain cannot keep up:

1. Check `vcgencmd get_throttled`. Anything but `0x0` means power, and no
   setting will fix it.
2. Raise `-B`, then lower `-b` (smaller, more frequent refills often help more
   than a bigger total buffer).
3. Confirm the service is using real-time priority: in `top`, the `PR` column
   for csound should show a negative number such as `-81`.
4. Stop working over SSH while listening. Wi-Fi traffic causes interrupt spikes
   and some clicks during testing are caused by your own session.

**Without underrun warnings** → the clicks are in your piece. Look for abrupt
starts, loop seams, or signals exceeding `0dbfs`.
:::

:::{admonition} No sound, but Csound reports no error
:class: dropdown

Check the `overall amps` line Csound prints when it exits. If it shows non-zero
values, Csound produced audio and it was lost after that.

- Verify the device independently: `speaker-test -D <your device> -c 2 -t wav -l 1`
- Check the mixer is not muted: `amixer -c <card> sget PCM`
- If the driver has previously logged errors, reboot. Audio drivers can get
  stuck in a state where they accept audio and play nothing.
:::

:::{admonition} Csound hangs and will not quit
:class: dropdown

Csound sometimes hangs while closing the audio device.

- **Ctrl+C** in the running terminal.
- If that fails, open a **second SSH session** and run `pkill -9 csound`. Typing
  it into the frozen terminal does nothing — it is just text.
- To close a frozen SSH connection, press **Enter**, then **`~`**, then **`.`**

Keep a second SSH session open while testing. `TimeoutStopSec=5` in the service
handles this case automatically.
:::

:::{admonition} `Killed`
:class: dropdown

Something terminated Csound from outside — usually a `pkill -9` from another
window. If it happens unprovoked, check for out-of-memory messages:

```bash
sudo dmesg | tail -n 20
```
:::

:::{admonition} The autostart stopped working
:class: dropdown

```bash
systemctl is-enabled installation
systemctl status installation
journalctl -u installation -b --no-pager | tail -n 30
```

If `is-enabled` says `disabled`, run `sudo systemctl enable installation`.
This is by far the most common cause.
:::

---

(appendix)=
## 12. Appendix: test files

### 12.1 Stereo channel test

```{code-block} xml
:caption: noise_stereo.csd

<CsoundSynthesizer>
<CsOptions>
-+rtaudio=alsa
-odac:plughw:CARD=Headphones,DEV=0
-b256 -B1024
--realtime
-d -m7
</CsOptions>
<CsInstruments>
sr     = 48000
ksmps  = 128
nchnls = 2
0dbfs  = 1

instr 1
  ichan = p4
  aenv  transeg 0, p3/2, 2, 1, p3/2, -2, 0
  anz   noise 0.7, 0
  printf_i "channel %d\n", 1, ichan
  outch ichan, anz * aenv
endin
</CsInstruments>
<CsScore>
i1 0 5 1
i1 5 5 2
r 1000
e
</CsScore>
</CsoundSynthesizer>
```

### 12.2 Eight-channel wiring test

A five-second noise swell on each channel in turn. Play it while walking
through the room to verify that each speaker is where you think it is —
`printf_i` prints the channel number in the log as each burst begins.

```{code-block} xml
:caption: noise_8ch.csd

<CsoundSynthesizer>
<CsOptions>
-+rtaudio=alsa
-odac:hw:CARD=CHANGEME,DEV=0
-b256 -B1024
--realtime
-d -m7
</CsOptions>
<CsInstruments>
sr     = 48000
ksmps  = 128
nchnls = 8
0dbfs  = 1

instr 1
  ichan = p4
  aenv  transeg 0, p3/2, 2, 1, p3/2, -2, 0
  anz   noise 0.7, 0
  printf_i "channel %d\n", 1, ichan
  outch ichan, anz * aenv
endin
</CsInstruments>
<CsScore>
i1  0 5 1
i1  5 5 2
i1 10 5 3
i1 15 5 4
i1 20 5 5
i1 25 5 6
i1 30 5 7
i1 35 5 8
r 1000
e
</CsScore>
</CsoundSynthesizer>
```

Replace `CHANGEME` with your card ID from `cat /proc/asound/cards`.

`r 1000` repeats the whole section 1000 times, so it loops for hours instead of
stopping after one pass — useful when you are still at the far end of the room.
Remove it for a single pass.

### 12.3 A synthesis example: self-modulating sine

Not everything needs a soundfile. This is a feedback-FM oscillator: each
oscillator's phase is modulated by its own previous output. At zero feedback it
is a pure sine; as feedback rises it grows towards a sawtooth-like tone and
eventually breaks into noise. The feedback amount drifts on its own, so the
sound keeps evolving indefinitely — useful for an installation that must not
repeat audibly.

```{code-block} xml
:caption: fbsine.csd

<CsoundSynthesizer>
<CsOptions>
-+rtaudio=alsa
-odac:plughw:CARD=Headphones,DEV=0
-b256 -B1024
--realtime
-d -m7
</CsOptions>
<CsInstruments>
sr     = 48000
ksmps  = 128
nchnls = 2
0dbfs  = 1

; feedback sine: phase modulated by its own output, sample by sample
opcode FbSine, a, kk
  kfreq, kfb xin
  setksmps 1
  kphs   init 0
  kprev1 init 0
  kprev2 init 0
  kout   = sin(2 * $M_PI * kphs + kfb * (kprev1 + kprev2) * 0.5)
  kprev2 = kprev1
  kprev1 = kout
  kphs   += kfreq / sr
  kphs   = kphs - int(kphs)
  xout a(kout)
endop

instr 1
  prints "self-modulating sine started\n"
  aenv linseg 0, 3, 1

  ; feedback amount drifts slowly between pure sine and saw-like
  kfbL randomi 0, 1.3, 0.07
  kfbR randomi 0, 1.3, 0.05

  ; slight independent pitch drift per channel
  kdrL randomi -0.004, 0.004, 0.11
  kdrR randomi -0.004, 0.004, 0.13

  aL FbSine 110 * (1 + kdrL), kfbL
  aR FbSine 165 * (1 + kdrR), kfbR

  printks "feedback L %.2f  R %.2f\n", 5, kfbL, kfbR

  outs aL * 0.4 * aenv, aR * 0.4 * aenv
endin
</CsInstruments>
<CsScore>
i1 0 z
e
</CsScore>
</CsoundSynthesizer>
```

:::{admonition} How it works
:class: dropdown

The feedback has to happen **sample by sample**, which ordinary Csound code at
`ksmps = 128` cannot do. That is why the oscillator lives in a user-defined
opcode with `setksmps 1`: inside it, every k-rate line runs once per sample,
while the rest of the orchestra stays efficient at 128.

Averaging the last **two** output samples, rather than feeding back only the
last one, is the classic trick from Tomisawa's original feedback-FM patent. It
stops the loop oscillating between two states at high feedback values.

Left sits at 110 Hz, right a fifth above at 165 Hz, each with its own slow
feedback and pitch drift, so the two channels never quite line up. The
`randomi` rates of 0.05–0.13 Hz mean new target values every 8 to 20 seconds,
smoothly interpolated. The three-second `linseg` fade-in prevents a click
whenever the service restarts it.

To change the character: the upper feedback limit (`1.3`) is the main handle.
Towards 1.6–2 it becomes noisy; below 1 it stays warm and filtered-sounding.
:::

---

## 13. Quick reference

```{code-block} bash
:caption: On your own computer

ssh pi@pi.local                       # connect
ssh-keygen -R pi.local                # after reflashing
rsync -avh --progress ~/audio/ pi@pi.local:/home/pi/installation/
```

```{code-block} bash
:caption: On the Pi

cat /proc/asound/cards                # which audio devices exist
aplay -l                              # playback devices in detail
cat /proc/asound/card1/stream0        # supported rates and channels
speaker-test -D hw:CARD=X,DEV=0 -c 8 -t wav -l 1

vcgencmd get_throttled                # 0x0 = power is fine
df -h /                               # disk space
top                                   # CPU load

sudo systemctl stop installation      # before any manual test
sudo systemctl restart installation
sudo systemctl enable installation    # start at boot
systemctl status installation
journalctl -u installation -f         # live log
```
