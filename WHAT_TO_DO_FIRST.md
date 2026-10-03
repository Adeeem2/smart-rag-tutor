# What to do first — ENET'Com Assistant

> Priority: **Reliable data → reliable retrieval → knowledge graph → multimodality → student model → RL.**
> Do NOT start with the chatbot, the ontology, or RL. Start with Phase 1: ingestion.

## 1. What you actually have (measured from `datajson/`)

- **9 catalogue files** = tracks × years: `GEC_1st_Year, GEC_2nd_Year, GII_1st/2nd, GT_1st/2nd, IDSD_1st/2nd, LTIC_1st_Year`.
- **18 top-level entries** (`subject: "semester 1" / "semester 2"` per file). These are semesters, not subjects — the field name is misleading.
- **~232 subject/subject-folder nodes** (2nd level), **~232 recursive subfolders**, **1336 file entries**.
- **Content split (approx):** ~739× `.pdf`, ~87× `.docx`, ~22× `.ppt/.pptx`, ~130× images (`.jpg/.png/.gif`), plus `.doc, .txt, .csv, .ipynb, .xls/.xlsx, .zip/.rar, audio (.m4a/.wma)`, code (`.c/.cpp/.css/.html/.xml`).
- **Max folder depth observed: 2** (semester → subject → subfolder), plus one extra level hidden inside `files[]` (see quirks).

### Data quirks you must handle now (not later)

1. **Subfolders hidden in `files[]`:** 191 entries with `mimeType: application/vnd.google-apps.folder` sit inside `files[]` instead of `subfolders[]`. Your traversal must check both.
2. **Confusing keys:** top level uses `subject` to mean semester; deeper levels use `name`. Normalize this.
3. **Missing metadata:** ~82 nodes have empty `mimeType`, 1 top-level node has empty `id`. Never assume fields exist — default to `""`.
4. **Encoding mojibake:** e.g. `"ProbabilitǸs "` instead of `"Probabilités"`. Plan a `fix_encoding()` / Unicode NFC normalization step.
5. **Duplicate folder names:** e.g. two folders both named `proba` with different Drive IDs. Use Drive `id` as identity, never the name.
6. **Inconsistent file entries:** 26 entries inside `files[]` themselves have `files/subfolders` keys; some filenames have no extension (95 entries). Classify by `mimeType` first, extension second.

## 2. Your first task: Phase 1 — catalogue / ingestion module

Goal: turn the 9 raw JSONs into **one normalized document table** with hierarchy and Drive URLs preserved. No text extraction, no embeddings, no LLM yet.

### Step 1 — Catalogue loader + recursive traversal (day 1)

Create `ingestion/catalogue.py` with:

- `load_catalogue(path) -> list` (one file).
- `walk(node, path_stack)` that visits **both** `subfolders[]` **and** folder-entries inside `files[]` (detect via `mimeType == "application/vnd.google-apps.folder"`).
- Carry context down: `(source_file, semester, subject, folder_path)`.
  - `source_file` = e.g. `IDSD_1st_Year.json` (this encodes track + year — keep it).
  - `semester` = top-level `subject` value (`"semester 1"`).
  - `subject` = 2nd-level `name`.
  - `folder_path` = full list from semester down.
- Output a flat row per file entry.

### Step 2 — Normalization (day 1–2)

Create `ingestion/schema.py` producing one record per file, exactly this schema:

```json
{
  "document_id": "<drive file id>",
  "file_id": "<drive file id>",
  "filename": "cours proba.pdf",
  "mime_type": "application/pdf",
  "source_url": "https://drive.google.com/file/d/.../view?usp=drive_link",
  "track": "IDSD",
  "year": "1st_Year",
  "semester": "semester 1",
  "subject": "Probabilités",
  "folder_path": ["semester 1", "Probabilités", "proba"],
  "document_type": "course | td | exam | correction | slide | image | other",
  "local_path": null,
  "status": "catalogued"
}
```

Rules:

- `document_id = file_id = Drive id`. Keep the original URL untouched.
- Derive `track/year` by parsing the catalogue filename (`GEC_1st_Year.json` → `GEC / 1st_Year`).
- `document_type`: heuristic on filename for now (`cours→course`, `td→td`, `exam/examen→exam`, `correction/corr→correction`, `slide/ppt→slide`, image mimes→image, else `other`). Keep it as a function so you can improve it later.
- Apply Unicode NFC + mojibake fix on `subject`/`filename` before storing.

### Step 3 — Stats + validation script (day 2)

Create `ingestion/stats.py` (or a notebook) that prints, over the merged corpus:

- # semesters, # tracks, # subjects, # folders, # documents, breakdown by `mime_type` / `document_type` / `track`.
- Validation checks (fail loudly):
  1. Every record has non-empty `document_id` + `source_url`.
  2. `folder_path[0]` == `semester`, `folder_path[1]` == `subject` (hierarchy preserved).
  3. Count of output rows == count of non-folder entries in raw JSONs (expect ~1145 real files = 1336 − 191 folder-entries; verify).
  4. No two records share a `document_id` (deduplicate across the 9 files if the same Drive ID appears twice).

Save output to `data/processed/corpus_index.json` + `data/processed/corpus_stats.json`.

### Step 4 — Sample + download probe (day 2–3)

1. Pick **one small subject** (e.g. one `proba` folder, ~5 PDFs + 1 image). Save its subset as `data/processed/sample_proba.json`.
2. Write `ingestion/drive_loader.py` with a single function `download(file_id, dest)` (Drive API or `gdown`). Download **only the sample** — do not bulk-download the whole Drive.
3. Confirm each downloaded file opens (PDF page count, image size) and record `local_path` + `status: downloaded`.

Stop here. If steps 1–4 pass, Phase 1 is done.

## 3. What NOT to do yet

- No chunking / embeddings / vector DB (Phase 3).
- No ontology extraction (Phase 5) — but keep `subject/folder_path` clean because the ontology will be built from it.
- No VLM, no student model, no RL.
- No bulk Drive download.

## 4. Suggested repo changes (minimal)

```text
ingestion/
  catalogue.py      # traversal of the 9 JSONs
  schema.py         # normalized record + document_type heuristic
  drive_loader.py   # single-file downloader (sample only)
  stats.py          # counts + validation checks
data/
  processed/
    corpus_index.json    # merged normalized catalogue (output of Phase 1)
    corpus_stats.json
    sample_proba.json    # small test subset
configs/
  corpus.yaml            # paths to the 9 JSONs, output paths (no hard-coding)
```

## 5. Definition of done for "first"

- [ ] All 9 JSONs load through one `walk()` handling both `subfolders[]` and folder-entries in `files[]`.
- [ ] `corpus_index.json` exists, one row per real file, with `document_id/file_id/source_url/semester/subject/folder_path/document_type`.
- [ ] Stats + 4 validation checks pass.
- [ ] One sample subject downloaded locally and opens correctly.
- [ ] Only then: move to Phase 2 (PDF/text/image extraction with page-level provenance).
