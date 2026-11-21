---
name: photo-sorter
description: Sorts a camera roll or photo library into folders (Selfies, Food, Travel, Screenshots, Talking heads, Screen recordings, or your own) and finds duplicate, burst and re-saved photos, with a live dashboard at 127.0.0.1:8765. Use when someone wants to sort or organise photos and videos, find or review duplicate photos, tidy a Google Photos Takeout or Apple Photos export, set up or open the photo sorter (github.com/kairothq/claude-photo-sorter), or says "open my photo sorter", "sort my camera roll", "find my duplicate photos". Ships the full tool (scripts/, dashboard/, categories.json) so it runs without downloading anything.
---

# Photo sorter

Sorts every photo and video in a folder into named folders, sends duplicates to `_Review/duplicates`, and shows it on a local dashboard. Originals are only read; sorted files are copies. Upstream: `https://github.com/kairothq/claude-photo-sorter` (MIT). This skill bundles that repo, so there is nothing to download.

```
photo-sorter/
  scripts/    scan.py  dupes.py  label_claude.py  label_jev.py  sheets.py  apply.py  server.py  demo.py  common.py
  dashboard/  index.html + app.js + css  (reads dashboard/data/, writes nothing)
  categories.json   the folders, one JSON object each: key, kind, label, emoji, rule
  requirements.txt  pillow, pillow-heif, imagehash
  setup.sh  start.sh  Open Photo Sorter.command   (macOS/Linux launchers)
  references/  folders.md  pipeline.md  troubleshooting.md
  assets/dashboard-demo.png
```

## Ground rules (never break)

1. **Never move, rename, delete or write into the user's photos folder.** The tool only reads originals and copies them elsewhere. If they want duplicates deleted, hand them `duplicates.csv` and let them decide.
2. **Ask before anything paid.** Jev (TypeSafe) bills per file: run `label_jev.py --estimate`, show the count and tokens, wait for a yes.
3. **Keep the library private.** Never commit or paste anything from `dashboard/data/`, `dashboard/peek/`, `work/` or `.env`. Previews are the user's photos.
4. **One question at a time**, numbered options, plain words ("folder", not "directory").

## First: where does this run?

The scripts run wherever Python 3.9+ and ffmpeg are. Sort out which case you are in before doing anything:

- **You are on the user's own machine** (their terminal, their Mac): run the launchers directly — `./setup.sh`, then `./start.sh`. The browser opens at http://127.0.0.1:8765.
- **You are in a sandbox that cannot see their disk** (the usual case for this agent): you cannot read their photos or open their browser. Do this instead:
  1. Run the demo here to prove the tool works, and screenshot it: `./setup.sh`, `./start.sh --demo` (or `python scripts/demo.py`), then fetch `http://127.0.0.1:8765/?demo` and show them the picture.
  2. Give them the two commands for their own machine (`setup.sh`, `start.sh`) plus the repo link, or the copy of this skill folder.
  3. Offer the chat route below if they would rather upload photos than run anything.

## Set up and open (the usual request)

```bash
cd <this skill folder>/photo-sorter
./setup.sh                 # venv + pillow, pillow-heif, imagehash
./start.sh                 # http://127.0.0.1:8765  (keeps running; opens the browser)
./start.sh --demo          # a fake camera roll, no photos needed
PORT=8766 ./start.sh       # if 8765 is taken
```

`start.sh` runs in the foreground and serves until killed. Start it, check it, and stop it inside one command when you only need to prove it works:

```bash
(PORT=8765 .venv/bin/python scripts/server.py > /tmp/srv.log 2>&1 &) ; sleep 4
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8765/     # expect 200
curl -s http://127.0.0.1:8765/api/state | head -c 300
```

Then tell them, in about three lines: the dashboard URL; click **Choose folder**, pick their photos, pick where the sorted copies go (default: a `<folder> (sorted)` next to them), press **Start sorting**; folders can be changed under **Advanced**. Later: double-click **Open Photo Sorter.command**, or ask "open my photo sorter".

