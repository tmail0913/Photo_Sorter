# Pipeline

Four steps, each its own script, each runnable alone. The dashboard runs them in order and watches the files they write.

```
photos folder ──scan.py──▶ dashboard/data/inventory.json + thumbs/ + mini/
                              │
                     dupes.py ─┴─▶ dashboard/data/dupes.json
                              │
        label_claude.py / label_jev.py / you ──▶ work/labels/*.json  ({"id": "key"})
                              │
                      apply.py ──▶ <output>/<Folder>/ + _Review/duplicates/ + sorted.csv + duplicates.csv
```

Everything the tool writes stays inside the skill folder: `dashboard/data/` (what the dashboard reads), `work/` (scratch), and the output folder the user chose. The photos folder is only ever opened for reading.

## 1. scan.py — inventory and previews

`python scripts/scan.py "<folder>" [--limit 200]`

- Walks the folder (skipping dotfiles and anything ending in ` (sorted)`), records size, sha256, dimensions, video duration, and for a Google Takeout export the date/GPS from the `.json` sidecar.
- Writes a 256px square preview to `dashboard/data/thumbs/<id>.jpg` and a 72px copy to `dashboard/data/mini/` for the fly-into-folder animation.
- Writes `dashboard/data/inventory.json` (list of `{id, path, name, ext, kind, size, sha256, w, h, duration, date}`) and `dashboard/data/source.json`.
- `--limit N` scans a random sample of N files — the right way to try 200 before committing to 20,000.
- Skips rescanning the same folder if it was scanned before.

## 2. dupes.py — duplicates

`python scripts/dupes.py`

- Groups by sha256: identical files.
- Then perceptual hashing plus a pixel check for screenshots, so two different app screens that look alike are not called copies; bursts and re-saved screenshots are.
- Writes `dashboard/data/dupes.json`: groups with `duplicate`, `kept`, `type` (`exact` / `near`) and paths, and a contact sheet at `work/duplicates.jpg` so the pairs can be eyeballed.
- Keeps the largest/first copy and marks the rest.

## 3. Labels

Three ways, all writing `work/labels/*.json` as `{"<id>": "<folder key>"}`:

| | How | Cost |
|---|---|---|
| **Claude Code** (`label_claude.py`) | Runs `claude -p` headless, 4 at a time, each call reading 3 contact sheets (~90 previews) and answering one JSON object | The user's Claude plan, no API key |
| **Jev** (`label_jev.py`) | TypeSafe Jev 1.14 image model, one question per file, 280px preview sent | Billed per file — estimate first, ask |
| **You, in chat** | `sheets.py` → `view_image` → write the JSON yourself | Free, slower for big libraries |

`label_claude.py` resumes: finished batches stay in `work/labels/claude_NN.json`, so pressing Start again only redoes what failed. `CLAUDE_WORKERS` and `CLAUDE_MODEL` override concurrency and model.

`sheets.py` numbers tiles with the inventory id; sheet N holds ids `30N`–`30N+29`, which is how you check coverage. `--sample` makes 3 random sheets for choosing folders.

## 4. apply.py — build the folders

`python scripts/apply.py "<output folder>"`

- Merges labels (Jev's answer wins at ≥70% confidence, otherwise Claude's), applies moves made in the dashboard (`work/overrides.json`), and sends any id with no label, or a label of the wrong kind, to the last folder of its kind.
- Copies into `<output>/<Label>/` and duplicates into `<output>/_Review/duplicates/`. On macOS/APFS, copies of files on the same disk are clones: instant, no extra space. Elsewhere they are real copies — say so if disk space is tight.
- Writes `sorted.csv` (`file, folder, original_path`) and `duplicates.csv` (`duplicate, kept_copy, type, duplicate_path, kept_path`).
- Refuses an output folder inside the source folder.
- Never moves, renames or deletes an original.

## server.py — the dashboard

`python scripts/server.py` (or `start.sh`). Listens on `127.0.0.1` only, default port 8765 (`PORT=…`).

- `GET /api/state` — counts, categories, whether a job is running; `GET /api/labels` — live label progress; `/data/…` serves previews.
- `POST` endpoints start each step and stream progress; the page polls and animates each preview into its folder as labels arrive.
- Folder picking uses the native dialog on macOS; elsewhere the page falls back to typing a path.

## demo.py

`python scripts/demo.py` draws ~1,057 placeholder images, labels and duplicates into `dashboard/demo/`, so `start.sh --demo` shows a full finished sort with no real photos. Use it for a screenshot or a screen recording.
