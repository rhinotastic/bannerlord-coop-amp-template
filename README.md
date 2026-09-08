# Bannerlord Coop — AMP Template

A CubeCoders AMP "Generic Module" configuration that runs the Bannerlord Coop
dedicated server under Wine in AMP's official Wine container
(`cubecoders/ampbase:wine-stable`), installs and updates it straight from the
Steam Workshop with SteamCMD, and exposes the server settings in AMP's UI.

Built from CubeCoders' own Wine templates (Subsistence, Longvinter-Wine) and
the official Mount & Blade II: Bannerlord template, plus the real settings
schema and CLI flags read out of `BannerlordCoopServer.exe` itself.

## What it does

- **Installs/updates from the Workshop.** Clicking *Update* in AMP runs
  SteamCMD with `+workshop_download_item 261550 <item>` — the same command as
  the manual guide, so updating the mod is one button.
- **Runs it under Wine.** No Wine install on the host; the container has it.
- **Config in the AMP UI.** Save name, password, autosave interval, ports,
  Steam advertising and the diagnostic toggles are real AMP settings mapped
  into `server-config.json`.
- **Keeps your world safe across updates.** The server is launched with
  `--data-dir` pointing at `CoopData/DedicatedServer` inside the instance, so
  saves and config live *outside* the Workshop folder that SteamCMD
  overwrites. (Without this, Wine would bury them in
  `.wine/drive_c/users/…/Documents/…`, which the Coop FAQ warns about.)

## Files

| File | Purpose |
|---|---|
| `bannerlord-coop.kvp` | The Generic Module config |
| `bannerlord-coopports.json` | UDP 4200 + 4201 (players), 7210 (engine, internal) |
| `bannerlord-coopupdates.json` | The SteamCMD download/update stage |
| `bannerlord-coopconfig.json` | The AMP settings UI |
| `bannerlord-coopmetaconfig.json` | AutoMaps those settings into `server-config.json` |
| `manifest.json` | Template-repo descriptor for AMP |

## Setup

### 1. Install the template into AMP

Push this folder to a GitHub repo, then in AMP:

1. Edit `manifest.json` — set `origin`/`url` to your repo (and a fresh `id`
   UUID if you rename things).
2. `Configuration → Instance Deployment → Add Configuration Repository`
3. Enter `yourusername/yourrepo:main`, click **Fetch**, refresh the browser.
4. "Bannerlord Coop Dedicated Server" now appears when creating an instance.

No-GitHub alternative: copy these files into a folder named
`LOCALbannerlordcoop-main` under your ADS instance's
`Plugins/ADSModule/DeploymentTemplates/` directory, then restart ADS.

### 2. Create the instance and set Steam credentials

Before the first update, open the instance's **Configuration → Bannerlord
Coop → Steam Workshop Update** and set:

- **Steam Username** — an account that **owns Bannerlord** (Workshop content
  for a paid game can't be fetched anonymously)
- **Steam Password** — only needed for the first login
- **Steam Guard Code** — current 2FA code, if the account has it. Codes
  expire fast, so paste it immediately before clicking Update.

Then click **Update**. It pulls ~6 GB (the Workshop item bundles the whole
dedicated server plus game assets), so give it time.

**After the first successful update, clear the password and Guard code —
leave only the username.** SteamCMD caches its own login token under
`bannerlordcoop/steamhome/Steam/config/`, so every later update logs in with
the username alone and never asks for a Guard code again. The Guard code is
a one-time cost, which is why it isn't worth wiring into a prompt.

### Why not AMP's built-in Steam login popup?

AMP does have a native credential prompt (`SteamUpdateAnonymousLogin=False`
+ `SteamForceLoginPrompt=True`), and it's nicer — AMP passes what you type
straight to SteamCMD without storing it. But that prompt only fires for
AMP's built-in `SteamCMD` update stage, which can only run `app_update` on a
Steam **application**. The Coop server isn't an app; it's a Workshop item,
and `workshop_download_item` isn't reachable from that stage — which is why
every official template that installs Workshop content (ARK, Arma 3, DayZ)
shells out to its own script exactly like this one does.

Using the native prompt would mean adding a pointless multi-GB app download
purely to trigger a login. Since the credentials are needed only once, the
settings fields are the better trade.

### Where the Workshop files land

`workshop_download_item` ignores `+force_install_dir` and always downloads to
`$HOME/Steam/steamapps/workshop/`. This template points `HOME` at
`bannerlordcoop/steamhome`, inside the instance, so the ~6 GB download
persists across container restarts. (AMP's own ARK template instead symlinks
`~/Steam/steamapps`, which leaves the content in container-local storage.)

### 3. Configure and start

Server settings are under **Configuration → Bannerlord Coop**. The defaults
match the mod's own defaults, with one change: **Advertise on Steam is off**.
Steam lobby hosting needs a logged-in Steam *client* on the machine, which a
headless container doesn't have — the Coop FAQ says VPS hosts should use
direct connect instead.

So players connect by **Direct IP** to your server's address on **UDP 4200**
(and 4201 — the server uses both). Both are declared in the template, so AMP
reserves them; open them on your host firewall/router.

Then hit Start. AMP marks the instance *Running* when the console prints
`[DedicatedServer] SERVING`. Campaign load takes ~30–60s and prints a lot of
engine noise on the way — that's normal.

The AMP console is a real server console: `save` writes a manual save, and
the Coop debug commands (`coop.debug.players.list`, etc.) work there too.

**Stopping** sends the server's own `stop` console command on stdin
(`ExitMethod=String`), which saves the world and shuts the engine down
cleanly. Allow up to ~30s for the save; the exit timeout is 90s.

Do *not* use `ExitMethod=OS_CLOSE` here, even though the server's own docs
say "press Ctrl+C to stop" and AMP's other Wine templates use it. AMP would
send SIGINT to the Wine process, which must then be translated into a Win32
`CTRL_C_EVENT` and passed through `start.exe` to the launcher — that chain
doesn't survive, and the instance hangs on Stop until the timeout expires.
The console command needs no signal translation.

## Gameplay settings (mod-config.json)

Difficulty, battle size, wanderer limits and so on live in a **separate**
file. Unlike `server-config.json`, its location is *not* affected by
`--data-dir` — the server always resolves it against the Windows "Documents"
folder, which under Wine means inside the prefix:

```
bannerlordcoop/.wine/drive_c/users/amp/Documents/Mount and Blade II Bannerlord/CoopData/mod-config.json
```

(Confirmed from the boot log: `[Coop] mod-config.json loaded
("C:\users\amp\Documents\...\CoopData\mod-config.json")`.) It still lives in
the instance directory, so it persists across restarts.

All 32 of its settings **are** exposed in AMP, under the **Gameplay** tab
(Campaign Difficulty / Mod Options). Changes need a server restart to apply.

Because AMP can only map config files at or below the base directory, the
update stage moves the file into `CoopData/DedicatedServer/` and leaves a
symlink at the Wine path so the server still finds it. The symlink points
*into* the data folder rather than the other way round, so AMP replacing the
file on save can't break the link. Running Update again re-establishes it if
anything ever does.

If the file doesn't exist yet, Update says so — start the server once to
generate it, then run Update again.

**Caveat:** the file the server generates is JSON *with comments and a
trailing comma*. AMP's mapper reads it, but the first time AMP writes to it
those comments are dropped. The settings' documentation lives in the AMP UI
descriptions instead, and the fully-commented original can always be
regenerated by deleting the file and restarting the server. If AMP fails to
read it at all, delete `mod-config.json`, restart to regenerate, and run
Update.

## Things worth knowing

- **Sandbox only.** The mod doesn't support the main story campaign.
- **Up to 8 players.** More works but performance degrades. The mod is
  single-thread-heavy, so single-core speed matters most.
- **Game version pinned.** This build targets Bannerlord v1.4.8, and clients
  must run the exact matching Coop build. **War Sails must not be enabled.**
- **The server has no character or party** — that's expected; join it with a
  game client.
- **Workshop updates aren't nightly.** The Workshop item updates roughly
  weekly, so clicking Update won't always change anything.

## Verified on a real run

Confirmed against a live server boot (Coop build for Bannerlord v1.4.8):

- Wine launch, `WorkingDir` and the `Z:`-drive `--data-dir` all resolve
  correctly — the boot banner reports
  `data : Z:\AMP\bannerlordcoop\CoopData\DedicatedServer\`
- Ports and region pass through:
  `port : UDP 4200 — coop clients connect here (engine custom server: 7210/EU)`
- Campaign loads in ~42s and reaches
  `[DedicatedServer] SERVING — coop server up, waiting for clients`,
  which `AppReadyRegex` matches (with or without the log timestamp prefix)
- `MetricsRegex` graphs players / parties / map events from the `pulse:`
  line the server emits every ~14s
- `ThrowawayMessageRegex` hides the `@DS@{...}` machine-readable event lines,
  including the enormous one-line command dump printed at startup
- `MonitorChildProcessName` tracks the real workload. The launcher
  (`BannerlordCoopServer.exe`, ~0.3% CPU) spawns the engine
  (`dotnet.exe TaleWorlds.Starter.DotNetCore.dll …`, ~150% CPU / 800 MB),
  and the regex is keyed to `/dedicatedcustomserver {{$EnginePort}}` so it
  stays correct with several instances on one host
- Join/leave detection, matched on the peer id

## Not yet tuned

These are deliberately left conservative rather than guessed at:

- **Watchdog.** The engine starts `Watchdog.exe`, Bannerlord's crash
  collector, which holds the process's single debugger slot. The Coop
  community reports this causes severe lag under Wine on Linux, and works
  around it by renaming the executable. This template instead passes the
  launcher's own `--no-watchdog` flag, controlled by the **Disable Crash
  Watchdog** setting (on by default).

  Prefer the flag over renaming: the update stage runs
  `workshop_download_item … validate`, which restores any renamed or deleted
  file, so a rename would have to be redone after every update.

  The cost is no crash dumps. If you need one for a bug report, turn the
  setting off temporarily.

  Confirmed working: with the setting on, the launcher forwards the engine
  token and no Watchdog process is started —
  `/dedicatedcustomserver 7211 EU 0 no_watchdog`.

  **Linux clients need this too.** Windows players were smooth while Linux
  players stuttered on the campaign map on the same server at the same time,
  and deleting the client's own Watchdog folder fixed it:

  ```
  Mount & Blade II Bannerlord/bin/Win64_Shipping_Client/Watchdog/
  ```

  There is no client-side equivalent of `--no-watchdog` — the flag belongs to
  the Coop dedicated-server launcher, which clients don't run — so deletion is
  the only lever. Steam's "verify integrity of game files" and some patches
  restore it, so it has to be redone afterwards.

  Note this is invisible in server CPU: the engine sat at ~150% both with and
  without Watchdog. The cost is frame-time jitter, not throughput.
- **AMP's player list shows IP addresses, not character names.** Join/leave
  detection works (matched on the peer id, like AMP's own V Rising template),
  but the only identity the server prints at connect time is the IP:
  `player connecting: peer 0 from 82.10.106.223`. The character name doesn't
  exist yet at that point — players pick it during character creation — and
  when it does appear it's inside a `@DS@{"ev":"players","list":[...]}` event.
  AMP's console parser handles per-user event lines, not list snapshots, so
  the name can't be recovered from it.
- **Chat is not surfaced.** No server-side chat line was observed in the
  logs, so `UserChatRegex` is left empty.
- **`--data-dir` uses the Wine `Z:` drive** (`Z:` maps to `/`, standard in
  every Wine prefix) to point at the instance's `CoopData/DedicatedServer`.
  If the data path in the server's boot banner ("data :" / "conf :") doesn't
  match, that's the line to check.
- **Passwords containing single quotes** will break the update script's Steam
  login. Everything else is fine, and clearing the password after first login
  sidesteps it entirely.
