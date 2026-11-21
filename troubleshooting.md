# Troubleshooting

## Setup

**`pip` fails: "Can not perform a '--user' install. User site-packages are not visible in this virtualenv."**
The environment sets `PIP_USER=1`. Run `PIP_USER=0 .venv/bin/pip install -r requirements.txt` (or `PIP_USER=0 ./setup.sh`). Seen in sandboxes and some CI images.

**"Python 3 is missing."** macOS: `xcode-select --install`, or install from python.org. Linux: install `python3` and `python3-venv` from the distro.

**"ffmpeg is not installed, so videos get grey previews."** Not fatal — photos sort normally and videos still land in their folders, just without a preview frame. macOS with Homebrew: `brew install ffmpeg`. Debian/Ubuntu: `apt-get install ffmpeg` (needs root).

**"A folder named `.venv` already exists."** Delete it and rerun `./setup.sh` — it is only packages.

## Starting

**"Photo sorter is already running."** Fine: `start.sh` just opens the page. If the old server is stale, kill it (`lsof -ti:8765 | xargs kill`).

**Port 8765 busy.** `PORT=8766 ./start.sh`, then use http://127.0.0.1:8766.

**The browser does not open.** Open http://127.0.0.1:8765 yourself. On a headless machine there is nothing to open — `curl http://127.0.0.1:8765/api/state` proves it is up.

**Nothing listens after starting.** Read the server log. If it says the port is in use, see above; if Python traceback mentions a missing module, the venv was not set up — run `./setup.sh` again.

## Sorting

**"`claude` command not found" during labelling.** The server's environment does not have the Claude Code CLI on PATH. Either start it from a terminal where `claude` works, or label without it: `scripts/sheets.py` + `view_image` + `work/labels/*.json`, or Jev under Advanced.

**A batch failed.** Press **Start sorting** again; finished batches are kept and only the failures rerun. To see why, run `label_claude.py --only 1` in a terminal and read the output.

**Jev 401.** Wrong key in `.env`. **Jev 400 "Invalid image dimensions".** Leave `JEV_MODEL` unset.

**HEIC photos fail to open.** `.venv/bin/pip install -U pillow-heif`, then rescan.

**Everything landed in one folder.** No labels were written (or every label key was wrong), so `apply.py` used the catch-all. Check `work/labels/*.json` exists and its keys match `categories.json`.

**"Put the sorted folders outside the photos folder".** The output was set inside the source. Pick a sibling folder, e.g. `~/Pictures/Camera roll (sorted)`.

**A photo is in the wrong folder.** Click it in the dashboard and use **Move to** — that is remembered in `work/overrides.json` and survives a re-sort.

**Some photos are missing from the output.** Look in `_Review/duplicates` first: bursts and re-saved screenshots go there by design. Then check the extension is in `common.IMG`/`common.VID`.

**Too many photos called duplicates.** `dupes.py` groups anything within a perceptual-hash distance of 4, which is deliberately tight for bursts and re-saves but will also group near-identical frames (a still scene shot five times, or flat synthetic test images). Check `work/duplicates.jpg` — it shows each group with the keeper tagged `KEEP` first — and move the false positives back in the dashboard (**Move to**). The keeper is always the largest file in the group.

**Everything was labelled but a whole batch is in the catch-all.** Those ids were missing from the label JSON. Sheet N holds ids `30N`–`30N+29`; compare that range with the keys in `work/labels/*.json`.

**Re-sorting is slow the first time only.** `scan.py` reuses the inventory for an unchanged folder; changing folders does not need a rescan.

## Privacy checks before you finish

- The server listens on `127.0.0.1` only — nothing on the network can reach it.
- `dashboard/data/`, `dashboard/peek/`, `work/` and `.env` hold the user's previews and must never be committed, uploaded or pasted into a reply.
- With Claude labelling, previews are read by the model; with Jev, 280px previews go to TypeSafe. Originals never leave the machine. Say which one is in use.
