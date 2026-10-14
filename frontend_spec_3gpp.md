# Frontend Spec — 3GPP Feature Spec-Reference Summarizer

> Derived from `goal_spec_3gpp.md`. Defines **how** the browser app works: stack, screens, components, state, validation and responsive behaviour. All data comes from the REST API defined in `backend_spec_3gpp.md` §7 — this document does not redefine it; field names below match it exactly.

---

## 1. Scope

A responsive single-page web app (desktop + mobile browsers, no native app) that lets a developer:

1. Sign up and log in with username + password.
2. Enter a 3GPP feature name, technology (4G/5G), target release, the specs to search and target use cases.
3. Watch the summary being generated.
3. Read the summary: per-layer subsection references and clause details (L1, MAC, RLC, RRC), scheduler impact on throughput / latency / UE capacity, and use-case notes.
4. View exact citations such as `TS 38.214 §5.1.2.2 — Resource allocation in frequency domain` and the brief extracted details tied to the feature.
5. Export the summary as a PDF (≤ 5 pages, produced by the backend).
6. Find and reopen summaries they made earlier.

Success metrics from the goal spec map to screens: log in (§4.1–4.2), enter the feature (§4.4), see the summary (§4.5), export PDF (§4.5 export button).

---

## 2. Technology choices

| Concern | Choice | Why |
|---|---|---|
| Framework | **React 18 + TypeScript**, built with **Vite** | Fast setup, typed API models |
| Routing | React Router v6 | |
| Server state | **TanStack Query** | Caching, polling while a summary generates, loading/error states |
| Forms | react-hook-form + **zod** | Validation rules mirror the backend in one schema file |
| Styling | **Tailwind CSS** | Responsive utilities; no custom CSS framework to maintain |
| UI primitives | Headless UI (dialogs, listbox, disclosure) | Accessible without a heavy component library |
| Icons | lucide-react | |
| Tests | Vitest + React Testing Library; Playwright for 1–2 end-to-end flows | |

No global state library is needed — server data lives in TanStack Query, the logged-in user in a small React context.

---

## 3. App structure

### 3.1 Routes

| Path | Screen | Auth |
|---|---|---|
| `/login` | Log in | Public (redirect to `/summaries` if already logged in) |
| `/signup` | Sign up | Public (same redirect) |
| `/summaries` | My summaries (home) | Required |
| `/summaries/new` | New summary form | Required |
| `/summaries/new?from={id}` | New summary prefilled from an earlier one ("Edit & regenerate") | Required |
| `/summaries/:id` | Summary view (progress → result) | Required |
| `*` | Not found | — |

`/` redirects to `/summaries`. Protected routes redirect to `/login?next=<path>` when `GET /api/auth/me` returns 401; after login the user returns to `next`.

### 3.2 Folder layout

```
frontend/
  src/
    main.tsx, App.tsx, routes.tsx
    api/
      client.ts          # fetch wrapper (§6)
      types.ts           # TS types matching backend §6.4 / §7
      auth.ts, specs.ts, summaries.ts   # query + mutation hooks
    auth/AuthContext.tsx, RequireAuth.tsx
    schemas/             # zod schemas (§5)
    pages/               # LoginPage, SignupPage, SummariesPage, NewSummaryPage, SummaryPage, NotFoundPage
    components/
      layout/            # AppShell, TopBar, MobileMenu
      form/              # TextField, PasswordField, SegmentedControl, Select, SpecPicker, UseCaseList
      summary/           # ProgressPanel, OverviewCard, LayerSection, SectionCard, KpiImpactTable, UseCaseNotes, GapsList, DocumentsUsed
      common/            # Button, Badge, EmptyState, ErrorBanner, ConfirmDialog, Toast, Skeleton
```

---

## 4. Screens

### 4.1 Log in (`/login`)

- Centered card (max-width 400 px), app name + one-line purpose: "Collect 3GPP spec references for a feature into one PDF."
- Fields: **Username**, **Password** (with show/hide toggle).
- Primary button **Log in**; link "No account? Sign up".
- Submit → `POST /api/auth/login`. On `200`: store user in AuthContext, go to `next` or `/summaries`.
- On `401 INVALID_CREDENTIALS`: banner "Incorrect username or password." Keep username, clear password.
- On `429`: banner "Too many attempts. Try again in a few minutes."
- Button shows spinner and is disabled while submitting.

### 4.2 Sign up (`/signup`)

- Same card layout. Fields: **Username**, **Password**, **Confirm password**.
- Inline helper text under fields showing the rules (§5.1); errors appear on blur and on submit.
- Submit → `POST /api/auth/signup`. On `201` the user is logged in → `/summaries`.
- `409 USERNAME_TAKEN` → field error on Username: "That username is taken."

