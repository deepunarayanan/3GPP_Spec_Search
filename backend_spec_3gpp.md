# Backend Spec — 3GPP Feature Spec-Reference Summarizer

> Derived from `goal_spec_3gpp.md`. Defines **how** the server side works: stack, data model, spec ingestion, search, summary generation, PDF export and the REST API. The REST API section (§7) is the contract the frontend spec (`frontend_spec_3gpp.md`) builds against — change it in one place only.

---

## 1. Scope

The backend must:

1. Register and authenticate users (username + password).
2. Ingest the 3GPP specification PDFs in the `Standards/` folder into a searchable index of numbered sections.
3. Accept a summary request (feature name, technology, target release, specs to search, target use cases).
4. Find the exact subsection-level sections relevant to the feature in each selected spec, grouped by baseband layer (L1, MAC, RLC, RRC), and match each result to the relevant 3GPP clause and page reference.
5. Produce a short, cited summary per layer that includes the exact subsection numbers, titles, and brief extracted standard details for each feature-related clause, plus a scheduler-impact analysis on throughput, latency and UE capacity.
6. Store each summary against the user so they can see it again later.
7. Render the summary as a PDF of **at most 5 pages**.

Out of scope (from goal spec §6): device-side (UE) design, proprietary algorithms, and any post-MVP feature.

---

## 2. Technology choices

Chosen for a 1-person, 2-weekend build: few moving parts, one process, no external infrastructure.

| Concern | Choice | Why |
|---|---|---|
| Language / framework | Python 3.11+, **FastAPI** | Best PDF tooling is in Python; FastAPI gives validation + OpenAPI docs for free |
| Database | **SQLite** (one file, `data/app.db`) | Zero setup; includes **FTS5** full-text search |
| ORM / migrations | SQLAlchemy 2.x + Alembic | Standard; Alembic optional for v1 (can `create_all`) |
| PDF parsing | **PyMuPDF** (`fitz`) | Fast; reads PDF bookmarks (`get_toc()`) and page text |
| Search | SQLite FTS5 with BM25 ranking | Good enough for spec text; no vector DB needed |
| Summary generation | Claude API (Anthropic Python SDK), model set by env var | Turns retrieved section text into short summaries and the KPI analysis |
| PDF output | **WeasyPrint** (HTML + CSS → PDF) | Templates in HTML/Jinja2; page count is easy to check after rendering |
| Password hashing | `passlib[bcrypt]` (or `argon2-cffi`) | Industry standard |
| Auth tokens | JWT (`pyjwt`) in an **httpOnly cookie** | No token handling in frontend JS |
| Background work | FastAPI `BackgroundTasks` (in-process) | Generation takes 20–60 s; no queue server needed for 1 user/low volume |
| Tests | pytest + httpx `TestClient` | |

---

## 3. Project layout

```
backend/
  app/
    main.py              # FastAPI app, routers, CORS, static frontend mount
    config.py            # Settings from env (pydantic-settings)
    db.py                # engine, session, FTS5 setup
    models.py            # SQLAlchemy models (§5)
    schemas.py           # Pydantic request/response models (§7)
    auth/                # signup, login, JWT cookie, current_user dependency
    specs/
      catalog.py         # spec number → layer / technology mapping (§4.2)
      ingest.py          # CLI: parse Standards/*.pdf into sections (§4)
      search.py          # retrieval (§6.2)
    summaries/
      router.py
      generator.py       # pipeline orchestration (§6)
      llm.py             # Claude API calls + prompts
      validate.py        # citation checks
    pdf/
      render.py          # WeasyPrint + 5-page enforcement (§8)
      templates/summary.html, summary.css
  tests/
  data/                  # app.db (git-ignored)
  Standards/             # input PDFs (or path via env)
```

---

## 4. Spec ingestion

Run once (and again whenever PDFs are added): `python -m app.specs.ingest --dir ./Standards`. Idempotent: a file whose SHA-256 is already indexed is skipped.

### 4.1 Identify each document from its file name

