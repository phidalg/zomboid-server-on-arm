# 🧟‍♂️ Zomboid Server on ARM
[![Star this repo](https://img.shields.io/github/stars/phidalg/zomboid-server-on-arm?style=social)](https://github.com/phidalg/zomboid-server-on-arm/stargazers)
[![Follow me](https://img.shields.io/github/followers/phidalg?style=social)](https://github.com/phidalg)
[![License](https://img.shields.io/github/license/phidalg/zomboid-server-on-arm)](https://github.com/phidalg/zomboid-server-on-arm/blob/main/LICENSE)

> Forked from [juanbravozu/zomboid-server-on-arm](https://github.com/juanbravozu/zomboid-server-on-arm). See [What's different in this fork](#whats-different-in-this-fork) below.  
> juanbravozu/zomboid-server-on-arm is a fork from [Dyarven/zomboid-server-on-arm](https://github.com/Dyarven/zomboid-server-on-arm)

A pair of bash scripts to ease the set up of a **Project Zomboid server** on ARM64 devices like Raspberry Pi or OCI Ampere Arm-based cloud instances using an emulation layer.

---

## Features
- **Automated Setup:** Installs everything you need for a functioning Zomboid server set up as a systemd service.
- **ARM64 Compatible:** Configures an emulation layer with Box64 to run the Zomboid server binary.
- **Version Selection:** You can specify the release of Project Zomboid you want to install (including beta branches).
- **Reliable downloads:** SteamCMD runs on a regular x86_64 machine instead of under ARM emulation (see below for why).

## What's different in this fork

The original script ran **SteamCMD itself** under ARM emulation (via Box86) to download the server files directly on the ARM box. In practice, SteamCMD reliably hangs forever at `[0%] Checking for available update...` under Box86 - this is a long-standing, unresolved compatibility issue reported repeatedly against the Box86 project across different hardware (Raspberry Pi, Orange Pi, Oracle Ampere instances), not something specific to any one setup.

This fork works around it: SteamCMD now runs **natively** on a normal x86_64 machine (no emulation involved for that part) to download/update the server files, which are then transferred to the ARM server. The ARM server only needs Box64 - which is considerably more mature and reliable than Box86 - to actually run the already-downloaded Java server binary. Box86 isn't used anywhere in this fork.

This means setup now uses **two machines** and **two scripts** instead of one - see the guide below.

A few smaller fixes are also rolled in from testing on Ubuntu 24.04 / Oracle Ampere:
- Added `set -euo pipefail` and error trapping so the script stops on the first real failure instead of plowing ahead silently.
- Fixed inconsistent `sudo` usage (the original mixed bare and `sudo`-prefixed privileged commands).
- Fixed `libncurses5:armhf`, which was removed entirely from Ubuntu's repos as of 24.04 - moot now anyway since this fork no longer needs `armhf`/Box86 packages at all.
- Removed a broken `cp box32 /usr/local/bin/` line left over in the Box86 build step of the original (that binary was never produced by the Box86 build - unrelated to this fork's changes, but worth knowing if you're comparing against upstream).
- Added a wait-for-apt-lock check, since fresh Ubuntu 24.04 images run `unattended-upgrades` aggressively right after boot and commonly collide with the script's own `apt` calls.
- Fixed the Box64 cmake flag to the correct generic `-DARM_DYNAREC=ON` (the original used a Raspberry-Pi-specific flag that doesn't apply to something like an Ampere cloud instance).

## Important things to know
- Ideally you'd create a **"zomboid"** user and run the scripts/service as that user.
- **You now need two machines**: any x86_64 (Intel/AMD) Linux box - your PC, a spare machine, or a temporary throwaway x86 cloud VM - to download the server files, and your ARM64 server to actually run it.
- Neither script should be run as `root` or with `sudo` in front of it - run them as a normal user with sudo privileges; the scripts call `sudo` internally whenever they need it.
- The first time you run the actual Zomboid server, you must launch it manually so you can set an admin password. Once you run the ARM setup script and it reaches that point, it will tell you to do so.
  - _You could theoretically automate this by setting a custom servername and providing an ini file, but since this is a PoC and often unstable, the manual step is kept._
- This script opens ports 16261 and 16262 UDP on your firewall (`ufw`) but you still need to forward/allow them in the Oracle Cloud console (VCN Security List / Network Security Group) for your VM instance - `ufw` alone won't help if Oracle's infrastructure-level firewall is blocking it first.

## Setup guide step by step

### 1. Clone the Repository (on both machines)

```bash
git clone https://github.com/juanbravozu/zomboid-server-on-arm.git
cd zomboid-server-on-arm
```

### 2. On your x86_64 machine: download the server files

```bash
chmod +x 01-download-server-x86.sh
./01-download-server-x86.sh
```

- Installs SteamCMD and its dependencies natively (no emulation).
- Asks whether you want the current release or a beta branch/version.
- Downloads the server files into `~/zomboid-server-files`.
- At the end, prints the exact `rsync` command you need for step 3.

### 3. Transfer the files to your ARM server

Still on the x86_64 machine, run the `rsync` command the script printed, e.g.:

```bash
rsync -avz --progress ~/zomboid-server-files/ your-user@your-arm-host:/opt/zomboid-server/
```

This may take a while depending on your connection - the server files are several GB.

### 4. On your ARM server: build Box64 and set up the service

```bash
chmod +x 02-setup-server-arm.sh
./02-setup-server-arm.sh
```

This script is safe to run more than once - it's the same script for both "before" and "after" the file transfer:
- **First run** (before you've transferred files): installs dependencies, builds Box64, opens the firewall ports, creates `/opt/zomboid-server`, then notices the server files aren't there yet, prints the `rsync` command again for convenience, and exits cleanly. You can run this before or after step 3 - order between steps 3 and 4's first run doesn't matter.
- **Run it again** once the transfer from step 3 is done: detects the files are present and continues.

### 5. When the script prompts you to, on another terminal (don't close the first one), run the server for the first time and set a password. When finished, shut it down and check that you correctly killed the process.

The server must run as `zomboiduser` (the dedicated service account), not your own login user — and since `zomboiduser` has no login shell, use `sudo -u`, not `sudo -iu`:

```bash
sudo -u zomboiduser bash -c '
cd /opt/zomboid-server
box64 jre64/bin/java -Djava.awt.headless=true -Xms<SEE_BELOW>g -Xmx<SEE_BELOW>g \
  -XX:ActiveProcessorCount=<YOUR_OCPU_COUNT> \
  -Dzomboid.steam=1 -Djava.library.path=linux64/:natives/ \
  -cp "java/:java/projectzomboid.jar" zombie.network.GameServer
'
```

**Don't copy the heap/CPU flags as-is** — size them to your actual instance. Rough starting point: `-Xmx` around 50–60% of total RAM (leaving room for the OS, Box64's own overhead, and some swap headroom), `-Xms` at half of `-Xmx` or less, `-XX:ActiveProcessorCount` set to your instance's OCPU count. Example for a 1 OCPU / 6GB instance: `-Xms1g -Xmx3g -XX:ActiveProcessorCount=1`.

Whatever values you use here, use the *same* values in the systemd service's `ExecStart` later — this manual run and the long-running service should be configured identically.

### 6. Go back to the first terminal and finish running the script.

### 7. Done! The server should automatically start as a systemd service. You can validate and check the logs with:

```bash
systemctl status zomboid-server
journalctl -xeu zomboid-server --follow
```

## Updating the server later

When a new Project Zomboid update comes out:

1. Re-run `01-download-server-x86.sh` on the x86_64 machine (it re-runs SteamCMD's update check and pulls only what's changed).
2. Re-run the `rsync` command to sync the changes to the ARM server.
3. On the ARM server: `sudo systemctl restart zomboid-server`.

## Troubleshooting

- **Apt lock errors** (`Could not get lock /var/lib/apt/lists/lock`): both scripts wait automatically for `unattended-upgrades` to finish before running `apt`/`dpkg` commands. If it's still stuck after 5 minutes, something else is genuinely wrong - check `ps aux | grep apt` manually.
- **Firewall**: only UDP ports `16261` and `16262` are opened by the ARM script via `ufw`. On Oracle Cloud, also check your VCN Security List / Network Security Group in the console.
- **Box64 tuning**: the systemd service sets `BOX64_DYNAREC_BIGBLOCK=0` and `BOX64_DYNAREC_STRONGMEM=1`, commonly recommended for JVM stability under Box64. If you see crashes, this is a reasonable first place to experiment - see the [Box64 documentation](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md) for the full list of tunable environment variables.