### 4.3 My summaries (`/summaries`)

- Top bar (all authenticated screens): app name (links home), **Generate spec Summary** button, user menu (username, **Log out**).
- Search box filtering by feature name (debounced 300 ms → `GET /api/summaries?q=`).
- List of summaries, newest first:
  - Desktop (≥ 768 px): table — Feature | Technology | Release | Status | Created | actions (Open, Download PDF, Delete).
  - Mobile: stacked cards with the same information; actions in a "⋯" menu.
- Status badge: `queued`/`processing` = blue "Generating…", `ready` = green "Ready", `failed` = red "Failed".
- Download PDF only enabled for `ready`.
- Delete → confirm dialog → `DELETE /api/summaries/{id}` → toast "Summary deleted".
- Pagination: "Load more" button (20 per page, `limit`/`offset`).
- **Empty state** (no summaries): short explanation + **Create your first summary** button.

### 4.4 New summary (`/summaries/new`)

One-page form, single column on mobile, max-width 720 px on desktop. Field order follows the goal-spec use case ("release version, technology area and the name of the feature"). The workspace heading is **RAN Feature mapping to 3GPP spec**.

| # | Field | Control | Notes |
|---|---|---|---|
| 1 | **Technology** | Segmented control: `5G NR` / `4G LTE` (default 5G) | Changing it reloads `GET /api/specs?technology=…`, replaces both release options and spec list with that technology's catalog, and resets release to that catalog's latest release + spec selection |
| 2 | **Target 3GPP release** | Select only from the `releases` returned for the currently selected technology by `GET /api/specs?technology=…` (shown as "Rel-17") | Default: latest release in that technology's catalog; never retain or submit a release from the previously selected technology |
| 3 | **Feature name** | Text input, placeholder "e.g. Bandwidth Part switching, Carrier Aggregation, DRX" | Autofocus |
| 4 | **Specifications to search** | `SpecPicker` (below) | At least 1 |
| 5 | **Target use cases** (optional) | `UseCaseList`: add up to 5 text items | e.g. "Power saving for low-traffic UEs" |

**SpecPicker**
- Grouped by layer in fixed order **L1 · MAC · RLC · RRC**, data from `GET /api/specs?technology=…`.
- Each spec is a checkbox row: `TS 38.214 — NR; Physical layer procedures for data` with version chip(s) (e.g. `V19.3.0`).
- Specs with `available: false` are shown disabled with the note "Not in Standards folder".
- Group header has a "Select all" toggle for that layer. By default every available spec is checked.
- If a whole layer has no available spec, show an inline info note: "No RLC specification is available — this layer will be empty in the summary."
- **Release hint**: if every version of a checked spec is newer than the target release, show a small warning under it: "Only V19.3.0 is available; it may include content newer than Rel-17." (Logic matches backend §6.3; the backend also reports it in `gaps`.)

**UseCaseList**
- Text input + **Add** button (Enter also adds). Items render as removable chips/rows. Counter "2 / 5".

**Actions**
- Primary **Generate summary** → `POST /api/summaries` → on `202` navigate to `/summaries/{id}`.
- Secondary **Cancel** → back to list (confirm if the form is dirty).
- Server `400` field errors are mapped onto the matching fields; `429` → banner "You already have 2 summaries generating. Wait for one to finish."

**Prefill (`?from={id}`)**: load `GET /api/summaries/{id}` and fill all fields with its inputs; heading becomes "Edit inputs & regenerate". Submitting creates a new summary.

### 4.5 Summary view (`/summaries/:id`)

Data: `GET /api/summaries/{id}`. While `status` is `queued` or `processing`, TanStack Query refetches every **2 s**; polling stops on `ready`/`failed` or after 4 minutes (then show "This is taking longer than expected — refresh to check again").

**Example flow**
- A user enters "Bandwidth Part switching" as the feature name under 5G NR Rel-17.
- The app identifies the relevant sections in TS 38.214 and TS 38.331, then sends the user to the summary page.
- The summary page opens with a stable link such as `/summaries/{id}` and displays the section citations, use-case notes, and a reusable summary URL for sharing.

**Header (always shown)**
- Feature name (H1); chips: `5G NR`, `Target Rel-17`, created date.
- Inputs summary (collapsible on mobile): specs searched, use cases.
- Actions: **Export PDF** (primary, enabled only when ready), **Copy summary link** (copies the current summary URL), **Edit & regenerate** (→ `/summaries/new?from=id`), **Delete**.

**A. Generating state** — `ProgressPanel`
- Step list from `progress_step` (backend §6.1), with check marks for completed steps and a spinner on the current one:
  Selecting documents → Building search terms → Searching specifications → Ranking sections → Writing summary → Checking references.