ETSI file names encode spec number and version: `ts_1382` + `11` + `v` + `19` `02` `00` + `p.pdf`.

```
regex: ^ts_1(3[68])(\d{3})v(\d{2})(\d{2})(\d{2})p\.pdf$
ts_138211v190200p.pdf → TS 38.211, V19.2.0, Release 19
```

- `spec_number` = `"38.211"`; `version` = `"19.2.0"`; `release` = major version number (V19.x.x → Rel-19).
- Files that do not match are logged and skipped.

### 4.2 Spec catalog (layer + technology)

Static mapping in `catalog.py`:

| Layer | 5G NR | 4G LTE |
|---|---|---|
| L1 (Physical) | 38.201, 38.211, 38.212, 38.213, 38.214 | 36.201, 36.211, 36.212, 36.213, 36.214 |
| MAC | 38.321 | 36.321 |
| RLC | 38.322 | 36.322 |
| RRC | 38.331 | 36.331 |

Technology is derived from the series: `38.` → 5G, `36.` → 4G.

**Current `Standards/` folder contents (checked against the PDFs in this workspace):** 4G — 36.201 (V17.1.0), 36.211 (V17.1.0), 36.212 (V17.1.0), 36.213 (V17.4.0), 36.214 (V16.2.0), 36.321 (V18.4.0), 36.322 (V15.1.0), 36.331 (V18.1.0); 5G — 38.201 (V15.0.0), 38.211 (V16.2.0 and V19.2.0), 38.212 (V19.4.0), 38.213 (V19.4.0), 38.214 (V19.3.0), 38.321 (V18.5.0), 38.322 (V19.1.0), 38.331 (V19.1.0). The API reports missing documents so the UI can say so instead of silently returning nothing (§6.4 "gaps").

### 4.3 Split into sections

1. Open with PyMuPDF. Use the PDF bookmarks (`doc.get_toc()`) as the section list — ETSI PDFs include them. Each entry gives level, title and start page.
2. Fallback when bookmarks are missing: scan page text for heading lines matching `^(\d+(\.\d+)*|[A-Z]\.\d+(\.\d+)*)\s+\S` (e.g. `5.1.6.2 ...`, `A.3 ...`).
3. Section text = text from its heading to the next heading at the same or higher level.
4. Preserve the exact 3GPP subsection number and title in every record so the system can output citations such as `TS 38.214 §5.1.2.2 — Resource allocation in frequency domain` and map them back to the feature.
5. Strip ETSI page headers/footers (lines repeating on most pages, e.g. `ETSI TS 138 214 V19.3.0 (2025-xx)`, page numbers).
6. Skip non-technical sections: Foreword, Modal verbs terminology, Scope, References, Definitions/Abbreviations (keep abbreviations in a separate table — useful for query expansion), Change history, History.
7. Sections longer than ~6,000 characters are split into chunks (with ~300-char overlap) that keep the parent section number.
8. **TS 38.331 (RRC) special case:** the ASN.1 blocks are large. Also extract every ASN.1 field/IE name (e.g. `CellGroupConfig`, `pdsch-TimeDomainAllocationList`) with its section number into an `ie_names` column, so feature-specific RRC fields can be matched by name.

### 4.4 Store

Rows go into `spec_documents` and `spec_sections` (§5), and the section text is indexed in the FTS5 table. Expected index size for the current folder: a few tens of thousands of sections, well under 200 MB.

---

## 5. Data model (SQLite)

