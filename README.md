# lutris-discord-rpc-flatpak

# Lutris → Discord Rich Presence (native Lutris + Flatpak Discord)

Show **"Playing Lutris"** with the **game name** underneath in Discord, for any game or emulator in Lutris, including games Lutris has no Discord app ID for.

Lutris's built-in Discord option only works for games that have a Discord application ID, so manually added games and emulators show nothing. This setup replaces it with a small script that Lutris runs on game launch and exit.

> **Note:** This project was created 100% with [Claude](https://claude.ai) (Anthropic's AI assistant). I don't know much Python or shell, so I can't vouch for the code beyond the fact that it works on my setup (below). Read the scripts before running them, and use them at your own risk.

## Tested on

| Component | Version / setup |
|---|---|
| OS | CachyOS (KDE Plasma, Wayland) |
| Lutris | 0.5.22, **native** package |
| Discord | **Flatpak** (`com.discordapp.Discord`), patched with Vencord |
| Python | 3.14 (the script uses only the standard library, no `pypresence`) |

Other setups may work but are untested. If Lutris is also a Flatpak, the socket access differs and this guide doesn't cover it.

## What you get, and what you don't

- Shows while a game runs, clears when it exits.
- Title is the name of your Discord app (for example "Lutris"). The game name is shown as a second line.
- It is live status only. There is no play history or hours-played record.

## How it works

1. Lutris runs a **pre-launch script** that starts `lutris-rpc.py` with the game name (`$GAME_NAME`).
2. The script talks to Discord's IPC socket directly and sets the activity.
3. Lutris runs a **post-exit script** that stops the script, which clears the presence.
4. A small **Discord launcher wrapper** removes a stale socket file before Discord starts (see [Root cause](#root-cause-of-the-it-works-once-problem)).

## Setup

### 1. Create the Discord application

1. Open <https://discord.com/developers/applications> and click **New Application**.
2. Name it what you want shown after "Playing" (for example `Lutris`).
3. Copy the **Application ID** from *General Information*.

### 2. Install the presence script

Save as `~/.local/bin/lutris-rpc.py` (create the folder with `mkdir -p ~/.local/bin`):

```python
#!/usr/bin/env python3
import glob
import json
import os
import socket
import struct
import sys
import time
import uuid

APP_ID = "YOUR_APP_ID"
RUNTIME = os.environ.get("XDG_RUNTIME_DIR", "/run/user/1000")


def log(*args):
    print(*args, file=sys.stderr, flush=True)


def candidates():
    yield f"{RUNTIME}/app/com.discordapp.Discord/discord-ipc-0"
    yield f"{RUNTIME}/discord-ipc-0"
    seen = set()
    for root in glob.glob("/proc/[0-9]*/root"):
        path = f"{root}{RUNTIME}/discord-ipc-0"
        try:
            st = os.stat(path)
        except OSError:
            continue
        key = (st.st_dev, st.st_ino)
        if key not in seen:
            seen.add(key)
            yield path


def send(sock, op, payload):
    data = json.dumps(payload).encode()
    sock.sendall(struct.pack("<II", op, len(data)) + data)


def recv(sock):
    op, length = struct.unpack("<II", sock.recv(8))
    return op, json.loads(sock.recv(length))


def open_ipc():
    for path in candidates():
        sock = socket.socket(socket.AF_UNIX)
        sock.settimeout(3)
        try:
            sock.connect(path)
            send(sock, 0, {"v": 1, "client_id": APP_ID})
            reply = recv(sock)
            log("OK", path, reply)
            return sock
        except Exception as e:
            log("FAIL", path, type(e).__name__, e)
            sock.close()
    return None


sock = None
for _ in range(10):
    sock = open_ipc()
    if sock:
        break
    time.sleep(2)
if not sock:
    sys.exit("no Discord socket answered the handshake")

send(sock, 1, {
    "cmd": "SET_ACTIVITY",
    "args": {
        "pid": os.getpid(),
        "activity": {
            "details": sys.argv[1],
            "timestamps": {"start": int(time.time())},
        },
    },
    "nonce": str(uuid.uuid4()),
})
log("activity:", recv(sock))

while True:
    time.sleep(60)
```

Replace `YOUR_APP_ID` with your Application ID, then test it with Discord open:

```sh
chmod +x ~/.local/bin/lutris-rpc.py
python3 ~/.local/bin/lutris-rpc.py "Test Game"
```

You should see an `OK ...` line, an `activity: ...` line, and the presence in Discord. Stop it with Ctrl+C.

### 3. Add the Lutris hook scripts

Lutris's script fields are file pickers, so they need **a path to an executable file**, not an inline command.

`~/.local/bin/lutris-rpc-start.sh`:

```sh
#!/bin/sh
pkill -f lutris-rpc.py 2>/dev/null
python3 "$HOME/.local/bin/lutris-rpc.py" "$GAME_NAME" >/dev/null 2>&1 &
```

`~/.local/bin/lutris-rpc-stop.sh`:

```sh
#!/bin/sh
pkill -f lutris-rpc.py 2>/dev/null
```

```sh
chmod +x ~/.local/bin/lutris-rpc-start.sh ~/.local/bin/lutris-rpc-stop.sh
```

In Lutris go to **Preferences → System options** (the global tab, so it applies to every game and emulator):

- **Pre-launch script:** `/home/YOUR_USER/.local/bin/lutris-rpc-start.sh`
- **Post-exit script:** `/home/YOUR_USER/.local/bin/lutris-rpc-stop.sh`
- **Wait for pre-launch script completion:** off (otherwise Lutris waits on the script and the game never starts)

Also turn **off** Lutris's built-in Discord Rich Presence option so the two don't conflict.

### 4. Add the Discord launcher wrapper

Flatpak Discord can leave a stale `discord-ipc-0` socket file behind after it quits, and the next start then fails to create its own socket. This wrapper removes the file, but only when no Discord process is running.

`~/.local/bin/discord-start.sh`:

```sh
#!/bin/sh
pgrep -f 'discord/app-' >/dev/null || rm -f "$XDG_RUNTIME_DIR/discord-ipc-0"
exec flatpak run com.discordapp.Discord "$@"
```

Make it executable and point a user-level copy of Discord's launcher at it (a user-level `.desktop` file takes priority over the Flatpak one):

```sh
chmod +x ~/.local/bin/discord-start.sh
mkdir -p ~/.local/share/applications
cp /var/lib/flatpak/exports/share/applications/com.discordapp.Discord.desktop ~/.local/share/applications/
sed -i "s|^Exec=.*|Exec=$HOME/.local/bin/discord-start.sh %U|" ~/.local/share/applications/com.discordapp.Discord.desktop
kbuildsycoca6
```

- If Discord is installed per-user, the source file is `~/.local/share/flatpak/exports/share/applications/com.discordapp.Discord.desktop`.
- `kbuildsycoca6` refreshes the KDE launcher cache. On other desktops, use that desktop's equivalent.
- If Discord is pinned to a taskbar or panel, unpin and re-pin it so it uses the new launcher.
- Always quit Discord from its tray icon (right-click → Quit) and open it from the launcher.

## Verify

With Discord running, both of these listeners should exist:

```sh
ss -xl | grep discord-ipc
```

```
/run/user/1000/app/com.discordapp.Discord/discord-ipc-0   # bridge created by the Flatpak launcher
/run/user/1000/discord-ipc-0                              # Discord's own socket
```

Quit and reopen Discord two or three times and re-check, then launch a game from Lutris. The failure this guide fixes only appeared after restarts.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Only the `app/com.discordapp.Discord/discord-ipc-0` listener shows in `ss` | A stale `/run/user/1000/discord-ipc-0` file blocked Discord. Quit Discord fully, run `rm -f $XDG_RUNTIME_DIR/discord-ipc-0`, start Discord again. Make sure it's started through the wrapper (step 4). |
| Script prints `FAIL ... TimeoutError` or hangs at the handshake | Same cause: the bridge accepts connections but Discord isn't behind it. Check `ss` as above. |
| Script prints `FAIL ... BlockingIOError` | Many hung connections have piled up. Restart Discord and run the script once, not in a loop. |
| Handshake reply contains an error | Wrong Application ID. Use the ID from *General Information*, not a token or secret. |
| Game doesn't launch | "Wait for pre-launch script completion" is on. Turn it off. |
| Presence shows no game name | `GAME_NAME` isn't reaching the script. Put `env > /tmp/env.txt` in the start script once and check it. |
| Nothing at all, and no Discord output | Lutris's built-in option isn't what runs this, so check that the pre-launch script path is set and executable. |

## Root cause of the "it works once" problem

After Discord quit, a dead socket file was left at `/run/user/1000/discord-ipc-0`. On the next start Discord could not create its own socket there. The Flatpak launcher's bridge (`socat`) still listened at `/run/user/1000/app/com.discordapp.Discord/discord-ipc-0` and accepted connections, but had nothing to forward them to, so clients hung or were refused.

This was confirmed with `ss -xl | grep discord-ipc`: one listener (bridge only) with the stale file present, two listeners after removing it with Discord fully closed.

## Things not to do

- **Don't create symlinks** at `$XDG_RUNTIME_DIR/discord-ipc-0` (for example with `ln -sf`). They block Discord from creating its real socket there.
- **Don't delete `discord-ipc-*` files while Discord is running.** Only remove them with Discord fully quit, which is what the wrapper does.
- **Don't retry the script in a tight loop** against a broken Discord. Each hung attempt adds to the connection queue.

## Not covered

- Per-game titles in place of the app name (Discord takes the title from the app ID).
- Per-game icons (they need a hosted image URL for each game).
- Play history or hours tracking.