- Note: "This usually takes under a minute. You can leave this page — the summary will be in your list."

**B. Failed state**
- Red banner with `error_message`, buttons **Try again** (re-submit same inputs via `POST /api/summaries`) and **Edit inputs**.

**C. Ready state** — sections in this order:

1. **Overview card** — `result.overview`; line "Introduced in: Rel-15" with a tooltip/note showing `release_info.note`, and a small label "From AI knowledge — verify" when `basis = model_knowledge`, or "From documents" when `basis = documents`. `introduced_in: null` → "Unknown".
2. **Documents used** — compact list of `documents_used` (spec, version, layer).
3. **Layer sections** — one block per layer (L1, MAC, RLC, RRC):
   - Desktop: a sticky left-side mini nav (L1 · MAC · RLC · RRC · KPI impact · Use cases) that jumps to blocks.
   - Mobile: the mini nav becomes a horizontally scrollable tab strip at the top.
  - Each `SectionCard`: a clickable `TS 38.214 §5.1.2.2` citation opening the backend-provided `source_url` at the exact source PDF page, section title, page, relevance bar (0–1 shown as a thin bar, not a number), extracted subsection detail, and summary text. Show each backend-provided functional subcategory (CUCP, CUUP, RRM, Layer 2, Layer 1) with its evidence-based `details`. A copy button copies `TS 38.214 §5.1.2.2 — Resource allocation in frequency domain`.
   - Empty layer: muted line "No relevant sections found in the selected specifications" (or the matching `gaps` message).
4. **Scheduler impact (KPI table)** — rows Throughput, Latency, UE capacity; columns Direction | Explanation | Related sections.
   - Direction shown as icon + word, coloured by whether it is good or bad for that KPI: throughput/UE capacity "increase" = green, "decrease" = red; latency "increase" = red, "decrease" = green; neutral = grey; mixed = amber. Never rely on colour alone (always show the word).
   - `scheduler_impact.disclaimer` shown in small text above the table.
   - Mobile: table becomes three stacked cards.
5. **Use-case notes** — one card per item (hidden if empty).
6. **Gaps & caveats** — bulleted `gaps`, amber info style.

**Shareable summary link**
- A link button appears beside the export button and copies the current summary URL to the clipboard: `/summaries/{id}`.
- The app also shows a text link under the header so the user can reload or share the exact brief page.
- The URL should remain valid even after the user leaves and returns to the summaries list.