```sql
users (
  id            INTEGER PRIMARY KEY,
  username      TEXT UNIQUE NOT NULL,      -- stored lower-case
  password_hash TEXT NOT NULL,
  created_at    TIMESTAMP NOT NULL
)

spec_documents (
  id           INTEGER PRIMARY KEY,
  spec_number  TEXT NOT NULL,              -- "38.214"
  title        TEXT,                       -- from PDF first page, e.g. "NR; Physical layer procedures for data"
  version      TEXT NOT NULL,              -- "19.3.0"
  release      INTEGER NOT NULL,           -- 19
  technology   TEXT NOT NULL CHECK (technology IN ('4G','5G')),
  layer        TEXT NOT NULL CHECK (layer IN ('L1','MAC','RLC','RRC')),
  file_name    TEXT NOT NULL,
  sha256       TEXT UNIQUE NOT NULL,
  page_count   INTEGER,
  ingested_at  TIMESTAMP NOT NULL,
  UNIQUE (spec_number, version)
)

spec_sections (
  id            INTEGER PRIMARY KEY,
  document_id   INTEGER NOT NULL REFERENCES spec_documents(id),
  number        TEXT NOT NULL,             -- "5.1.6.2"
  title         TEXT NOT NULL,
  level         INTEGER NOT NULL,
  page_start    INTEGER NOT NULL,
  page_end      INTEGER NOT NULL,
  chunk_index   INTEGER NOT NULL DEFAULT 0,
  text          TEXT NOT NULL,
  ie_names      TEXT                       -- space-separated ASN.1 names (RRC only)
)

-- FTS5 index over sections (external content)
CREATE VIRTUAL TABLE spec_sections_fts USING fts5(
  title, text, ie_names, content='spec_sections', content_rowid='id',
  tokenize = "unicode61 tokenchars '-_'"
);

abbreviations (
  document_id INTEGER REFERENCES spec_documents(id),
  abbr        TEXT NOT NULL,               -- "BWP"
  expansion   TEXT NOT NULL                -- "Bandwidth Part"
)

summaries (
  id            TEXT PRIMARY KEY,          -- UUID v4
  user_id       INTEGER NOT NULL REFERENCES users(id),
  feature_name  TEXT NOT NULL,
  technology    TEXT NOT NULL,             -- '4G' | '5G'
  target_release INTEGER NOT NULL,         -- e.g. 17
  spec_numbers  TEXT NOT NULL,             -- JSON array: ["38.214","38.321","38.331"]
  use_cases     TEXT NOT NULL,             -- JSON array of strings (may be empty)
  status        TEXT NOT NULL CHECK (status IN ('queued','processing','ready','failed')),
  progress_step TEXT,                      -- see §6.1, for the UI progress display
  error_message TEXT,
  result        TEXT,                      -- JSON, shape in §6.4; NULL until ready
  created_at    TIMESTAMP NOT NULL,
  updated_at    TIMESTAMP NOT NULL
)
CREATE INDEX ix_summaries_user ON summaries(user_id, created_at DESC);
```

Summaries are immutable once `ready`. "Edit inputs and regenerate" creates a **new** summary (keeps history simple).

---

## 6. Summary generation pipeline

Runs as a background task after `POST /api/summaries`. Updates `status` and `progress_step` as it goes.

### 6.1 Steps

| # | `progress_step` | What happens |
|---|---|---|
| 1 | `selecting_documents` | Pick one document per requested spec (§6.3) |
| 2 | `expanding_query` | Build search terms for the feature (§6.2) |
| 3 | `searching` | FTS5 search per spec; keep top candidates |
| 4 | `ranking` | LLM picks the truly relevant sections per layer |
| 5 | `summarizing` | LLM writes section summaries, overview, KPI impact, use-case notes |
| 6 | `validating` | Check every cited section exists in the DB; drop any that don't |
| 7 | — | Save `result`, set `status = ready` |

Any unhandled error → `status = failed`, `error_message` set to a user-readable message (stack trace only in server logs). Target: whole pipeline < 90 s; hard timeout 180 s.

### 6.2 Retrieval

1. **Query expansion.** One LLM call returns JSON: synonyms, acronyms, and likely RRC IE/field names for the feature. Example — "Carrier Aggregation" → `["carrier aggregation", "CA", "SCell", "secondary cell", "PCell", "CellGroupConfig", "sCellToAddModList"]`. Merge in matches from the `abbreviations` table. Cap at 15 terms.
2. **Search.** For each selected document, run an FTS5 `MATCH` with the terms OR-ed together (phrases quoted), ranked by `bm25(spec_sections_fts, 5.0, 1.0, 3.0)` (title and IE names weighted above body). Keep the top **25** per document.
3. **Rank.** One LLM call per layer with section number, title and the first ~800 chars of each candidate; it returns the most relevant section numbers (max **6 per layer**) with a 0–1 relevance score. Sections scoring < 0.4 are dropped.

