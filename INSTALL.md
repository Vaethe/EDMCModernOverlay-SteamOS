# EDMCModernOverlay 0.9.2 on SteamOS — verified installation

**Tested 13 November 2026:** Steam Deck, KDE Plasma Desktop Mode (Wayland/Xwayland), EDMC Flatpak `io.edcd.EDMarketConnector` 6.1.2, EDMCModernOverlay Linux release 0.9.2, Python 3.14.6, display 1920×1200. Full test overlay rendered over Elite Dangerous. Gaming Mode is **not tested**.

All commands below run on the **SteamOS host**, never inside Fedora Distrobox. Leave SteamOS's protected system partition read-only.

## 1. Prepare EDMC
Verify EDMC Flatpak is installed, disable the older edmcoverlay2/EDMCOverlay plugin in EDMC, then close EDMC.

```bash
flatpak list --app | grep -i market
flatpak ps
# Only if EDMC is still running:
flatpak kill io.edcd.EDMarketConnector
```

## 2. Download Linux release
Get the *Linux release asset*, not the Windows version or generic GitHub source archive, from https://github.com/SweetJonnySauce/EDMCModernOverlay/releases . For the verified build download `EDMCModernOverlay-linux-0.9.2.tar.gz`.

```bash
cd ~/Downloads
mkdir -p EDMCModernOverlay-install
tar -xzf EDMCModernOverlay-linux-0.9.2.tar.gz -C EDMCModernOverlay-install
cd EDMCModernOverlay-install
# If necessary cd into the nested directory containing install_linux.sh
bash install_linux.sh --help
```

## 3. Make wmctrl available without pacman
`wmctrl` is **necessary** for the game-window tracking that fixed the overlay clipping. Install its executable in the home directory rather than unlocking SteamOS's system partition.

```bash
mkdir -p ~/.local/bin ~/Downloads/wmctrl-extract
curl -fL https://archlinux.org/packages/extra/x86_64/wmctrl/download/ -o ~/Downloads/wmctrl.pkg.tar.zst
bsdtar -xf ~/Downloads/wmctrl.pkg.tar.zst -C ~/Downloads/wmctrl-extract
install -m 755 ~/Downloads/wmctrl-extract/usr/bin/wmctrl ~/.local/bin/wmctrl
~/.local/bin/wmctrl -l
```

The last command should show open X11/Xwayland windows. Compatibility of *future* Arch binaries with SteamOS is not guaranteed: if necessary use `ldd ~/.local/bin/wmctrl` to diagnose library errors.

