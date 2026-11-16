# nxnative on the Mac: from a Switch game to an app

*Versión en español: [README.es.md](README.es.md)*

A guide to turning any Switch or Switch 2 game into a Mac app with [nxnative](https://github.com/seedds/nxnative). Mac only, no iOS.

Example setup used throughout this guide (adjust the paths to your own):

| What | Where |
|---|---|
| Repository | `~/nxnative` |
| Python with the dependencies (venv) | `/Volumes/BACKUP/venv` |
| Games (`.nup` / `.nsp`) | `/Volumes/BACKUP/Games/<game>/` |
| Keys (Switch 1 games only) | `~/keys/prod.keys` |
| Save copies | `~/Documents/nxnative-saves` |

---

## 0. Requirements (once)

1. **A Mac with Apple silicon** and macOS 26.6 or later.
2. **Xcode 27.2** in `/Applications/Xcode.app`, selected as the active developer tools:
   ```
   sudo xcode-select -s /Applications/Xcode.app
   xcodebuild -downloadComponent MetalToolchain
   ```
   To avoid `sudo`, put `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` in front of each build command instead.
3. **Homebrew tools:**
   ```
   brew install xcodegen zstd glslang
   ```
4. **Python with the dependencies**, in a venv on the external drive. Don't run `pip install` directly, because Homebrew blocks it.
   ```
   python3 -m venv /Volumes/BACKUP/venv
   /Volumes/BACKUP/venv/bin/pip install -r ~/nxnative/.github/requirements/tools.txt
   ```
   If pip shows `Ignoring invalid distribution -xxx` warnings, they're harmless: they come from the `._*` files macOS creates on exFAT drives. To clear them: `dot_clean -m /Volumes/BACKUP/venv/lib`.

---

## 1. Before each game

- **Update the repository:** `cd ~/nxnative && git pull`
- **Check that the game is supported:** look up its title ID (it's in the file name, for example `[0400486024DEC000]`) in `~/nxnative/dm/games/README.md`. If it isn't listed, the script tries a "provisional" build that may not start.
- **Read the game's note,** `~/nxnative/dm/games/<titleID>/README.md`. It says whether the game works on the Mac and what's missing.
- **Keep the game on the external drive,** never in your home folder.
- **Check free space:** `df -h ~`. A large game needs about **70 GB free on the Mac**:

  | What | Approximate size |
  |---|---|
  | Extracted game files | ≈ the game's size |
  | Compiled shaders | 10–15 GB |
  | The finished app | ≈ the game's size + 10 GB |

  If space runs out, the build fails. Worse, the system disk fills up.

---

## 2. Build the Mac app

```
cd ~/nxnative
/Volumes/BACKUP/venv/bin/python3 scripts/build_ipa_from_nsp.py \
  "/Volumes/BACKUP/Games/<game>/<file>.nup" \
  --platform mac-app \
  --work-dir ~/Games/<titleID>_build \
  -o ~/Games/"<Game name>.app"
```

- `--platform mac-app` builds a Mac app instead of an iOS IPA. **Don't forget it.**
- `--work-dir` is the work folder: extracted files and compiled shaders. It's best on the internal drive, which is faster. If you keep it, rebuilding the same game takes minutes instead of hours.
- `-o` is where the app goes. Put it on the internal drive so the game loads quickly.
- **Switch 1 games** (`.nsp`, or a folder of NSPs): add `-k ~/keys/prod.keys`. Switch 2 `.nup` files don't need keys.
- Always use the venv's Python (`/Volumes/BACKUP/venv/bin/python3`), not plain `python3`.

**Useful options:**

| Option | What it does |
|---|---|
| `--icon image.png` | Sets the app's icon, for when the game doesn't carry one |
| `--seen-shaders` | Compiles only the shaders already tested: minutes instead of hours. Some areas may draw incomplete |
| `-l es` | Sets the game's language, if the game has it |
| `--help` | Lists every option |

**How long it takes** (reference: Ocarina of Time, 24 GB, on an M4):

| Step | First time | Rebuild with the work folder |
|---|---|---|
| Unpack | ≈ 2 min | reused |
| Mac shaders | ≈ 45 min | reused |
| Pipeline archive (graphics for your GPU) | ≈ 27 min | reused |
| Pack and sign the app | ≈ 5 min | ≈ 5 min |

To leave it running and follow it from another window:
```
... command above ... > ~/Games/build.log 2>&1 &
tail -f ~/Games/build.log
```
Lines starting with `[RED]` are errors. The build ends with a `done: <app path>` line.

---

## 3. Play

- Open the `.app` from Finder. The first start takes longer.
- On the settings screen you choose the frame rate, resolution, mode and other options, then press **Play**. Hold Option while the app opens to see that screen again.
- DualSense, Xbox and Pro Controller controllers work, and so do Joy-Con 2 and the Pro Controller 2 over Bluetooth.

**Where everything is** (`<titleID>` is the game's):

| What | Where |
|---|---|
| Save and settings | `~/Library/Application Support/nxnative/<titleID>/` |
| Log, for checking errors | `~/Library/Logs/nxnative/<titleID>.log` |
| Setting variables | `~/Library/Application Support/nxnative/<titleID>/env.txt` |

---

## 4. Save copies in Documents

The app has a save sync feature that accepts any folder. To have it keep a copy in `~/Documents/nxnative-saves`:

```
mkdir -p ~/Documents/nxnative-saves
echo "DM_SAVE_SYNC=$HOME/Documents/nxnative-saves" >> ~/Library/Application\ Support/nxnative/<titleID>/env.txt
```

- Use the absolute path: `~` doesn't work inside `env.txt` (the double-quoted `$HOME` above is expanded by the shell before writing).
- The folder `~/Library/Application Support/nxnative/<titleID>/` only exists after you open the game once. If it doesn't exist yet, create it with `mkdir -p`.
- It takes effect the next time you open the game.
- Each game keeps a single file, `~/Documents/nxnative-saves/<titleID>/save.dmsync`, holding the whole save. It's updated a few seconds after each save and when you quit the game.
- If macOS asks whether the app may open Documents, say yes.
- Before replacing a save, the app keeps the old one in `save-backups` and holds on to the 10 newest.

---

## 5. Clean up afterwards

- **Once the app works, delete the work folder** (`~/Games/<titleID>_build`) to free 25 to 40 GB. The app doesn't depend on it. The downside is that a rebuild takes hours again.
- Keep the original `.nup` or `.nsp` on the external drive in case you need to rebuild the app.
- The `.app` is all you need to play.

---

## Problems we ran into

| Error | Fix |
|---|---|
| `xcodebuild requires Xcode, but active developer directory ... CommandLineTools` | `sudo xcode-select -s /Applications/Xcode.app`, or use `DEVELOPER_DIR=...` (see step 0) |
| `error: externally-managed-environment` (pip) | Install into the venv, not the system Python |
| `[RED] the current Python ... is missing modules` | You ran the script with `python3`; use `/Volumes/BACKUP/venv/bin/python3` |
| `[RED] ... Command Line Tools ... no iOS SDK` | Xcode isn't selected (see the first row) |
| Disk space runs out during the build | Stop the build (Ctrl+C), free up space and start it again with the same `--work-dir`: it picks up what's already done |

---

## Disclaimer

This is an unofficial, community-written guide and is not affiliated with Nintendo or with the nxnative project. Only use games and keys you have legally dumped from hardware you own.