### 6.3 Release handling

The user gives a **target release** (the release their network software will comply with).

- For each requested spec, choose the document with the **highest release ≤ target**.
- If only newer versions exist, use the oldest available one and add a gap note: *"Only TS 38.214 V19.3.0 is available; sections may include content added after Rel-17."*
- Example: target Rel-17 + TS 38.211 → V16.2.0 is used (V19.2.0 is newer than the target).
- **"Introduced in release"**: the documents alone usually can't prove when a feature first appeared (one version per spec). The LLM is asked for its best answer and the result records `basis: "model_knowledge"` with a note to verify against the 3GPP work-item list. It is `basis: "documents"` only when the feature is present in a later document version and absent in an earlier one in the folder.

### 6.4 Example use case flow

Simple worked example: "Bandwidth Part switching" for a 5G NR target release of Rel-17.

1. `POST /api/summaries` receives:
   ```json
   {
     "feature_name": "Bandwidth Part switching",
     "technology": "5G",
     "target_release": 17,
     "spec_numbers": ["38.214", "38.321", "38.331"],
     "use_cases": ["Power saving for low-traffic UEs"]
   }
   ```
2. The backend expands the query into candidates such as `BWP`, `bandwidth part`, `resource switching`, and `RRC reconfiguration`.
3. It selects the highest available release <= 17 for each document and retrieves the matching subsections from the relevant PDFs.
4. It ranks sections, keeps only the best clause-level matches, and composes a summary that includes citations like `TS 38.214 §5.1.2.2` and `TS 38.331 §6.3.2`.
5. The result is saved and the backend returns a URL or summary ID that the frontend uses to reopen the saved brief from the summary page.

A generated summary page therefore exposes a stable link such as `/summaries/{id}` so the engineer can return to the brief, inspect the references again, or share it with a design review group.

### 6.5 Result JSON (stored in `summaries.result`, returned by the API)

```json
{
  "overview": "2–4 sentence description of the feature from the network-side baseband viewpoint.",
  "release_info": {
    "introduced_in": 15,
    "basis": "model_knowledge",
    "note": "Verify against the 3GPP work item list."
  },
  "documents_used": [
    { "spec_number": "38.214", "version": "19.3.0", "release": 19, "layer": "L1",
      "title": "NR; Physical layer procedures for data" }
  ],
  "layers": [
    {
      "layer": "L1",
      "sections": [
        {
          "spec_number": "38.214",
          "section_number": "5.1.2.2",
          "section_title": "Resource allocation in frequency domain",
          "page": 34,
          "relevance": 0.86,
          "citation": "TS 38.214 §5.1.2.2",
          "summary": "≤ 60 words: what this section defines and why it matters for the feature.",
          "details": "A brief extracted subsection detail that explains the standard rules or procedures relevant to the feature."
        }
      ]
    },
    { "layer": "MAC", "sections": [] },
    { "layer": "RLC", "sections": [] },
    { "layer": "RRC", "sections": [] }
  ],
  "scheduler_impact": {
    "disclaimer": "Engineering analysis generated from the cited sections; not normative 3GPP text.",
    "throughput":  { "direction": "increase", "explanation": "≤ 80 words", "related_sections": ["38.214 5.1.2.2"] },
    "latency":     { "direction": "neutral",  "explanation": "...", "related_sections": [] },
    "ue_capacity": { "direction": "mixed",    "explanation": "...", "related_sections": [] }
  },
  "use_case_notes": [
    { "use_case": "Dense urban eMBB cell", "notes": "≤ 80 words" }
  ],
  "gaps": [
    "No RLC specification (TS 38.322) is in the Standards folder.",
    "No relevant MAC sections found for this feature."
  ]
}
```

