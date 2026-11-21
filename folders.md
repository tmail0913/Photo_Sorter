# Folders

Every folder the sorter can make lives in `categories.json` at the root of the skill:

```json
{"key": "food", "kind": "photo", "label": "Food & drinks", "emoji": "🍜",
 "rule": "meals, snacks, drinks, cafes, the food is the subject"}
```

- `key` — internal name, lowercase, unique. Labels refer to it.
- `kind` — `photo` or `video`. A file is only ever placed in a folder of its own kind.
- `label` — what the folder is called on disk and in the dashboard.
- `emoji` — dashboard only.
- `rule` — the sentence the labeller reads to decide. Make it concrete: list the subjects it covers, and say what it is *not*.

Rules that keep the result clean:

- At least one `photo` and one `video` folder, or `common.categories()` asserts.
- The **last** folder of each kind is the catch-all (`apply.py` sends anything unlabelled or mismatched there). Its rule should read like "anything else everyday".
- Videos and photos are separate trees; a talking-head video never lands in a photo folder.
- Roughly 8–14 folders works best. More folders means more mislabels, fewer means useless sorting.

## The standard set

Photos: Selfies · People & family · Pets & animals · Food & drinks · Travel & nature · City & nights out · Home & everyday · Documents & receipts · Memes & saved posts · Other screenshots.
Videos: Talking & selfie videos · Travel clips · Screen recordings · Everyday clips.

This is what ships in `categories.json`.

## Sets that fit a library

Suggest one of these after looking at `work/sample/` (three sheets of random previews). Always offer the standard set as option 1.

**Creator / founder** — Talking heads · B-roll & product · Screenshots & references · Events & people · Documents · Everything else. Videos: Talking heads · Screen recordings · B-roll.

**Parent** — Kids · School & art · Family & events · Pets · Food · Home & admin · Screenshots · Everything else. Videos: Kids · Family clips · Screen recordings.

**Student** — Notes & whiteboards · Slides & screenshots · Friends · Campus & city · Food · Travel · Receipts & admin · Everything else. Videos: Lectures & recordings · Friends · Screen recordings.

**Traveller** — Selfies · People · Food · Landscapes · Streets & cities · Monuments · Tickets & documents · Screenshots · Everything else. Videos: Talking · Scenery · Screen recordings.

## Changing folders mid-flight

- **Dashboard → Edit folders**: rename and re-describe; the dashboard writes `categories.json`.
- **Dashboard → Advanced → +**: add a folder; the bin removes one.
- In chat, edit `categories.json` directly, keep keys unique and the last folder of each kind a catch-all, then rerun `apply.py`. Labels already written stay valid for unchanged keys; a label whose key disappeared falls back to the catch-all.

Re-sorting is cheap: `scan.py` skips work when the same folder was scanned already, and `apply.py` just copies again into a new destination.
