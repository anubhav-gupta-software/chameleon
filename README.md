# CHAMELEON

CHAMELEON reads a design JSON file and a target canvas, returns the same elements with adjusted geometry and optional text font sizes, and renders a PNG. Stage one runs offline in a container. Copy, font family, brand colors, logo, and asset pixels stay as supplied. The engine rearranges elements. It does not stretch the whole poster and it does not regenerate assets.

## Approach

`resize()` is a deterministic reflow engine, not a scale/stretch transform. No model is required; an optional text-context model (see Model, below) can replace the geometric role guess and importance numbers, but font fitting, placement, and collision cleanup stay the same either way.

1. **Role classification.** Each element is tagged from its geometry, type, z-index, and `group_id` alone — never its id or content — as a full-bleed background, logo, hero image, headline, body text, legal/fine print, a pill group (CTA or badge), a decorative shape, or a rotated ribbon accent. This is what lets the same logic generalize to designs it has never seen.
2. **Archetype candidates.** The target canvas's aspect ratio (and, for very short canvases, its absolute height) narrows the choice to one of three layout templates: a vertical stack (portrait/near-square), a two-column layout (wide banners/posters), or a single-row layout (ultra-thin strips, e.g. a 728×90 leaderboard).
3. **Importance scoring.** Each classified item gets a continuous 0–1 score from its role plus its own font size/area relative to the rest of *this* design — an unusually large badge or an already-dominant headline earns more room than its role alone would give it.
4. **Space allocation and placement.** Each archetype allocates space by importance weight, then places each element (a group moves as one rigid unit, so a button's background and label never drift apart). Images are aspect-preserving contain-fit (except `fit:cover`, which crops by design); text's `font_size` is shrunk or grown to fit using the real renderer's own font metrics, floored at an importance-scaled minimum so a headline never shrinks as far as fine print.
5. **Decorative passthrough.** Ungrouped shapes and rotated accent groups are scaled proportionally instead of forced into the flow — they're meant to bleed or overlap by design.
6. **Candidate scoring.** Every candidate from step 2 is built in full and scored on unwanted collisions (weighted by importance, extra-penalized for text sitting on an image), compressed/illegible text, and anything left off-canvas; the lowest-cost candidate wins.
7. **Collision resolution.** The winning candidate gets a cleanup pass that pushes apart any two message-bearing units still overlapping, splitting the separation in inverse proportion to importance.
8. **Clamp and snap.** A final pass keeps everything but an already-intentionally-bleeding decorative element on-canvas, then snaps every element's position (never its size) to a small pixel grid.

## Setup

Python 3.10–3.12. From a checkout:

```bash
python3.12 -m venv .venv-chameleon
source .venv-chameleon/bin/activate
python -m pip install -r requirements.txt
python -m pip install jsonschema
python -m unittest tests.test_text_context tests.test_layout_eval
```

`python -m engine.main` reads every `input/*.json`, writes `<task>_output.json`, `<task>_output.png`, and `<task>_sidecar.json` to `output/`, and checks the Adobe schema plus the local contract. If the real renderer cannot be imported, the command fails. It does not write a labeled-box placeholder.

## Model

One optional `.gguf` lives in `models/` and is gitignored. The image does not download weights. `llama-cpp-python` is installed at build time only when that file is present (`requirements-text.txt`). The runtime does not call Ollama.

`CHAMELEON_TEXT_MODE` controls the path:

- `auto` (the default) uses the packaged file when it is present. With no file, the sidecar says `no-model` and the engine still writes JSON and PNG.
- `off` never calls a model.
- `on` requires a successful model judgment. A missing file or a failed call exits 2 and writes no JSON or PNG.

The model reads the whole design and returns a role, prominence, and must-read flag for each text element. It does not choose coordinates or font sizes. The prompt in this tree is `text-context-p5`. A judged layout is kept only when it does not add a text collision, add an overflow, or make an essential line less readable than the no-model layout. A price ranked `primary` or `high` keeps the lead when the no-model layout already had it ahead of the other line. A quieter price may still shrink. A failed call recorded as `fallback` is not a model result.

`layout_decision` is only that text-layout choice: `judged`, `baseline_kept`, or `baseline`. When the model leaves both the source headline slot and the source disclosure slot empty, the engine compares a second candidate that puts those source elements back and keeps it only when the measured picture is clearer. The sidecar then sets `layout_override` and leaves the model's roles unchanged. When a rotated accent the model ranked `discount` at `primary` or `high` then moves, the sidecar sets `offer_moved` to true. That flag does not mean the judged text layout was kept. A rotated group the model ranked `call_to_action` at `primary` or `high` can use that same corner search when it is the only action and the short side has room. The sidecar then sets `accent_moved`. The stored role stays `call_to_action`. A badge, and a second action beside that ribbon, do not take the corner. The move is checked for readability and for collisions with glyphs and visible foreground shapes. A shape or image that already covers the canvas is the backdrop and is not one of those obstacles.

The provisional weight used for local checks is Llama 3.2 3B, Q4_K_M, Llama 3.2 Community License, sha256 `dde5aa3fc5ffc17176b5e8bdc82f587b24b2678c6c66101bf7da77af9f7ccdff`, 2,019,377,376 bytes. Adobe has not approved this model or published a size cap. Do not treat it as the submission model until that decision exists. `python3 scripts/check_packaged_model.py` requires exactly one GGUF, checks the header, and prints the size and sha256. Pass `--expected-sha256` and `--max-bytes` when those values are known. The check does not grant approval.

## Offline container

Build with Buildx and select `linux/amd64` on the command. The Dockerfile does not pin a platform: the legacy `docker build --platform` flag does not select the base image, and a `FROM --platform` pin fails when the local `python:3.11-slim` tag is only arm64.

```bash
docker buildx build --platform linux/amd64 --load -t my-chameleon-engine .
docker run --rm --network none \
  -v "$(pwd)/input:/app/input" \
  -v "$(pwd)/output:/app/output" \
  my-chameleon-engine
```

This submission does not package a `.gguf` (see Model, above — none is Adobe-approved yet), so it runs the no-model path by default (`CHAMELEON_TEXT_MODE` defaults to `auto`, which falls back cleanly when no model file is present). **Do not set `CHAMELEON_TEXT_MODE=on`** without a packaged model — `on` requires a successful model judgment and exits 2, writing nothing, if `models/` has no `.gguf`.

If you do package a `.gguf` and want to confirm it's actually being used, first verify the file:

```bash
python3 scripts/check_packaged_model.py \
  --expected-sha256 dde5aa3fc5ffc17176b5e8bdc82f587b24b2678c6c66101bf7da77af9f7ccdff
```

then add `-e CHAMELEON_TEXT_MODE=on -e CHAMELEON_TEXT_NO_CACHE=1` to the `docker run` command above and confirm each sidecar says `model` and `model_judgment` is true. These checks are local. They are not Adobe's hidden score.

The linux/amd64 image `chameleon-quality-amd64:center-2026-10-26` (created 2026-10-26) is an older build from prompt `text-context-p4`. It does not contain the current logo placement or the offer move. A container run of this tree still has to be built from the current source.

## Output contract

Each output JSON keeps the same element ids, count, text, font family, and asset references as the source. Text font size and geometry may change. Painted bounds stay on the canvas. Image aspect ratio stays as supplied. Published JSON keeps source opacity, rotation, and z-index. A source circle that would hang off the target is scaled by the same factor on both axes and then moved inside.

## Known limits

- On a banner at least six times as wide as it is tall, the logo can leave the headline row when that makes the headline larger, the classified badge labels, the action, and the legal line stay the same size, and the parked logo keeps at least 65% of the height it had on the row. The badge check uses those pill groups, not the element id. That canvas uses the single-row layout even when it is taller than 160 px. A wide canvas up to 220 px tall also tries the row, so a line of chips cannot be the only candidate. A sparse canvas (a few lines, no product, no pills) also tries a left column sized to the words, and the right side stays open. Other canvases still consider the wide card from 160 px. On the official 728×90 banner the headline is 13 px. The logo is smaller and sits in the lower band. Badges stay about 7.6 px. The legal line stays present.
- A rotated accent the model ranks as a high or primary discount can move, with its shape, into open space when 22% of the short side is at least a 28 px label. A 90 px banner and a 120 px banner cannot. A 220 px canvas can, even when it is wider than 6:1. The accent has to grow by at least 35% of its own size, and it stays inside 46% of the width and 34% of the height. Aspect and rotation stay. A solitary rotated call to action may also move under the constraints above; its stored role is not rewritten. On the 1200×628 sale card a fresh model call placed the offer at 52.8 px, the price at 76.8 px, and the product name at 44.5 px. The sidecar for that card stays `baseline_kept` and sets `offer_moved` to true.
- A center-aligned text box that the layout already placed on the canvas center is not slid sideways by the background search. A center-aligned line in a side column can still move. Vertical nudges and collision checks remain.
- `fit: cover` may still crop through the renderer. Asset files are not edited.
- No human-labeled accuracy score is claimed. Llama 3.2 3B still calls some quantity deals and time marks a call to action. A tighter wording, `text-context-p6`, was run uncached and reverted: on the 1200×628 sale card it called “50% OFF” a call to action and the ribbon went from 52.8 px to 14.7 px, and it demoted “Shop the Summer Line.” The prompt in this tree remains `text-context-p5`.
