# Safe update and rollback

These patches are specific to upstream **EDMCModernOverlay 0.9.2**. A later release may fix the issues or restructure the code. Do not apply them blindly or force them onto changed code.

## Back up current working installation
Close EDMC. This backup is **local**, never upload it to a public repository.

```bash
flatpak kill io.edcd.EDMarketConnector
PLUGIN="$HOME/.var/app/io.edcd.EDMarketConnector/data/EDMarketConnector/plugins/EDMCModernOverlay"
MAINT="$HOME/Applications/EDMCModernOverlay-maintenance"
mkdir -p "$MAINT"
tar -czf "$MAINT/EDMCModernOverlay-0.9.2-working.tar.gz" -C "$(dirname "$PLUGIN")" "$(basename "$PLUGIN")"
```

The archive includes the plugin and its virtualenv but may **not** include EDMC preferences/configuration stored outside the plugin folder. Back up relevant settings separately. Keep a copy of the working upstream Linux release tarball too.

## Upgrade checklist
1. Read upstream release notes: https://github.com/SweetJonnySauce/EDMCModernOverlay/releases
2. Download/extract the **new** Linux release into a new directory; do not delete the known-good installer.
3. Check whether the upstream developer has fixed package checking and host `wmctrl` discovery.
4. If still necessary, `git apply --check -p1` against the **new version** of `install_linux.sh` and/or `load.py`. Stop on failed/misaligned patch; do not use `--force`. Consider changes if the hunk applies with unexpected offset/fuzz.
5. Check syntax with `bash -n install_linux.sh` and `python3 -m py_compile load.py`.
6. Install and manually test full overlay rendering over Elite Dangerous in **Desktop Mode** before treating the update as successful.
7. Record the new versions and outcome; preserve a separately named patch set for the new version.

Do not assume Gaming Mode works solely because Desktop Mode works.

## Roll back
Close EDMC. Only proceed if the backup exists; preserve the failing copy separately:

```bash
flatpak kill io.edcd.EDMarketConnector
PLUGIN="$HOME/.var/app/io.edcd.EDMarketConnector/data/EDMarketConnector/plugins/EDMCModernOverlay"
MAINT="$HOME/Applications/EDMCModernOverlay-maintenance"
test -f "$MAINT/EDMCModernOverlay-0.9.2-working.tar.gz" || { echo "No backup found"; exit 1; }
mv "$PLUGIN" "$PLUGIN.failed-$(date +%Y%m%d-%H%M%S)"
tar -xzf "$MAINT/EDMCModernOverlay-0.9.2-working.tar.gz" -C "$(dirname "$PLUGIN")"
```

Restart EDMC and test. Caveat: an archived virtualenv might not remain usable following major Python/SteamOS updates, in which case reinstall the compatible older release and apply the patches instead.

## Preserve exact local modifications
If the working installation still contains the originally saved `load.py.backup`, derive an exact patch from the **actual working files**:

```bash
cd ~/.var/app/io.edcd.EDMarketConnector/data/EDMarketConnector/plugins/EDMCModernOverlay
diff -u --label a/load.py --label b/load.py load.py.backup load.py > "$HOME/load-verified.patch"
```

Exit status 1 means the files differ; it is normal. Review for personal information before committing. Similarly, diff `install_linux.sh.backup` against the successful patched installer.

For future versions, prefer a dedicated Git branch / PR with release-specific patches and a test report. Keep backups, logs, virtualenvs and secrets **out of GitHub**.