**Export PDF**
- Calls `GET /api/summaries/{id}/pdf` as a blob, triggers a download using the filename from `Content-Disposition`.
- Button shows "Preparing PDF…" while waiting (first render can take a few seconds).
- Note under the button: "PDF is limited to 5 pages; lower-relevance sections may be left out."
- On mobile browsers that block blob downloads, fall back to opening the PDF URL in a new tab (cookie auth works since it's same-origin).

### 4.6 Not found / errors

- Unknown route or `404` summary → "We couldn't find that page" + link to My summaries.
- Global `ErrorBanner` for network failures: "Can't reach the server. Check your connection and retry." with **Retry**.

---

## 5. Validation (zod schemas — mirror backend §7)

### 5.1 Auth
| Field | Rule | Message |
|---|---|---|
| username | 3–32 chars, `^[a-zA-Z0-9_.-]+$` | "3–32 characters: letters, numbers, _ . -" |
| password | 8–128 chars, ≥ 1 letter and ≥ 1 digit | "At least 8 characters with a letter and a number" |
| confirmPassword (signup) | equals password | "Passwords don't match" |

### 5.2 New summary
| Field | Rule |
|---|---|
| feature_name | trimmed, 2–120 chars |
| technology | `"4G"` \| `"5G"` |
| target_release | integer in the `releases` catalog returned for the selected `technology`; changing technology must reset/revalidate this value before submission |
| spec_numbers | 1–10 items, all `available` for the chosen technology |
| use_cases | 0–5 items, each trimmed 3–300 chars, no duplicates |

Client validation is for fast feedback only; the backend's `400` `fields` map is always applied as the final word.

---

## 6. API client

- `client.ts` wraps `fetch`:
  - Base URL comes from the build-time `VITE_API_BASE_URL`; local development defaults to `/api` (Vite proxy forwards to `http://localhost:8000`), while the GitHub Pages production build uses the public HTTPS API URL.
  - `credentials: "include"`, `Content-Type: application/json`, and header `X-Requested-With: fetch` on every request (backend CSRF rule).
  - Parses the shared error shape `{ error: { code, message, fields } }` into a typed `ApiError`.
  - On any `401` from a protected call: clear AuthContext, invalidate queries, redirect to `/login?next=…`.
- `types.ts` defines `User`, `SpecCatalog`, `SummaryInput`, `SummaryListItem`, `Summary`, `SummaryResult`, `LayerBlock`, `SectionRef` (including `source_url` and `{ name, details }` subcategories), `KpiImpact`, matching backend §6.4 / §7 exactly.

**Query keys**

| Key | Endpoint | Notes |
|---|---|---|
| `["me"]` | `GET /auth/me` | Run once at app start; `retry: false` |
| `["specs", technology]` | `GET /specs` | `staleTime: Infinity` (changes only on re-ingest) |
| `["summaries", q]` | `GET /summaries` | Infinite query for "Load more" |
| `["summary", id]` | `GET /summaries/{id}` | `refetchInterval: 2000` while generating |

Mutations (`createSummary`, `deleteSummary`, `login`, `signup`, `logout`) invalidate `["summaries"]` / `["me"]` as appropriate.

---

## 7. Responsive design

Mobile-first with Tailwind breakpoints:

| Breakpoint | Layout |
|---|---|
| < 640 px (phone) | Single column, 16 px side padding; top bar collapses actions into a menu; list as cards; KPI table as cards; layer nav as scrollable tabs; sticky bottom bar on Summary view with **Export PDF** |
| 640–1023 px (tablet) | Single column, wider cards; table layout for the summaries list from 768 px |
| ≥ 1024 px (desktop) | Content max-width 1120 px; Summary view gets a 220 px sticky left nav + content column |

- Tap targets ≥ 44 × 44 px; no hover-only actions.
- No horizontal page scrolling at 320 px width; long spec IDs/IE names wrap (`break-words`).
- Test on Chrome (desktop + Android) and Safari (macOS + iOS).

---

## 8. Visual style

- Clean, technical, document-like: neutral greys, one accent colour (blue) for primary actions, monospace for spec/section numbers.
- Typography: system font stack for UI; `ui-monospace` for spec references.
- Layer colour tags (consistent in web and PDF): L1 indigo, MAC teal, RLC amber, RRC violet — always with the text label.
- Light theme required; dark theme via `prefers-color-scheme` is optional (cheap with Tailwind `dark:`).

---

## 9. Accessibility

- WCAG 2.1 AA contrast; visible focus rings; full keyboard use (form, spec picker, dialogs).
- Labels on every input; errors linked with `aria-describedby`; error banners use `role="alert"`.
- Progress panel uses `aria-live="polite"` to announce step changes.
- Status and KPI direction always conveyed by text, not colour only.

---

## 10. Testing

- **Unit/component (Vitest + RTL):** zod schemas; SpecPicker (disabled specs, select-all, release warning); UseCaseList limits; KPI colour mapping per KPI; ProgressPanel step rendering; error-shape mapping onto form fields.
- **API mocking:** MSW (Mock Service Worker) fixtures for every endpoint, including a summary that moves queued → processing → ready.
- **End-to-end (Playwright, against the real backend in fallback/no-LLM mode):**
  1. Sign up → create summary → wait for Ready → see layer sections → export PDF (file downloaded).
  2. Log out → protected URL redirects to login → log in returns to that URL.
- Run E2E once at a 390 × 844 mobile viewport as well as desktop.
- Verify the production build under the GitHub Pages repository base path, including direct navigation and refresh on `/summaries` and `/summaries/:id`, API calls to the configured HTTPS backend, and authentication in Chrome and Safari.

---

## 11. Build & deploy

- `npm run dev` (Vite on :5173, proxy `/api` → :8000).
- `npm run build` → `frontend/dist`; set Vite's base path from the GitHub repository name so assets and routes work at `https://<owner>.github.io/<repository>/`.
- Publish `frontend/dist` to GitHub Pages with a GitHub Actions workflow on pushes to the deployment branch. Include a Pages-compatible `404.html` SPA fallback so direct route loads and refreshes resolve to the app.
- Configure `VITE_API_BASE_URL` as a GitHub Actions build variable pointing to the separately deployed public HTTPS backend. The API URL is public configuration, not a secret; never place API keys or backend credentials in the frontend bundle.
- The backend must be deployed separately and remain publicly reachable over HTTPS. Production sign-in requires credentialed CORS and secure cookies; use a custom domain for the Pages site and an API subdomain under the same registrable domain where possible to avoid browser third-party-cookie restrictions.

---

## 12. Build order (fits the 2-weekend timeline)

| When | Work |
|---|---|
| Weekend 1, day 1 | Vite + Tailwind + Router skeleton, API client, AuthContext, Login/Signup pages |
| Weekend 1, day 2 | My summaries page, New summary form with SpecPicker (against MSW mocks) |
| Weekend 2, day 1 | Summary view: progress polling, layer sections, KPI table, gaps |
| Weekend 2, day 2 | PDF export, responsive polish on phone, Playwright flows against real backend |
