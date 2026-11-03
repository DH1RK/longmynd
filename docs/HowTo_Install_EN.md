# HowTo: Longmynd Server on Raspberry Pi 4B (64-bit)

Installation guide for Longmynd with web UI and WebSocket control
(Longmynd Client by ZR6TG, OpenTuner) on a Raspberry Pi 4B with
MiniTiouner (Express). The source is the fork `DH1RK/longmynd`. It is based
on `philcrump/longmynd` and additionally contains the fix from pull
request #2 by dl1rf. No further patch is needed (background in section 5).

Status: November 2026. Tested with Raspberry Pi OS (64-bit), GCC 14.2.0.

## Overview

| Component | Value in this setup |
|---|---|
| Hardware | Raspberry Pi 4B (4 GB), MiniTiouner Express on USB 3.0 (blue) |
| Operating system | Raspberry Pi OS (64-bit), Lite is sufficient (headless) |
| Hostname / IP | `longmynd-server` / `<RPi-IP>` (e.g. `192.168.2.158`) |
| User | `dh1rk` |
| Longmynd source | `https://github.com/DH1RK/longmynd.git` |
| Included fix | PR #2 by dl1rf (`json_object_new_double` → `json_object_new_int`), already in the fork |
| Control + status | WebSocket, port 8080 |
| Video (UDP) | set by the client, here port 4003 to the PC |

> **Why 64-bit?** The Pi 4B is a 64-bit device. "Legacy" in the Imager only
> means an older Debian release. Longmynd is plain C and runs on 32 and 64 bit.

## 1. Create the SD card

Creating the SD card with the Raspberry Pi Imager (v2.0.12) is described in
[01_RaspberryPi_Imager_EN.md](01_RaspberryPi_Imager_EN.md). Short version:
device **Raspberry Pi 4**, operating system **Raspberry Pi OS (64-bit)** (Lite
is sufficient), in the customisation set **hostname**, **username and
password**, **Wi-Fi** (if there is no LAN cable) and **enable SSH**. Then
write the card, insert it into the Pi and power on.

An Ethernet cable is more stable than Wi-Fi. If the first login fails,
test with a cable first (most common mistake: typo in SSID/password).

## 2. Find the Pi on the network and log in

> **Example values:** The login in this section shows the author's setup as
> an example: hostname `longmynd-server`, user `dh1rk`, IP `192.168.2.158`.
> Replace them with your own values from the Imager (hostname and username)
> and from your network (`<RPi-IP>`).

Windows (PowerShell):

```powershell
ping longmynd-server.local          # hostname from the Imager
ssh dh1rk@longmynd-server.local
```

`longmynd-server` is the **hostname** from the Imager (the default would be
`raspberrypi`), not the username and not the software. If the name is not
resolved: look up the IP in the router (Fritz!Box:
Home Network → Network) or check `arp -a` and connect with `ssh dh1rk@<IP>`.

### Logging in via SSH

The format is always `ssh <user>@<hostname-or-IP>`. Username and password
are the values you set in the Imager.

```powershell
ssh <user>@<hostname-or-IP>
# example with hostname:  ssh dh1rk@longmynd-server.local
# example with IP:        ssh dh1rk@192.168.2.158
```

Sequence at the first login:

1. Confirm the host key question (`Are you sure you want to continue connecting
   (yes/no/[fingerprint])?`) with `yes`. The fingerprint is stored in
   `C:\Users\<Name>\.ssh\known_hosts`.
2. Enter the password. **No characters** and no asterisks are shown.
   That is normal. Confirm with Enter.
3. The prompt `<user>@<hostname>:~ $` appears (example: `dh1rk@longmynd-server:~ $`). From here on, all
   commands of the following steps run **on the Pi**.

Log out with `exit` or `Ctrl+D`. Windows 11 already ships the OpenSSH client
(`ssh` and `scp` in PowerShell). PuTTY works as well.

If the login fails:

| Message / symptom | Cause and remedy |
|---|---|
| `Could not resolve hostname` | The `.local` name is not resolved. Connect with the IP. |
| `Connection timed out` / `refused` | Pi not booted yet, wrong IP, or SSH not enabled in the Imager. Check cable and LED. |
| `Permission denied, please try again` | Wrong username or password. The username is the one from the Imager, not the hostname. |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | SD card was re-flashed, the host key is new. Delete the old entry: `ssh-keygen -R longmynd-server.local` and, if needed, `ssh-keygen -R <RPi-IP>`. |

Optionally, assign a fixed IP in the router (DHCP reservation) so it does
not change.

### Which command runs where?

From here on you work with two windows. How to tell where you are:

| Window | Prompt | Purpose |
|---|---|---|
| **Windows PowerShell** | `PS D:\...>` | `ssh`, `scp`, browser, Wireshark, `Get-NetUDPEndpoint` |
| **SSH session to the Pi** | `<user>@<hostname>:~ $` (e.g. `dh1rk@longmynd-server:~ $`) | all commands of steps 3–10 (`apt`, `git`, `make`, `systemctl`) |

All commands in this guide that start with `sudo`, `apt`, `git`, `make`,
`systemctl`, `journalctl` or `./longmynd` run **on the Pi**.
Commands with `ssh`, `scp`, `ping`, `Test-NetConnection` and `Get-...` run
**on the Windows PC**.

### Copying files between Pi and Windows (scp)

`scp` is part of OpenSSH and copies over the SSH connection. Always start it
**from Windows** in a PowerShell, not in the SSH window. The Pi has an SSH
server, Windows does not by default. If `scp` runs on the Pi and points to a
Windows path, it fails with
`Could not resolve hostname d: Name or service not known`.

**Important:** The target path must not contain a drive letter with a colon
(`D:\...`). `scp` reads everything before the `:` as a host name. So first
change into the target folder and use `.` as the destination.

Fetch a file **from the Pi to Windows**:

```powershell
cd <target-folder>
scp <user>@<hostname-or-IP>:~/pi_history.txt .
# example:  cd D:\repos\longmynd\docs
#           scp dh1rk@longmynd-server.local:~/pi_history.txt .
```

Send a file **from Windows to the Pi** (source relative, the destination after
the `:` is the path on the Pi):

```powershell
cd <source-folder>
scp <file> <user>@<hostname-or-IP>:~/
# example:  cd D:\repos\longmynd\docs
#           scp HowTo_Install.md dh1rk@longmynd-server.local:~/
```

Copy a whole folder: option `-r`, for example
`scp -r <user>@<hostname-or-IP>:~/longmynd/web/htdocs .` (example: `scp -r dh1rk@longmynd-server.local:~/longmynd/web/htdocs .`)

Export the Pi's command history: first run `history > ~/pi_history.txt` in the
**SSH window**, then the `scp` from above in **PowerShell**.
Review the file before passing it on, because it contains every typed
command, including passwords if any appeared in a command.