If `pip` refuses with `--user install ... not visible in this virtualenv`, rerun the install as `PIP_USER=0 .venv/bin/pip install -r requirements.txt`. Without ffmpeg, videos still sort but get grey previews — say so rather than failing.

## The dashboard

| Control | What it does |
|---|---|
| **Choose folder** | Pick the photos folder (Google Takeout, Apple Photos export, any folder) and the output folder |
| **Start sorting** | Runs scan → duplicates → labels → folders, with progress, timer and estimate. About 4 min per 1,000 photos |
| **Advanced** | Shows the folders that will be made; remove, add or edit them; choose Claude or Jev; "try 200 photos first" |
| **Click a folder** | Browse it; click a photo for full size, arrow keys to flip, **Move to** another folder |
| **Sort with Claude** | Replays the animation afterwards (`?d=20` slows it) |
| **New sort** | New source and destination |
| **Open sorted folder** | Opens the result in the file manager |

## Chat route (when they would rather not use the dashboard)

```bash
.venv/bin/python scripts/scan.py "<photos folder>" [--limit 200]
.venv/bin/python scripts/dupes.py                  # look at work/duplicates.jpg
.venv/bin/python scripts/sheets.py --sample        # 3 sheets in work/sample/ → suggest folders
.venv/bin/python scripts/sheets.py                 # contact sheets, 30 previews each, ids on every tile
.venv/bin/python scripts/label_claude.py           # needs the `claude` CLI
.venv/bin/python scripts/apply.py "<output folder>"
```

**Labelling without the `claude` CLI.** If `claude` is not installed (often true here), label the sheets yourself with your own vision:

1. `scripts/sheets.py` writes `work/sheets/sheet_NN.jpg`, 30 previews per sheet, each tile tagged with its id (a video adds `VIDEO 12s`, a PNG adds `PNG`).
2. Open the sheets with `view_image`, five at a time, and read the ids.
3. Write `work/labels/<any-name>.json` as `{"id": "folder_key", ...}` covering **every** id — a missing id lands in the catch-all folder.
4. `scripts/apply.py "<output folder>"` builds the folders.

Sheet N covers ids `30N` to `30N+29`, so it is easy to check you have covered them all. For a large library, do this in batches and tell the user how many files are done.

## Duplicates

`dupes.py` compares sha256 fingerprints, then adds a pixel check for screenshots so two similar app screens are not mistaken for copies. Exact copies and look-alikes (bursts, re-saved screenshots) go to `_Review/duplicates`; the kept copy is the largest file and is named in `duplicates.csv` (`duplicate, kept_copy, type, duplicate_path, kept_path`). Nothing is deleted — say that plainly, and let them delete from the CSV.

## Folders

`categories.json` is the whole folder scheme: `key`, `kind` (`photo` or `video`), `label`, `emoji`, `rule` (the sentence the labeller reads). Offer three choices: the standard set, a set you suggest after looking at `work/sample/`, or their own list. Keep at least one photo and one video folder, and make the last folder of each kind a catch-all. Details and ready-made sets: `references/folders.md`.

## Output

```
<output>/Selfies/…            one folder per category
<output>/_Review/duplicates/  duplicates and near-duplicates
<output>/sorted.csv           file, folder, original_path
<output>/duplicates.csv       duplicate, kept_copy, type, duplicate_path, kept_path
```

Refuse to write the output inside the source folder (the next scan would count the copies twice) — `apply.py` already blocks it.

## After a run, report

Files checked, how many went to each folder, duplicates found and space they hold, where the sorted copies are, and that the originals are untouched. Then offer the next step: review `_Review`, change a folder, or sort another folder.

More detail: `references/pipeline.md` (how each step works and what it writes), `references/troubleshooting.md` (port busy, HEIC, missing `claude`, failed batches).