Optionally make wmctrl available to *interactive Konsole shells*:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
command -v wmctrl
```

This does **not** solve the Flatpak-spawned overlay client's PATH; that is step 7.

## 4. Patch the release installer
Return to the directory containing the extracted `install_linux.sh`. The release's pacman check lists `python-pip` and `wmctrl` as missing and ordinarily aborts when the user declines installation.

Back up and dry-run the version-specific patch:

```bash
cp install_linux.sh install_linux.sh.backup
git apply --check -p1  ~/Projects/EDMCModernOverlay-SteamOS/patches/0.9.2/install_linux.patch
```

If successful, apply it:

```bash
git apply -p1  ~/Projects/EDMCModernOverlay-SteamOS/patches/0.9.2/install_linux.patch
bash -n install_linux.sh
```

No output from `bash -n` means valid Bash syntax. This patch changes the unconditional abort into an optional `Skip dependency installation and continue?` prompt, mirroring the script's existing rpm-ostree approach. It does *not* install skipped requirements or guarantee the same behaviour for future releases.

## 5. Give EDMC Flatpak host-spawn permission
```bash
flatpak override --user --talk-name=org.freedesktop.Flatpak io.edcd.EDMarketConnector
flatpak override --user --show io.edcd.EDMarketConnector
```

Look for `org.freedesktop.Flatpak=talk`. This allows the Flatpak to request launching host processes; use only trusted plugin code.

**Do not set a global Flatpak PATH override.** That previously broke EDMC startup with `bwrap: execvp edmarketconnector: No such file or directory`.

## 6. Run modified installer
```bash
bash install_linux.sh
```

When offered system package installation for `python-pip` and `wmctrl`, answer **n**; when asked to **skip dependency installation and continue**, answer **y**. When KDE Wayland compositor-specific Qt overrides are offered, accept them:

```text
QT_AUTO_SCREEN_SCALE_FACTOR=0
QT_ENABLE_HIGHDPI_SCALING=0
QT_SCALE_FACTOR=1
```

The verified overlay process also ran with `QT_QPA_PLATFORM=xcb`, i.e. Xwayland. Allow the installer to finish.

## 7. Patch load.py to pass PATH ONLY to host overlay client
This is the essential fix. The overlay uses `flatpak-spawn --host`; that host process initially had no `/home/deck/.local/bin` in its PATH, so it could not execute `wmctrl`. The patch inserts `--env=PATH=/home/deck/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/bin` into **that host process's launch command**, not EDMC's Flatpak environment.

Close EDMC, then:

```bash
cd ~/.var/app/io.edcd.EDMarketConnector/data/EDMarketConnector/plugins/EDMCModernOverlay
cp load.py load.py.backup
git apply --check -p1  ~/Projects/EDMCModernOverlay-SteamOS/patches/0.9.2/load.py.patch
```

Only if the dry run succeeds:

```bash
git apply -p1  ~/Projects/EDMCModernOverlay-SteamOS/patches/0.9.2/load.py.patch
python3 -m py_compile load.py
```

No output means valid Python syntax. Never mix tabs and spaces; a `TabError` indicates indentation problems. If needed restore `cp load.py.backup load.py`. The path `/home/deck` assumes the default Steam Deck username; adapt it if different.

## 8. Verify in game
Start EDMC. Confirm `EDMCModernOverlay` appears enabled under **File → Settings → Plugins** with its own settings tab. For initial testing enable **Keep overlay visible when Elite Dangerous is not the foreground window**. In SteamOS **Desktop Mode**, launch Elite Dangerous and click **Test Overlay** in EDMC. Return focus to Elite Dangerous. The complete test artwork should display over the game.

Optional checks:

```bash
~/.local/bin/wmctrl -l
pgrep -af 'overlay_client|EDMCModernOverlay'
xwininfo -root -tree | grep 'EDMC Modern Overlay'
tr '\0' '\n' </proc/$(pgrep -f '^.*python3 -m overlay_client\.overlay_client$' | head -n 1)/environ | grep '^PATH='
```

The final PATH should include `/home/deck/.local/bin`.

## 9. Troubleshooting notes
- **Invisible/cropped image:** enable 'Keep overlay visible'; inspect `overlay_client.log` for `wmctrl binary not found; overlay follow mode disabled`. Before the PATH fix, observed windows were 44×169 or 640×480, clipping test artwork; the Xwayland desktop/game itself correctly reported 1920×1200.
- **EDMC won't start after environment changes:** check `flatpak override --user --show io.edcd.EDMarketConnector` and user override file `~/.local/share/flatpak/overrides/io.edcd.EDMarketConnector`. Remove accidental `[Environment] PATH=` and/or `unset-environment=PATH;` settings while preserving `org.freedesktop.Flatpak=talk`.
- **Unrelated traceback:** `TypeError` from `state['Modules']` was from EDMC-NavRoute, not EDMCModernOverlay.
- **Logs:** `~/.var/app/io.edcd.EDMarketConnector/data/EDMarketConnector/logs/EDMCModernOverlay/overlay_client.log`.
- **Gaming Mode:** not verified.

More extensive narrative guide: https://docs.google.com/document/d/18gadtdmyzNndZ0GQmyX49leWySxMAjn7_4jx7mu2sxU/edit

After successful installation see [UPDATE.md](UPDATE.md).
