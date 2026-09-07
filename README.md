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
Stopping the instance sends Ctrl+C, which saves the world first (allow up to
~30s; the template's exit timeout is 60s).

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

It's not in the AMP settings UI because it's JSON *with comments* — AMP's
config mapper would strip them and lose all the inline documentation. Edit it
with AMP's File Manager instead, and restart the server to apply (difficulty
changes need a restart). Delete the file and restart to regenerate it with
any new options after a mod update.

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

## Not yet tuned

These are deliberately left conservative rather than guessed at:

- **`App.MonitorChildProcessName` is empty.** The launcher spawns the actual
  engine as a child process, so AMP's CPU/RAM figures will reflect only the
  launcher. Once it's running, `ps aux` inside the container will show the
  engine's command line and we can write a regex for it. A *wrong* regex
  makes AMP think the server crashed, so it's better empty than guessed.
- **User join/leave/chat regexes are empty.** The mod logs through Serilog
  templates whose rendered format I couldn't confirm from the binary. Send me
  a console excerpt of someone joining and I'll add them so AMP's player list
  populates.
- **`--data-dir` uses the Wine `Z:` drive** (`Z:` maps to `/`, standard in
  every Wine prefix) to point at the instance's `CoopData/DedicatedServer`.
  If the data path in the server's boot banner ("data :" / "conf :") doesn't
  match, that's the line to check.
- **Passwords containing single quotes** will break the update script's Steam
  login. Everything else is fine, and clearing the password after first login
  sidesteps it entirely.