- `layers` always contains the four layers in order L1, MAC, RLC, RRC (empty list when nothing found or not requested).
- `direction` ∈ `increase | decrease | neutral | mixed`. ("Increase" latency is bad, "increase" throughput is good — the UI labels them, the backend just reports the direction.)
- `page` is the PDF page in the source document (for the reader to jump to).

### 6.5 LLM usage rules

- Model name, API key and max tokens come from env (`ANTHROPIC_API_KEY`, `LLM_MODEL`, `LLM_MAX_TOKENS`).
- All LLM calls ask for **JSON only** and are parsed with Pydantic; one retry on invalid JSON, then fail the step.
- Prompts include **only retrieved section text**; the LLM is told to cite section numbers from that text and nothing else. `validate.py` drops any section citation not found in `spec_sections` for the used documents.
- Prompts state the out-of-scope rules: network-side view only, no UE implementation advice, no proprietary algorithms.
- Section text sent to the LLM is truncated to fit a ~60k-token budget per call.
- **No API key configured:** fall back to a non-LLM mode — query expansion from the abbreviations table only, BM25 top 4 per layer, section "summary" = first 2 sentences of the section, `scheduler_impact` explanations set to `"Not available without an LLM key."`. Lets the app run and be tested offline.

---

## 7. REST API (contract with the frontend)

Base path `/api`. JSON in/out, UTF-8. Timestamps ISO-8601 UTC. Interactive docs at `/api/docs` (FastAPI).

### 7.1 Conventions

- **Auth:** `access_token` httpOnly cookie (JWT, HS256, 7-day expiry, `Secure` in production; `SameSite=None` when the GitHub Pages frontend and API are cross-site, otherwise `Lax`). Every endpoint except signup/login/health returns `401` without a valid cookie.
- **CSRF:** state-changing requests require header `X-Requested-With: fetch` and an `Origin` matching the configured frontend origin (rejected with `403` otherwise).
- **Errors** — one shape for every non-2xx response:

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Feature name is required.",
             "fields": { "feature_name": "Required" } } }
```

| HTTP | `code` values |
|---|---|
| 400 | `VALIDATION_ERROR` |
| 401 | `NOT_AUTHENTICATED`, `INVALID_CREDENTIALS` |
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `USERNAME_TAKEN`, `SUMMARY_NOT_READY` |
| 429 | `RATE_LIMITED` |
| 500 | `INTERNAL_ERROR` |

(FastAPI's default 422 is remapped to 400 `VALIDATION_ERROR` with the `fields` map.)

### 7.2 Auth

**`POST /api/auth/signup`**
```json
{ "username": "deepu", "password": "S3cure-pass" }
```
- `username`: 3–32 chars, `[a-zA-Z0-9_.-]`, case-insensitive unique.
- `password`: 8–128 chars, at least one letter and one digit.
- `201` → `{ "user": { "id": 1, "username": "deepu", "created_at": "..." } }` and sets the cookie (user is logged in).
- `409 USERNAME_TAKEN`.

**`POST /api/auth/login`** — same body. `200` → `{ "user": {...} }` + cookie. Wrong username or password → `401 INVALID_CREDENTIALS` (same message for both). Rate limit: 10 attempts / 15 min per IP+username → `429`.

**`POST /api/auth/logout`** — clears the cookie. `204`.

**`GET /api/auth/me`** — `200 { "user": {...} }` or `401`. Frontend calls this on load to restore the session.

### 7.3 Spec catalog

**`GET /api/specs?technology=5G`**
```json
{
  "technology": "5G",
  "releases": [15, 16, 17, 18, 19],
  "layers": [
    { "layer": "L1",  "specs": [
        { "spec_number": "38.214", "title": "NR; Physical layer procedures for data",
          "versions": [ { "version": "19.3.0", "release": 19 } ], "available": true } ] },
    { "layer": "RLC", "specs": [
        { "spec_number": "38.322", "title": "NR; Radio Link Control (RLC) protocol specification",
          "versions": [], "available": false } ] }
  ]
}
```
- Lists every spec in the catalog (§4.2) for the technology; `available: false` when no document is ingested.
- `releases`: 5G → 15–19, 4G → 8–19 (configurable).

### 7.4 Summaries

**`POST /api/summaries`** — start generation.
```json
{
  "feature_name": "Bandwidth Part switching",
  "technology": "5G",
  "target_release": 17,
  "spec_numbers": ["38.213", "38.214", "38.321", "38.331"],
  "use_cases": ["Power saving for low-traffic UEs", "Mixed numerology cell"]
}
```
Validation:
- `feature_name`: 2–120 chars, trimmed.
- `technology`: `"4G"` or `"5G"`.
- `target_release`: integer in the release catalog for the submitted `technology` (5G → 15–19; 4G → 8–19 by default; catalogs may be configured). Reject a release outside that technology's catalog with `400 VALIDATION_ERROR` and a `fields.target_release` message; do not silently coerce or substitute a release.
- `spec_numbers`: 1–10 items, each in the catalog for that technology and `available: true`.
- `use_cases`: 0–5 items, each 3–300 chars.
- Limit: max 2 summaries in `queued`/`processing` per user (`429 RATE_LIMITED`).

`202` →
```json
{ "summary": { "id": "6f1c…", "status": "queued", "progress_step": null, "feature_name": "...",
               "technology": "5G", "target_release": 17, "spec_numbers": [...], "use_cases": [...],
               "created_at": "...", "updated_at": "...", "result": null, "error_message": null } }