## 3. Update the system, install dependencies

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git make gcc cmake libusb-1.0-0-dev libasound2-dev libjson-c-dev
```

`cmake` is needed because the Makefile builds libwebsockets from the
submodule itself. After a kernel update, run `sudo reboot` and log in again.

## 4. Get the source code

```bash
cd ~
git clone --recursive https://github.com/DH1RK/longmynd.git
cd longmynd
```

`--recursive` is mandatory, otherwise libwebsockets is missing (submodule under
`web/libwebsockets`).

## 5. Background: difference to philcrump and BATC

This section is background only. Nothing needs to be done for the
installation, the fork `DH1RK/longmynd` builds as is.

### 5.1 Origin of the code

```
myorangedragon/longmynd   (BATC original)
├── F5OEO/longmynd        (variant of the original, no web UI)
└── philcrump/longmynd    (further development with web UI and WebSocket)
    ├── dl1rf/longmynd    (fork with the fix from PR #2)
    └── DH1RK/longmynd    (this fork: philcrump + PR #2 by dl1rf)
```

- The **BATC original** and the **`F5OEO`** variant have no web UI and no
  WebSocket control and are unsuitable for this purpose.
- **`philcrump/longmynd`** brings web UI and WebSocket (port 8080).
- **`dl1rf/longmynd`** is Phil's state plus one commit (integers in the JSON).
- **`DH1RK/longmynd`** is Phil's state plus the fix by dl1rf.

### 5.2 The fix from PR #2 (`web/web.c`)

`philcrump/longmynd` sends all numbers in the WebSocket JSON as `double`
(`7.0`). The Longmynd Client and OpenTuner expect integers and otherwise show
no usable status. PR #2 (https://github.com/philcrump/longmynd/pull/2,
author dl1rf) switches `web/web.c` to `json_object_new_int`. Status at the
time of writing: the PR is **open, not merged** at Phil's. In the fork
`DH1RK/longmynd` it is included via a merge (the commit by dl1rf stays in the
history under his name).

What changes: all status values (frequency, symbol rate, MER, BER,
PIDs, constellation, ports) are output as `int`. The timestamp
is truncated to whole seconds. The source values are integers anyway.

Check that the fix is in the clone:

```bash
grep -c json_object_new_int web/web.c     # result greater than 0
```

### 5.3 Makefile: no change needed

The Makefile of the fork `DH1RK/longmynd` (unchanged from `philcrump/longmynd`) uses `-O3 -march=native
-mtune=native` and contains **no** 32-bit specific options. It builds
unchanged on aarch64.

> Only if you ever find a Makefile with the lines
> `COPT_RPI2 = -mfpu=neon-vfpv4` / `COPT_RPI34 = -mfpu=neon-fp-armv8`
> (these come from the BATC original): on 64-bit the build fails with
> `unrecognized command-line option '-mfpu=...'`. Remedy:
> ```bash
> sed -i 's/^COPT_RPI2 = .*/COPT_RPI2 =/; s/^COPT_RPI34 = .*/COPT_RPI34 =/' Makefile
> ```
> This is **not** needed for this fork.

## 6. Compile

```bash
make
ls -l longmynd
```

The first time, libwebsockets is built first (several minutes, many lines
of output). In the end `longmynd`, `fake_read` and `ts_analyse` are in the
folder. For problems (old or mixed object files) rebuild cleanly:

```bash
make clean
make
```

`make clean` deletes Longmynd and the `.o` files, but not the built
libwebsockets.

## 7. udev rule for the MiniTiouner

Unplug the MiniTiouner, then:

```bash
sudo cp minitiouner.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
```

Plug in the MiniTiouner (USB 3.0 port, blue) and check:

```bash
lsusb | grep 0403
lsusb -v -d 0403:6010 2>/dev/null | grep iProduct
```

Expected: `Future Technology Devices International ... FT2232C/D/H` and the
product name `MiniTiouner-Express` (or `USB <-> NIM tuner`). With this,
Longmynd runs without `sudo`.

## 8. Manual test run

Power supply: a good 5V/3A power supply, no other power-hungry
USB devices on the same port pair.

The QO-100 wideband beacon at **10491.5 MHz** with **1500 kS/s** serves as
test signal. Longmynd takes the intermediate frequency (IF), depending on
your LO:

| LO of your LNB | IF = 10491.5 − LO | Start value |
|---|---|---|
| 9360 MHz | 1131.5 MHz | `1131500` (kHz) |
| 9750 MHz | 741.5 MHz | `741500` (kHz) |

Example with LO 9360 MHz (as in the author's setup):

```bash
cd ~/longmynd
./longmynd -W 8080 -i 127.0.0.1 4003 -I 127.0.0.1 4002 1131500 1500
```

With LO 9750 MHz the last line reads `... 741500 1500`.

The log must show in the configuration:

```
Web interface enabled: Port 8080
```

Also `STV0910 MID = 0x51, DID = 0x20` and `found new NIM with LNAs`.
The message `No-data timeout, re-init config` is normal as long as no
matching signal is present.

Test from the PC: browser `http://<RPi-IP>:8080` or
`Test-NetConnection <RPi-IP> -Port 8080` in PowerShell.

Stop with `Ctrl+C`.

### Meaning of the parameters

| Part | Meaning |
|---|---|
| `-W 8080` | Web UI and WebSocket control on port 8080 |
| `-i 127.0.0.1 4003` | Initial destination of the transport stream (video, UDP). Client/OpenTuner override it via WebSocket. |
| `-I 127.0.0.1 4002` | Destination of the UDP status. Your client does not use it (status comes via WebSocket), but it prevents the FIFO error. |
| `1131500` | Initial IF in **kHz** (example: beacon 10491.5 MHz, LO 9360 MHz) |
| `1500` | Initial symbol rate in **kS/s** |

More options: `-w` lower NIM input, `-p v|h` LNB voltage (13 V / 18 V),
`-b` MER beep. All in `man -l longmynd.1`.

> **Why not leave out `-i` and `-I`?** Without `-I`, Longmynd writes the
> status to the FIFO `longmynd_main_status`. The file must then exist,
> otherwise the start aborts (`ERROR: Failed to open fifo`). The same applies
> to `-i` and `longmynd_main_ts`. With `127.0.0.1` there is no error case.
> The FIFO is opened non-blocking, a reader is not required.
>
> `-i` and `-I` must not have the same IP **and** the same port
> (`main.c`: *Cannot set Status IP & Port identical to TS IP & Port*).

### Calculating the frequency (IF)

Longmynd takes the intermediate frequency, not the receive frequency:

```
IF (kHz) = receive frequency (kHz) − LO (kHz)
```

Examples with the beacon at 10491.5 MHz:

- LO 9360 MHz: `10491.5 − 9360 = 1131.5 MHz = 1131500 kHz`
- LO 9750 MHz: `10491.5 − 9750 = 741.5 MHz = 741500 kHz`

The Longmynd Client and the web UI
do this conversion themselves with "Freq Offset" (`9360000`). The tuner
(STV6120) covers about 144–2450 MHz.

## 9. Autostart via systemd

The service runs as user `dh1rk`. The template
`longmynd.service.skel` in the repo does not fit (user `phil`, script
`run.sh` missing), so the file is created directly:

```bash
sudo systemctl stop longmynd 2>/dev/null   # if it already exists

sudo tee /etc/systemd/system/longmynd.service > /dev/null <<'EOF'
[Unit]
Description=Longmynd Receiver
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=dh1rk
WorkingDirectory=/home/dh1rk/longmynd
ExecStart=/home/dh1rk/longmynd/longmynd -W 8080 -i 127.0.0.1 4003 -I 127.0.0.1 4002 1131500 1500
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now longmynd
```

First stop any manually started Longmynd (`pgrep -a longmynd`),
otherwise it blocks the MiniTiouner and port 8080.

**Adjust the start values:** `1131500 1500` at the end of the `ExecStart`
line are the IF (kHz) and symbol rate (kS/s) of the beacon with **LO
9360 MHz**. If your LNB has **LO 9750 MHz**, use `741500 1500` instead
(see section 8). The values only apply at start, the client or OpenTuner
change the frequency later themselves. After a change:
`sudo systemctl daemon-reload && sudo systemctl restart longmynd`.
If the write protection from section 11 is already active, the change is lost
after a reboot. Switch it off first (11.5), change the file and enable it
again.

Check:

```bash
systemctl status longmynd
journalctl -u longmynd -n 30 --no-pager
ss -ltn | grep 8080
```

Final test: `sudo reboot`. After about a minute the web UI must be reachable
without any further action.

## 10. Update procedure

```bash
cd ~/longmynd
git pull --recurse-submodules
make
sudo systemctl restart longmynd
```

The Pi fetches updates from the fork `DH1RK/longmynd` (branch `main`). News
from Phil (`philcrump/longmynd`, branch `master`) only get there once they
have been pulled into the fork. After the update,
`grep -c json_object_new_int web/web.c` confirms that the fix by dl1rf is
still included.

## 11. After the Longmynd server has been set up successfully: protect the file system

For remote operation the Pi is only switched on and off, without a clean
shutdown. To keep the SD card from being damaged by this, the file system is
made read-only. This uses the overlay file system of Raspberry Pi OS: root
and boot are only read, and all writes go to RAM and are gone after power-off.

**Only enable it when everything is finished** (steps 1–10 complete,
autostart confirmed by a reboot test). Later changes require switching the
protection off first (see below).

### 11.1 Enable the overlay

On the Pi, either via the menu (variant a) or directly by command
(variant b).

**Variant a: via `raspi-config`**

```bash
sudo raspi-config
```

`Performance Options` → `Overlay File System` → *Enable overlay file system*
with **Yes**, and answer the question about a read-only boot partition with
**Yes** as well. Then reboot.

**Variant b: direct commands**

```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot
```

### 11.2 Verify

```bash
mount | grep -E "overlay|root-ro|root-rw"
findmnt /boot/firmware
```

Expected (abridged):

```
tmpfs-root /media/root-rw tmpfs rw,relatime 0 0
overlayroot / overlay rw,relatime,lowerdir=/media/root-ro,upperdir=/media/root-rw/overlay,... 0 0
/dev/mmcblk0p2 /media/root-ro ext4 ro,relatime 0 0
```

and for `/boot/firmware` an `ro` in the `OPTIONS` column. The SD card
(`/dev/mmcblk0p2`) is mounted `ro`, the upper layer of the overlay lives
on a `tmpfs` in RAM.

### 11.3 Functional test

`/` belongs to root, so use `sudo` (without `sudo` you get
`Permission denied`, independent of the overlay):

```bash
sudo touch /testfile && ls /testfile
ls /media/root-rw/overlay/testfile      # lives in RAM (overlay, upper layer)
ls /media/root-ro/testfile              # missing: nothing was written to the SD card
sudo reboot
```

After the reboot, `ls /testfile` reports *No such file or directory*. The
protection is then confirmed. Also check that Longmynd and the web UI start by
themselves after the reboot (`systemctl status longmynd`, browser on port 8080).
After that the Pi may be disconnected from power at any time.

### 11.4 Operating notes

- **Settings are not persistent.** Frequency, symbol rate and anything changed
  via the web UI or the client only applies until power-off.
  Start values are in the `ExecStart` line of the service (step 9).
- **Logs are lost.** They live in RAM. To keep them from filling the memory,
  the journal can be limited (`/etc/systemd/journald.conf`:
  `Storage=volatile`, `RuntimeMaxUse=20M`).
- **Clock:** The Pi has no real-time clock and only sets the time via NTP.
  This affects only timestamps in logs, not reception.
- **Storing permanently** (if needed): mount a separate small data partition
  or USB stick with `rw,noatime` and write only there.

### 11.5 Changes and updates on the server

For updates (step 10), new packages or changed services, the protection must
be switched off temporarily.

**Variant a: via `raspi-config`**

`sudo raspi-config` → `Performance Options` → `Overlay File System` →
*Disable overlay file system* (and make the boot partition writable again),
then `sudo reboot`. After the changes, enable it again the same way
(see 11.1 variant a).

**Variant b: direct commands**

```bash
sudo raspi-config nonint disable_overlayfs
sudo raspi-config nonint disable_bootro
sudo reboot
```

Then make and test the changes, and switch it on again afterwards:

```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot
```

## Continue with commissioning

Client settings, troubleshooting and key points are in
[BringingUp_EN.md](BringingUp_EN.md).
