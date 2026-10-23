# SpendNest Revenue Intelligence Inbox

A revenue intelligence workspace built for sales managers to triage call and meeting records, inspect evidence passages, maintain audit-protected manual route corrections, and export machine-readable manifests.

Built for the Volopay hiring assignment (October 2026).

---

## 1. Quick Start

### Prerequisites
- Node.js (v18+ recommended; tested on v22.17)
- npm (v9+; tested on v10.9)

### Running Locally
```bash
# 1. Install dependencies
npm install

# 2. Run unit tests
npm test

# 3. Start local development server
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) in your browser.

The application automatically seeds the completed post-update state on first launch (44 records, 18 usable texts). You can also click **Clean Reset** or **Demo Load** anytime in the **Replay & Imports** console.

### Production Build
```bash
npm run build
npm run preview
```

---

## 2. Replay & Verification Steps (How to Reproduce on Camera)

The app implements all replay behaviors specified in `DATA_CONTRACT.md` and `ASSIGNMENT.md`:

1. **Clean Baseline:**
   - Navigate to **Replay & Imports** tab.
   - Click **Clean Reset**. The app loads `data/01_initial.json`.
   - The dashboard reports **44 records** (30 calls, 14 meetings) and **17 usable texts** (38.6% coverage).
   - `M07` (Willowfield Trust) shows 0 turns ("transcript pending"). Its default route is `Needs Review`.
2. **Idempotent Re-Import:**
   - In **Replay & Imports**, click **Repeat Import (Shuffled)**.
   - The reconciliation report logs: `44 received`, `0 added`, `0 updated`, `44 unchanged`.
   - Total records strictly remains **44**. Headline metrics do not inflate.
3. **Manual Route Correction:**
   - In the **Dashboard & Inbox**, click on `M07`.
   - In the right-hand drawer, select target route: **Customer Retention / Success**.
   - Input justification: *"Customer experiencing recurring UI save bugs in bill pay; escalation required."*
   - Click **Apply Route Correction**. The row badge updates immediately to `CUSTOMER RETENTION` with an `[OVERRIDDEN]` tag.
4. **Late Transcript Update:**
   - In **Replay & Imports**, click **Import 02_update.json**.
   - The reconciliation report logs: `1 received`, `0 added`, `1 updated (M07)`.
   - Total records strictly remains **44**.
   - Usable text coverage increments from **17 to 18** (40.9%).
5. **Preserving Human Judgment:**
   - Open `M07` again. Revision is now **2** and all **27 new turns** are visible.
   - The manual route remains **Customer Retention / Success** and was **not** overwritten by the update batch.
   - Press **Ctrl+R / Cmd+R** to refresh the browser: all state, overrides, and 44 records persist from storage.

---

## 3. Metric Definitions & Formulas

All metrics represent selected records for **21 August 2026 (Asia/Kolkata)**.

| Metric | Formula | Source Fields | Reviewer Verification |
|---|---|---|---|
| **Total Activity** | Count distinct `(source_type, source_id)` | `source_type`, `source_id` | Click tile: lists all 44 records. |
| **Transcript Coverage** | $\frac{\text{Usable Texts}}{\text{Total Activity}} \times 100$ | Calls: `transcript` non-empty.<br>Meetings: `turns.length > 0`. | Initial: $17/44 = 38.6\%$. Post-update: $18/44 = 40.9\%$. Click tile to filter. |
| **Source Connect Rate** | $\frac{\text{Calls with } is\_connected = true}{\text{Calls with non-null } is\_connected} \times 100$ | `is_connected` in call payload | $11 / 30 = 36.7\%$. Denominator is 30 calls. Click tile to view 11 calls. |
| **Substantive Call Rate** | $\frac{\text{Calls with } is\_conversation = true}{\text{Total Calls}} \times 100$ | `is_conversation` in call payload | $9 / 30 = 30.0\%$. Distinguishes raw connects from actual conversations. |
| **Ghost Meetings Alert** | Count of recorded meetings with 0 turns | `recording_status == 'recorded'` and `turns.length == 0` | 3 meetings (`M05, M08, M11`). Click tile to inspect ghost recordings. |
| **Manual Overrides** | Count of records in override layer | `record_overrides` count | Displays active human corrections that re-imports never overwrite. |

---

## 4. Architecture & Design Decisions

### Technical Stack
- **Framework:** React 18 + TypeScript + Vite.
- **Styling:** Custom dense utility CSS (zero external heavyweight component libraries, zero gradient banners, zero stock AI templates).
- **Persistence:** Local browser storage (`localStorage`) adhering strictly to `DATA_CONTRACT.md` L30 (*"Local storage or a local file is sufficient"*). Guarantees instant cold start (< 300ms) on deployed URLs and avoids container disk wipes on free server platforms.
- **Packaging:** Client-side ZIP compilation via `JSZip`.

### Routing Architecture
A deterministic rule engine assigns each record to one of six categories based on participant signals and transcript content:
1. `deal_next_steps`: Commercial buyer evaluation with confirmed next step (`C03, C05, C08, C09, M03, M06`).
2. `sales_coaching`: SDR technique issues, dialer seat misattribution, or cold call objection handling (`C01, C04, C06, C10`).
3. `customer_retention`: Existing customers with bugs, card declines, or setup failures (`M04, M07, M09`).
4. `vendor_partner`: External software suppliers or banking referral partners (`M01, M02, C07`).
5. `internal_training`: Team enablement sessions with zero external attendees (`M10`).
6. `needs_review`: No speech captured, silent recordings, or unclassified records (`B01`–`B20`, `M05, M08, M11`–`M14`).

### Runtime vs Pre-Generated Analysis Disclosure
- All 18 conversation analyses (key takeaways, citations, negative scope, next steps, owners) are pre-computed and stored in code.
- Dynamic runtime logic handles:
  - Envelope validation and quarantine checks.
  - Content hashing (SHA-256) and deduplication.
  - Revision comparison and in-place updates.
  - Rule-based route assignment.
  - Override protection and audit logging.
  - Live metric filtering and drill-down queries.
  - ZIP archive packaging, relative path mapping, and manifest checksum computation.

### Three Most Important Findings
1. **Vendor Mistaken for Sales Prospect (`M02`, Turn 1–2):** LedgerBridge was discussing accounts receivable and electronic invoicing templates. A sales manager relying on keywords would assume this was a customer pipeline deal. In reality, SpendNest was evaluating LedgerBridge as a supplier. Automated routing sends this to `Vendor / Partner`.
2. **21% Activity Overcount from Ghost Recordings (`M05, M08, M11`):** Three meetings were logged as `recorded` with summaries `stored`, but nobody spoke (`turns: []`). A manager tracking completed meetings would overcount activity without checking turns.
3. **Dialer Seat Misattribution (`C06`):** Telephony logs attribute call C06 to Meera, but the audio transcript begins with Ravi introducing himself. Indicates shared dialer seats or unmanaged VoIP licenses.

### One Deliberately Rejected Decision
- **Rejected:** Running an automated LLM call during batch imports to summarize and route records.
- **Why Rejected:** 
  1. API key liabilities: `ASSIGNMENT.md` L86 prohibits API credentials in submissions.
  2. Latency & rate limits: API calls during reviewer evaluation risk timeouts or quota failures.
  3. Hallucination on silent records: LLMs fabricate conversation details when given empty meetings (like `M05` or `M08`).
  4. Override compliance: Re-running automated models risks overwriting manager corrections unless an explicit override layer exists.
- **Accepted Trade-off:** Used a transparent deterministic router with cached analysis and an explicit human override layer.

---

## 5. Exports & Manifest Structure

Clicking **Export Archive (.zip)** in the top navigation generates `spendnest_exports_2026-08-21.zip` containing:
```
spendnest_exports_2026-08-21/
├── manifest.json
├── manifest.csv
├── daily_briefing/
│   └── manager_brief_2026-08-21.md
├── deal_next_steps/
├── sales_coaching/
├── customer_retention/
├── vendor_partner/
├── internal_training/
└── needs_review/
```
Both `manifest.json` and `manifest.csv` include `source_type`, `source_id`, `record_ref`, `applied_revision`, `assigned_routes`, `output_paths`, `processing_state`, `is_manually_overridden`, `override_reason`, `analysis_freshness`, and `checksum_sha256`.

---

## 6. Project Stats & Time Spent

- **Total Time Spent:** Approximately 7 hours.
- **Known Limitations:**
  - Date filtering is locked to 21 August 2026 as all supplied records fall on this single date.
  - Background calls (`B01`–`B20`) have no audio transcripts and remain in intake review states.
- **What I Would Improve Next With More Time:**
  - Implement full-text search across transcripts with keyword highlighting.
  - Add CSV import drag-and-drop for arbitrary custom delivery batches.
  - Build webhook listeners for live telephony connectors.