```

**`GET /api/summaries?limit=20&offset=0&q=bandwidth`** — current user's summaries, newest first, `q` filters on feature name.
```json
{ "items": [ { "id": "...", "feature_name": "...", "technology": "5G", "target_release": 17,
               "status": "ready", "created_at": "..." } ],
  "total": 12 }
```

**`GET /api/summaries/{id}`** — full summary object (as in the 202 response) with `result` filled when `status = ready`. The frontend polls this every 2 s while status is `queued`/`processing`. Another user's id → `404` (never reveal it exists).

**`DELETE /api/summaries/{id}`** — `204`.

**`GET /api/summaries/{id}/pdf`** — `200 application/pdf`, header `Content-Disposition: attachment; filename="3gpp-summary-bandwidth-part-switching-rel17.pdf"`. `409 SUMMARY_NOT_READY` unless status is `ready`. PDF is rendered on first request and cached on disk (`data/pdf/{id}.pdf`).

### 7.5 Health

**`GET /api/health`** — `200 { "status": "ok", "documents_indexed": 8, "llm_configured": true }`. No auth.

---

## 8. PDF export (≤ 5 pages)

Rendered with WeasyPrint from `templates/summary.html`, A4, 18 mm margins, 10 pt body font.

**Layout**

1. **Header block** — title "3GPP Feature Spec Reference: {feature}", technology, target release, generated date, username.
2. **Overview** + "Introduced in" line with its basis note.
3. **Documents used** — compact table: spec, title, version, layer.
4. **Per-layer sections** (L1 → MAC → RLC → RRC) — for each: `TS 38.214 §5.1.2.2 — Title (p. 34)` + summary. Empty layers show one line: "No relevant sections found" or "Specification not available".
5. **Scheduler impact** — 3-row table: KPI | Direction | Explanation | Related sections, with the disclaimer.
6. **Use-case notes** (if any).
7. **Gaps / caveats.**
8. Footer on every page: "Generated by 3GPP Spec Summarizer — verify against the source specifications. Page X of Y".

**5-page enforcement**
1. Render, count pages (`len(document.pages)`).
2. If > 5: drop the lowest-relevance section across all layers (keep at least 1 per non-empty layer), re-render. Repeat.
3. If still > 5 after reaching 1 section per layer: truncate summaries to 40 words, then use-case notes to 40 words.
4. Add a line to the gaps list: "N lower-relevance sections omitted to fit 5 pages; see the web view for the full list."

The web view always shows the full result; only the PDF is trimmed.

---

## 9. Security & non-functional requirements

- Passwords: bcrypt (cost 12) — never logged or returned.
- `JWT_SECRET` from env; app refuses to start in production mode without it.
- All summary queries filtered by `user_id` from the token.
- Input lengths enforced in Pydantic; feature name and use cases are escaped in the PDF template (Jinja2 autoescape on).
- CORS: allow `http://localhost:5173` in dev and the exact deployed GitHub Pages/custom-domain frontend origin in production; enable credentials and never use wildcard origins. The Pages frontend and API are separate origins.
- Deployment: publish the Vite static build to GitHub Pages using GitHub Actions; deploy FastAPI separately on a public HTTPS host. Configure the Pages URL in backend `CORS_ORIGINS` and configure the backend URL in the frontend `VITE_API_BASE_URL`. Prefer a Pages custom domain and API subdomain under the same registrable domain so browsers treat the secure auth cookie as same-site. Confirm that the selected backend host provides persistent storage for SQLite and the ingested Standards index.
- Logs: request id, user id, summary id, step timings; no section text or passwords.
- Performance: list/detail endpoints < 200 ms; PDF render < 5 s; generation < 90 s typical.
- Single-process deployment; SQLite in WAL mode.

