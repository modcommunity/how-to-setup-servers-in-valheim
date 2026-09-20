In this guide we will be **downloading**, **configuring** and **running** a [Valheim](https://www.valheimgame.com/) dedicated server on **Windows**, **Linux** and **macOS**.

A dedicated server gives you a persistent world that keeps running whether or not anybody is logged in, which is a real step up from the host-and-play setup most groups start with. Valheim's server is free, does not need a second copy of the game, and is genuinely one of the easier ones to get going.

We cover the vanilla setup here. If you want mods on top, our [Valheim mod guide](https://moddingcommunity.com/blog/how-to-install-mods-in-valheim/) covers the server side of BepInEx as well as the client.

[**View Guide On TMC (Recommended Due To Better Formatting)**](https://moddingcommunity.com/blog/how-to-setup-a-valheim-server/)

## Table Of Contents
* [Requirements](#requirements)
* [Steam Backend Or Crossplay Backend](#steam-backend-or-crossplay-backend)
* [Downloading The Server Files](#downloading-the-server-files)
    * [Through The Steam Client](#through-the-steam-client)
    * [Through SteamCMD](#through-steamcmd)
* [Linux Prerequisites](#linux-prerequisites)
    * [Debian And Ubuntu](#debian-and-ubuntu)
    * [Fedora, RHEL And Rocky](#fedora-rhel-and-rocky)
* [Launch Arguments](#launch-arguments)
    * [World Modifiers](#world-modifiers)
* [Starting The Server On Windows](#starting-the-server-on-windows)
* [Starting The Server On Linux](#starting-the-server-on-linux)
    * [Running It With Screen](#running-it-with-screen)
    * [Running It As A systemd Service](#running-it-as-a-systemd-service)
* [Running Via Docker](#running-via-docker)
* [macOS](#macos)
* [Port Forwarding](#port-forwarding)
* [Admins, Bans And The Permitted List](#admins-bans-and-the-permitted-list)
* [Saves And Backups](#saves-and-backups)
* [Stopping The Server Properly](#stopping-the-server-properly)
* [Troubleshooting](#troubleshooting)
* [Conclusion](#conclusion)
* [See Also](#see-also)

## Requirements
* A machine running **Windows 10** or later, or a modern **Linux** distribution. See the [macOS](#macos) section for why Macs are a special case.
* At least **4 GB** of RAM for a small group. Valheim servers grow their memory use as the world gets explored and built on, so 8 GB is a safer target for a group of ten.
* Around **2 - 3 GB** of disk space for the server files, plus room for world saves and backups.
* A **Steam account**. It does not need to own Valheim, since the dedicated server is a free tool.
* Basic comfort with a terminal and with editing a text file.

**TIP** - As with any game server, run it under a separate user account rather than as your main user or as root. It costs nothing and limits the damage if anything goes wrong.

## Steam Backend Or Crossplay Backend
Decide this before you start, because it changes whether you need to touch your router.

**Steam backend** is the default. The server talks directly to clients, which means you need to port forward, and only Steam players can see or join.

**Crossplay backend**, enabled with `-crossplay`, routes traffic through a PlayFab relay. You do **not** need port forwarding, and players from any platform, including Xbox and the Microsoft Store version, can join.

Crossplay sounds like the obvious win and often is, particularly if you cannot control your router. The trade-offs are that traffic goes through a relay rather than direct, which can add latency, and you cannot connect to a crossplay server over a local or loopback IP. You connect by public IP and port, by join code, or through the server list.

**NOTE** - If you are running mods, bear in mind that Xbox and Microsoft Store clients cannot load BepInEx. Enabling crossplay on a modded server lets those players find it but not necessarily play on it.

## Downloading The Server Files
The Valheim dedicated server is Steam app **896660**, and it is free.

### Through The Steam Client
The simplest route if the machine has Steam on it with a desktop.

1. Open your Steam **Library**.
2. Click the dropdown at the top left and make sure the **Tools** checkbox is ticked.
3. Find **Valheim Dedicated Server** in the list and click **Install**.
4. Once it finishes, right-click it and choose **Manage**, then **Browse local files** to open the install folder.

### Through SteamCMD
The right approach for a headless box, and the one to use on a Linux server.

We have a separate guide covering SteamCMD itself:

https://moddingcommunity.com/blog/how-to-download-run-steamcmd/

With SteamCMD installed, the commands are:

```
login anonymous
force_install_dir /path/to/valheim-server
app_update 896660 validate
quit
```

Or as a one-liner you can drop into a script or a cron job:

```bash
./steamcmd.sh +login anonymous +force_install_dir /home/valheim/server +app_update 896660 validate +quit
```

Anonymous login works here, so no credentials are needed. Re-run the same command any time you want to update the server, which you will need to do after every Valheim patch, since clients refuse to connect to a server on an older build.

## Linux Prerequisites
The Linux server shares most of its library requirements with the Steam client, plus a few extras.

### Debian And Ubuntu
```bash
sudo apt update
sudo apt install -y libatomic1 libpulse-dev libpulse0 screen
```

### Fedora, RHEL And Rocky
```bash
sudo dnf install -y libatomic pulseaudio-libs pulseaudio-libs-devel screen
```

The server also needs **GLIBC 2.29** and **GLIBCXX 3.4.26** or newer. Any current distribution has these. If you are on something older, such as CentOS 7, the supported answer is [Docker](#running-via-docker) rather than trying to upgrade glibc by hand.

## Launch Arguments
All of the server's configuration is done through launch arguments rather than a config file. Here are the ones that matter.

| Argument | Default | Description |
| -------- | ------- | ----------- |
| `-name "My server"` | *N/A* | The server name shown in the server list. |
| `-port 2456` | `2456` | The UDP port. Valheim uses this port **and** the one above it, so 2456 means 2456 and 2457. |
| `-world "Dedicated"` | *N/A* | The world to load. Creates it if it does not exist, or loads an existing world of that name. |
| `-password "Secret"` | *N/A* | The join password. |
| `-public 1` | `1` | `1` lists the server publicly. `0` hides it, so players join by IP. Good for LAN. |
| `-crossplay` | off | Use the PlayFab crossplay backend instead of Steam. No port forwarding needed. |
| `-savedir [PATH]` | see below | Overrides where worlds and permission files are stored. |
| `-logFile "/path/log.txt"` | *N/A* | Where to write the log. |
| `-saveinterval 1800` | `1800` | How often the world saves, in seconds. Default is 30 minutes. |
| `-backups 4` | `4` | How many automatic backups to keep. |
| `-backupshort 7200` | `7200` | Interval before the first automatic backup, in seconds. |
| `-backuplong 43200` | `43200` | Interval between subsequent backups, in seconds. |
| `-instanceid "1"` | *N/A* | Needed only when running several servers on the same port from the same MAC address, so each gets a unique PlayFab ID. |

Default save paths, if you do not override them:

* **Windows**: `%USERPROFILE%\AppData\LocalLow\IronGate\Valheim`
* **Linux**: `~/.config/unity3d/IronGate/Valheim`

**WARNING** - Your password cannot be blank and cannot be a substring of the server name or the world name. The server refuses to start if it is, and the error message is easy to miss in the log.

### World Modifiers
You can set the world's difficulty and rules from the command line too, which is handy for a server since nobody has to agree on the in-game sliders.

`-preset` sets everything at once. Valid values are `Normal`, `Casual`, `Easy`, `Hard`, `Hardcore`, `Immersive` and `Hammer`.

`-modifier <name> <value>` sets one at a time, and should come **after** a preset if you use both, since a preset overwrites anything before it:

| Modifier | Values |
| -------- | ------ |
| `combat` | veryeasy, easy, hard, veryhard |
| `deathpenalty` | casual, veryeasy, easy, hard, hardcore |
| `resources` | muchless, less, more, muchmore, most |
| `raids` | none, muchless, less, more, muchmore |
| `portals` | casual, hard, veryhard |

`-setkey <name>` toggles a checkbox key: `nobuildcost`, `playerevents`, `passivemobs` or `nomap`.

For example, a no-raid, generous-resources server:

```
-preset normal -modifier raids none -modifier resources more
```

## Starting The Server On Windows
The server ships with `start_headless_server.bat` in its install folder.

**Make a copy of it and edit the copy.** Steam resets the original every time the server updates, which will wipe your settings. The one catch is that the Steam library shortcut only points at the original filename, so you launch your copy directly rather than through Steam.

1. Open the server's install folder.
2. Copy `start_headless_server.bat` to something like `start_myserver.bat`.
3. Right-click your copy and choose **Edit**.
4. Find the line starting with `start valheim_server` and set your arguments on it.

A finished line looks like this:

```batch
start valheim_server -nographics -batchmode -name "My Valheim Server" -port 2456 -world "Dedicated" -password "secret123" -public 1 -crossplay
```

5. Save, then double-click the batch file.
6. If Windows Firewall prompts you, tick every box so the server can talk to the internet.

You are up when the console prints:

```
Game server connected
```

## Starting The Server On Linux
The Linux equivalent is `start_server.sh`. Copy it first, for the same reason as on Windows.

```bash
cd /home/valheim/server
cp start_server.sh start_myserver.sh
chmod u+x start_myserver.sh
nano start_myserver.sh
```

Set your arguments on the `./valheim_server.x86_64` line:

```bash
#!/bin/bash
export templdpath=$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=./linux64:$LD_LIBRARY_PATH
export SteamAppId=892970

./valheim_server.x86_64 \
    -nographics \
    -batchmode \
    -name "My Valheim Server" \
    -port 2456 \
    -world "Dedicated" \
    -password "secret123" \
    -public 1 \
    -crossplay

export LD_LIBRARY_PATH=$templdpath
```

Note the `SteamAppId=892970` line, which is the **game's** app ID rather than the server's. It is already in the stock script and it needs to stay.

Run it with:

```bash
./start_myserver.sh
```

### Running It With Screen
Run the server in the foreground of an SSH session and it dies the moment you disconnect. `screen` fixes that.

```bash
screen -S valheim ./start_myserver.sh
```

Detach with `CTRL` + `A` then `D`. Reattach with:

```bash
screen -r valheim
```

List your sessions with `screen -ls`.

### Running It As A systemd Service
For anything you actually care about, a systemd unit is better than a screen session. It starts on boot and restarts on crash.

Create `/etc/systemd/system/valheim.service`:

```ini
[Unit]
Description=Valheim Dedicated Server
After=network.target

[Service]
Type=simple
User=valheim
WorkingDirectory=/home/valheim/server
ExecStart=/home/valheim/server/start_myserver.sh
Restart=on-failure
RestartSec=10
KillSignal=SIGINT
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now valheim
```

`KillSignal=SIGINT` matters. Valheim needs the equivalent of `CTRL` + `C` to shut down cleanly and save the world, and `TimeoutStopSec=120` gives it time to finish. Getting this wrong is a good way to lose progress.

Check on it with `systemctl status valheim` and follow the log with `journalctl -u valheim -f`.

## Running Via Docker
The dedicated server needs GLIBC 2.29 and GLIBCXX 3.4.26. If your distribution is older than that, Docker is the officially suggested route, and the server ships a script for it.

1. Install Docker using the [instructions for your distribution](https://docs.docker.com/engine/install/).
2. Make sure the daemon is running and your user is in the `docker` group. This should work without error:

```bash
docker ps
```

3. Run the bundled script, passing it your startup script:

```bash
./docker_start_server.sh start_myserver.sh
```

The first run bootstraps the container and takes a few minutes. Later runs start straight away.

Game data goes into a Docker volume named in `DOCKER_DATA_VOLUME` inside `docker_start_server.sh`, defaulting to `valheim_server_data`. To keep the data in your filesystem instead, change it to a path:

```bash
DOCKER_DATA_VOLUME="${HOME}/valheim_storage"
```

Create that directory before running the script. Your worlds, `adminlist.txt`, `bannedlist.txt` and `permittedlist.txt` all end up there. Stop the server the same way as when running it directly.

## macOS
Valheim has a native macOS **client**, but there is no macOS build of the dedicated server. Steam ships it for Windows and Linux only.

If you want to host from a Mac, the realistic options are:

* Run Linux in a VM and follow the Linux instructions above.
* Use Docker Desktop for Mac with a community Valheim server image.
* Host the server on a Linux box or a rented VPS and just play from the Mac.

Mac players can of course join a server hosted anywhere, and modding the Mac client works with the Rosetta workaround covered in our mod guide.

## Port Forwarding
Skip this section entirely if you are using `-crossplay`.

On the Steam backend, forward these on your router to the machine running the server:

| Protocol | Port | Purpose |
| -------- | ---- | ------- |
| UDP | 2456 | Game port |
| UDP | 2457 | Game port + 1, used automatically |

If you changed `-port`, forward your chosen port and the one above it.

You may also need to allow them through the OS firewall:

```bash
# Debian/Ubuntu with ufw
sudo ufw allow 2456:2457/udp

# Fedora/RHEL with firewalld
sudo firewall-cmd --permanent --add-port=2456-2457/udp
sudo firewall-cmd --reload
```

On Windows, ticking every box on the firewall prompt at first launch handles it.

**TIP** - Running several servers on one machine works fine as long as each gets its own port pair. Use 2456/2457, 2458/2459 and so on, and give each an `-instanceid`.

## Admins, Bans And The Permitted List
Three text files in the save directory control access, one Platform User ID per line:

* `adminlist.txt` - who gets admin privileges.
* `bannedlist.txt` - who is banned.
* `permittedlist.txt` - an allow list.

**WARNING** - Putting anybody on `permittedlist.txt` bans everyone not on it. That is the intended behaviour, and it surprises people. Leave it empty unless you specifically want a closed server.

Platform User IDs look like `[Platform]_[User ID]` and are case sensitive. You can read them out of the server log, or in-game from the **F2** panel.

In-game, admins open the console with **F5** and have:

| Command | Effect |
| ------- | ------ |
| `kick PLAYERNAME` | Kicks a player. |
| `ban PLAYERNAME` | Bans a player. |
| `unban PLAYERNAME` | Lifts a ban. |
| `banned` | Lists everyone banned. |

## Saves And Backups
The server saves the world every `-saveinterval` seconds, 30 minutes by default, and keeps `-backups` automatic backups.

Out of the box that gives you one backup around two hours old and three more twelve hours apart. For an active server that is thin, and something like this is a better starting point:

```
-saveinterval 900 -backups 8 -backupshort 3600 -backuplong 21600
```

That saves every 15 minutes and keeps eight backups, the first an hour old and the rest six hours apart.

The automatic backups live next to your world files in the save directory. They are on the same disk as the thing they are backing up, so copy them somewhere else on a schedule if the world matters to you. A cron job with `rsync` is plenty.

## Stopping The Server Properly
Press `CTRL` + `C` in the server console.

Do not close the window with the X. Iron Gate's own documentation is refreshingly candid on this point: the server may keep running in the background and they do not really know. Either way, you risk losing anything since the last save.

Under systemd, `systemctl stop valheim` sends SIGINT if you set `KillSignal=SIGINT` as above, which is the same thing.

## Troubleshooting
**The server starts and then immediately exits.** Nine times out of ten this is the password rule. It cannot be blank, and it cannot appear inside the server name or the world name.

**Nobody can find the server in the list.** Public listings can take several minutes to show up, and they are unreliable at the best of times. Have someone join by IP with **Join IP** to confirm the server itself is reachable, then worry about the listing.

**Players get a version mismatch.** The server is on a different build from the clients. Re-run the SteamCMD update, or update through the Steam client.

**Connection refused or timeout on the Steam backend.** Port forwarding. Both UDP ports, forwarded to the right internal IP, and allowed through the OS firewall.

**Crossplay server is unreachable on my LAN.** Expected. Crossplay servers cannot be reached over a local or loopback IP. Use the public IP, a join code, or the server list.

**Missing library errors on Linux.** Install `libatomic1`, `libpulse0` and `libpulse-dev`. If the errors mention GLIBC or GLIBCXX versions, your distribution is too old and Docker is the answer.

**Memory use keeps climbing.** Normal as the world is explored and built on. Restarting the server on a schedule is a common and perfectly reasonable workaround.

**The world reverted after a crash.** You lost everything since the last save. Shorten `-saveinterval`, and make sure you are shutting down with `CTRL` + `C`.

## Conclusion
A Valheim server is one of the friendlier ones to set up. Pull app **896660** with SteamCMD, copy the start script, put your name, port, world and password on the command line, and run it.

The two decisions worth thinking about up front are crossplay against the Steam backend, which determines whether you touch your router at all, and your backup settings, since the defaults are thinner than most groups want.

If you plan on adding mods later, our [Valheim mod guide](https://moddingcommunity.com/blog/how-to-install-mods-in-valheim/) covers `start_server_bepinex.sh` and which mods need to be on the server as well as the client.

## See Also
* [Official Valheim dedicated server guide](https://www.valheimgame.com/support/a-guide-to-dedicated-servers/) - Iron Gate's own documentation, and the source for the argument list above.
* [Valheim Crossplay FAQ](https://valheim.com/support/crossplay-faq)
* [SteamCMD documentation](https://developer.valvesoftware.com/wiki/SteamCMD)
* [Valheim Modding Discord](https://discord.gg/RBq2mzeu4z)
* [Valheim Modding Wiki](https://github.com/Valheim-Modding/Wiki/wiki)
* [TMC App](https://github.com/modcommunity/tmc-app) - Our own app, with a server browser, live latency graphs and RCON. It also manages Valheim mods, which our [Valheim mod guide](https://moddingcommunity.com/blog/how-to-install-mods-in-valheim/) covers. Open source under GPL-3.0 and in very early development, so trying it or leaving feedback is genuinely appreciated.

We keep this guide as current as we can, but Valheim and its server tooling change over time. If an instruction here no longer matches what you are seeing, please report it or open a [pull request](https://github.com/modcommunity/how-to-setup-servers-in-valheim/pulls) on this guide's GitHub repository.

Join our [Discord server](https://discord.moddingcommunity.com) if you have any questions or want help with anything server or modding related!
