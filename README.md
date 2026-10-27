# AYN Thor notes

Things I worked out setting up an AYN Thor (Snapdragon 8 Gen 2, Adreno 740, dual screen) in October 2026. Mainly: **Valheim online multiplayer**, which most guides stop short of, plus Discord voice on the bottom screen and a few emulation fixes.

Versions used: DroidDeck 0.5.0, Discord v351.7, Aurora Store 4.8.4, Cocoon 3.07, RetroArch 1.22.2, Eden Duo 1.3.2, Lime3DS 2123.3 (Azahar-branded).

---

## 1. Valheim online (multiplayer) via DroidDeck

### What doesn't work

- **GameNative**: single-player runs fine, but joining a server never connects. Its online modes all failed for me (Bionic Steam: "Failed to connect to Steam Service"; the GUI Steam client: black screen). The cause is that GameNative doesn't run a real Steam login, and Valheim's crossplay join needs one.
- **GameHub Lite**: logged into Steam and showed Valheim in the library, but the download sat at 0.00 B / "Computing" forever.
- **Full GameHub**: stopped at a GameSir account registration wall.

### What works: DroidDeck

[DroidDeck](https://github.com/Droid-Deck/DroidDeck) (open source, GPL-3) runs Valve's real Steam client in a Linux runtime of about 3 GB, so joining works like it does on a Steam Deck. It needs an Adreno 730 or newer; the Thor's 740 is fine. Check the APK against the `SHA256SUMS` in the release.

1. Install the APK.
2. Run DroidDeck setup. It downloads the runtime and asks for the Android "phantom process" limit to be lifted. Over adb:
   ```sh
   adb shell settings put global settings_enable_monitor_phantom_procs false
   adb shell device_config set_sync_disabled_for_tests persistent
   adb shell device_config put activity_manager max_phantom_processes 2147483647
   ```
   DroidDeck detects the change by itself.
3. Play Steam, then log in with the Steam app's QR code.
4. Install Valheim from your library (4.2 GB, about 3 minutes on my connection).
5. In Valheim: **Join Game**, then pick the server or enter its join code. Servers in your Steam favourites sync over via Steam Cloud.

Tested joining a self-hosted dedicated server running in crossplay mode (`-crossplay`, join by code, no ports forwarded). The join and character spawn both showed up in the server log.

**Keep Valheim's FEX preset on "Use Default".** The "Performance" preset (TSO off) breaks the crossplay join: "Failed to connect" straight after the handshake, twice in a row. Valheim is a Unity game, and DroidDeck has since fixed this upstream: [PR #1415](https://github.com/Droid-Deck/DroidDeck/pull/1415) (merged 23 Oct 2026) keeps TSO on for Unity games whatever the preset. It landed after the 0.6.0 release, so until the next release, stay on the default.

### Making it smooth

Out of the box it is CPU-bound in a built-up base: the main thread maxes out one core and the GPU sits around 35%, about 19 fps on screen. DroidDeck pins the game to cores 3 to 5 by design, so AYN's performance mode doesn't help. What did help:

| Change | Effect |
|---|---|
| DroidDeck frame generation **Win-FG 2×** | 19 → about 43 fps on screen in the base, picture stays clean |
| Valheim 3D render resolution 1080 → 720 | It was supersampling at 720p output for no gain |
| Grass/clutter low | Small CPU saving |
| Valheim FPS limit **30** | With Win-FG 2× this gives a **locked 60**: 59.7 fps measured, every frame 16 ms |

Valheim keeps these settings in `compatdata/892970/pfx/user.reg` inside DroidDeck's Steam data. Back that file up before editing it by hand.

**Shortcut:** DroidDeck › Games › Valheim › Shortcut › Add to home screen. In Cocoon that tile launches Steam and Valheim in about 30 seconds.

### Other Steam games through DroidDeck

Measured 22–23 Oct 2026. Frame generation in DroidDeck is one global setting: the per-game card edits the global one.

| Game | fps |
|---|---|
| Vampire Survivors, Brotato, Halls of Torment, BALL x PIT, Megabonk, Deep Rock Galactic: Survivor, The Binding of Isaac: Rebirth, Blazing Beaks | 59.8–60 on stock settings |
| Hades | 60 (frame gen off) |
| Terraria | 59 (frame gen off) |
| Stardew Valley | about 57 |
| DREDGE | 51 with frame gen off, 54–60 with it on |

Stardew's Steam Cloud save came down on first launch.

---

## 2. Discord voice on the bottom screen while gaming

Result: the game on the top screen, a Discord voice channel on the bottom screen.

- **Don't install Discord from Aptoide.** Its only build is 9.8.8 from 2019, and its login always fails with a misleading "This field is required".
- **Use [Aurora Store](https://f-droid.org/packages/com.aurora.store/)** (from F-Droid, anonymous login) to get current Discord without a Google account. If it says "Server busy, please try again later", the shared anonymous token is being throttled: clear Aurora's data (`adb shell pm clear com.aurora.store`) and start a fresh anonymous session.
- **Pinning apps to screens:** Wayfinder (package `app.wayfinder`) can set per-app rules: DroidDeck opens on the top screen with the bottom screen kept on, Discord opens on the bottom. After that, tap Discord, tap DroidDeck, play.
  - Wayfinder needs its accessibility service. If you turn it on over adb, **keep AYN's own service in the list** rather than replacing it.
  - Without Wayfinder you can move Discord yourself: `adb shell am start --display 4 -n com.discord/.main.MainDefault` (on my unit 0 = top and 4 = bottom).

### Headset: use wired

- **AirPods/Bluetooth earbuds are a poor fit.** With the earbuds' call profile on (the Bluetooth toggle is labelled "Handheld console calls"), the mic sounds robotic and game audio drops to mono. With it off, the mic switches to the Thor's built-in one (fine), but Discord voice plays out of the **speakers**. Android has no voice route over normal Bluetooth audio, and the firmware has no LE Audio.
- **A wired 3.5 mm headset with an inline mic just works.** It's detected as a headset and the mic is picked up automatically. A 2.4 GHz dongle headset should work too, but I haven't tested one.

---

## 3. Emulation fixes

- **Every RetroArch game hangs on a black screen from the frontend?** Check RetroArch actually has cores. Mine had none (the app-private cores folder had been wiped), and the log said `--libretro ... is not a file`. Fix: download the `arm64-v8a` cores from `buildbot.libretro.com/nightly/android/latest/arm64-v8a/`, put the `.so` files in RetroArch's downloads folder, then **Load Core › Install or Restore a Core**. Use the touchscreen for this menu: d-pad and keyboard input silently did nothing for me.
- **RetroArch defaults worth changing:** turn on automatic aspect ratio, and turn off the on-screen overlay (it's a handheld with real buttons).
- **Underclock the light emulators:** in AYN's ClusterTune you can set per-app profiles. "Medium Underclock" on RetroArch, WatermelonDS and PPSSPP kept SNES at 59.8 fps with lower clocks, for better battery and heat.
- **DS on two screens:** WatermelonDS (the renamed melonDualDS) puts one DS screen on each Thor screen.
- **Switch (Eden Duo) crashing on launch?** Mario Kart 8 Deluxe crashed on Eden Duo 1.3.1 and 1.3.2 with the Thor's own Qualcomm driver (a SIGSEGV in the `libllvm-qgl` shader compiler). A per-game GPU driver, **Turnip v27.0.0-R1**, fixed it. With render resolution 1x it ran at about 56–60 fps in a Grand Prix, with the dual-screen companion on the bottom screen.
- **3DS: rename `.3ds` to `.cci`.** Lime3DS 2123.3 (which is Azahar under the old package name) rejects the `.3ds` extension and accepts the same file as `.cci`.
- **Azahar layout settings:** set Landscape / Single Screen / Secondary = Bottom Screen from the **in-game** drawer › Settings › Layout. Set from the main menu, they didn't stick for me.
- **Keeping sideloaded emulators updated:** [Obtainium](https://github.com/ImranR98/Obtainium) tracks GitHub releases. For Cocoon, whose tags look like `beta-3.07`, set the version-trim regex to `[0-9.]+`. A sideloaded APK that Obtainium doesn't track never updates by itself. That includes Tailscale, so check it after each Tailscale security bulletin. pkgs.tailscale.com now only ships a universal APK, with no separate arm64 build.

---

## 4. Cocoon tips

- **Pinning a shortcut does nothing?** Cocoon's grid and its "Add Shortcut" dialog live on the **bottom** screen. If another app (for me, an Android Settings page) is covering the bottom screen, the pin request arrives but the dialog never shows, and nothing is added. Close whatever is on the bottom screen and pin again.
- **Switch games through Eden Duo:** Cocoon's Switch player list only has plain Eden. To launch a game in Eden Duo (for the dual-screen companions), go to Eden Duo › the game's page › the add-to-home icon (top left). In Cocoon's "Add Shortcut" dialog, set **Platform override = Nintendo Switch** and turn **Add to home** off. Then hide the plain-Eden copy that a rescan adds.
- **Rescan with a real Y press.** Tapping the on-screen Y badge didn't start the rescan for me; pressing Y did.
- **3DS player:** pick Cocoon's **"Azahar"** player entry, not "Lime 3DS". With Lime3DS 2123.3 installed, the "Lime 3DS" entry points at an activity the app doesn't have, so the launch fails.
- **Edit mode:** a drag only lands on an empty slot. Dropping onto an occupied tile puts it into a folder.

---

Not affiliated with AYN, DroidDeck, Valve or any of the projects above. Corrections welcome as issues.