---

## 10. Configuration (env vars)

| Variable | Default | Purpose |
|---|---|---|
| `APP_ENV` | `dev` | `dev` / `prod` |
| `DATABASE_URL` | `sqlite:///data/app.db` | |
| `STANDARDS_DIR` | `./Standards` | Input PDFs |
| `JWT_SECRET` | (required in prod) | Cookie signing |
| `JWT_EXPIRY_DAYS` | `7` | |
| `ANTHROPIC_API_KEY` | (empty → fallback mode) | LLM access |
| `LLM_MODEL` | set at deploy time | Model name |
| `LLM_MAX_TOKENS` | `4000` | |
| `CORS_ORIGINS` | `http://localhost:5173` | Comma-separated exact frontend origins; include the production GitHub Pages/custom-domain origin |

---

## 11. Testing

- **Unit:** file-name parser; catalog mapping; section splitter on a small fixture PDF; release selection (§6.3 incl. the 38.211 V16/V19 case); 5-page trimming logic; citation validator.
- **API:** signup/login/logout/me flows; validation errors return the shared error shape; user A cannot read user B's summary (404); PDF returns 409 before ready.
- **Pipeline:** run with a mocked LLM client returning fixed JSON; run fallback mode with no key.
- **Smoke (manual):** ingest the real `Standards/` folder; generate summaries for "Bandwidth Part", "Carrier Aggregation", "DRX"; confirm cited section numbers exist in the PDFs and the export is ≤ 5 pages.

---

## 12. Build order (fits the 2-weekend timeline)

| When | Work |
|---|---|
| Weekend 1, day 1 | Project skeleton, config, DB models, auth endpoints + tests |
| Weekend 1, day 2 | Ingestion CLI (bookmarks → sections → FTS5), `GET /api/specs`, fallback (no-LLM) search |
| Weekend 2, day 1 | Summary pipeline with LLM, result JSON, summaries endpoints, polling |
| Weekend 2, day 2 | PDF template + 5-page enforcement, integration with frontend, smoke tests |

---

## 13. Open questions / issues found in the goal spec

1. **No 4G documents** in `Standards/` — 4G can be selected in the UI but every spec will show "not available" until 36-series PDFs are added. Confirm whether 4G is needed for v1.
2. **No RLC spec (TS 38.322)** in `Standards/`, though the goal spec lists RLC as a required layer.
3. **TS 38.201 is V15.0.0** and TS 38.321 is V18.5.0 while most others are V19 — fine under the release rules in §6.3, but older/newer than some targets.
4. `goal_spec_3gpp.md` is saved in **RTF format** despite the `.md` extension; worth re-saving as plain text so other tools read it cleanly.
5. Glossary row "Itinerary Export" looks left over from a template; this spec uses **Summary Export** (the ≤ 5-page PDF) instead.
