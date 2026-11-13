# EDMCModernOverlay-SteamOS

SteamOS-specific installation, compatibility patches and maintenance notes for [EDMCModernOverlay](https://github.com/SweetJonnySauce/EDMCModernOverlay).

**Verified environment (13 November 2026):** Steam Deck, SteamOS KDE Plasma **Desktop Mode** (Wayland with Xwayland), EDMC Flatpak 6.1.2 and EDMCModernOverlay Linux release 0.9.2. The complete test overlay was displayed successfully over Elite Dangerous. **SteamOS Gaming Mode was not tested.**

## Start here

- **[Fresh installation guide](INSTALL.md)** — setup, user-local `wmctrl`, Flatpak host-launch permission, both compatibility patches, syntax checks and in-game validation.
- **[Updating and restoring](UPDATE.md)** — backup of the known-good installation, release assessment, dry-run patching and rollback.
- **[0.9.2 installer patch](patches/0.9.2/install_linux.patch)** — lets the installer continue after declining package-manager installation.
- **[0.9.2 host-overlay PATH patch](patches/0.9.2/load.py.patch)** — adds `/home/deck/.local/bin` only to the external overlay client's `flatpak-spawn --host` environment, allowing it to find `wmctrl`.
- **[Detailed Google Doc](https://docs.google.com/document/d/18gadtdmyzNndZ0GQmyX49leWySxMAjn7_4jx7mu2sxU/edit)** — background and reproducible installation documentation.

## Why the two patches exist

SteamOS is read-only by default and the EDMC Flatpak is sandboxed. The upstream 0.9.2 Linux installer detected missing `python-pip` and `wmctrl` through `pacman` and aborted when the user declined installing system packages. We successfully ran the installer without writing to the protected system partition.

Separately, the overlay client runs on the SteamOS host and originally inherited a PATH excluding `~/.local/bin`. Thus a working, locally installed `wmctrl` was invisible to the client, disabling its window-follow mode and producing a clipped image. The targeted `load.py` patch resolved the problem.

**Do not set a global `PATH` override on the EDMC Flatpak.** That previously prevented EDMC from starting (`bwrap: execvp edmarketconnector: No such file or directory`).

## Version safety

These patches document the changes that worked with **0.9.2**; they are not guaranteed to match later versions. Always back up and run `patch --dry-run -p1`, inspect patch output (including any fuzz or offsets), and verify actual behaviour in-game. If upstream fixes the underlying issues, the patches may become unnecessary. The current patch files are reconstructed from the confirmed original/replacement blocks and **have not yet been dry-run against the original 0.9.2 release archive**; confirm applicability against a local extracted copy before relying on them.

This is an independent community compatibility resource, not a fork of or official distribution for EDMCModernOverlay. Upstream credits/licensing belong to its authors. No executables, personal Flatpak data or game logs belong in this repository.

## Links

- [Upstream project](https://github.com/SweetJonnySauce/EDMCModernOverlay)
- [Release downloads](https://github.com/SweetJonnySauce/EDMCModernOverlay/releases)
- [Steam Deck issue discussion](https://github.com/SweetJonnySauce/EDMCModernOverlay/issues/146)
