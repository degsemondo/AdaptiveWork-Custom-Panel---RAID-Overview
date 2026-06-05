# Version History

## v2.3 - Layout, Change Requests & Excel Round-Trip (2026-06-05)

### What's New
- **Single-row header** — removed the "Project risks, issues & requests" title
  and merged the tab bar with a shared toolbar (search, status filter, New) on
  one row. The toolbar is contextual: switching tabs updates the search
  placeholder, status options, and New-button label, all acting on the active tab.
- **Change Requests tab** — the Requests tab is relabelled "Change Requests" and
  shows only `EnhancementRequest` records where `RequestType = Change Request`.
  New items created here are set to that type automatically; the redundant Type
  column was removed.
- **ID column** on all three tabs (`SYSID`) — shown in the table, export, and used
  as the match key for import.
- **Import from Excel** — an "Import" button reads a workbook produced by Export
  (parsed in-browser via `DOMParser`, no library), matches rows by ID, updates
  changed cells, and creates rows with no ID (with the `RelatedWork` project
  link; Change Requests get their type set). Picklists match by label, owner by
  name, dates by parsing the shown value; read-only columns and empty cells are
  ignored. A confirm summary (update/create counts + warnings) is shown before
  applying.

### Files
- `script.md`, `html.md` (CSS and Data unchanged from v2.2)

### Notes
- Re-export from v2.3 before editing/importing — earlier exports lack the ID
  columns on Issues/Change Requests.

---

## v2.2 - Risks Field Rework (2026-06-03)

Reworks the Risks tab to a new field specification (Risks only; Issues and
Requests are unchanged from v2.1).

### What's New
- **New Risks columns:** ID (`SYSID`), Title, Probability (`C_Likelihood`),
  Impact (`C_Impact`), Score (`C_RiskRating`, auto-calculated, heat-mapped),
  Status (`State`), Owner (`Owner`, user type-ahead), Reporting Level
  (`C_ReportingLevel`), Impact Date (`C_ImpactDate`).
- **Reporting Level** picklist (EFDC / Portfolio / Project Board / Project Team),
  editable on existing rows and the new-item draft row.
- Status now reads/writes the standard `State` field; Owner is the `Owner`
  reference (was `AssignedTo` in v2.1); the date column is `C_ImpactDate`.
- **Picklist metadata loader** — fetches real option paths via `describeEntities`
  so saved picklist values are valid without prefix guessing; falls back to
  data-derived options if unavailable.
- Picklist type prefixes follow the *defining* entity, not the field name:
  Impact/Probability are defined on Risk (`/C_RiskImpact`, `/C_RiskProbability`);
  Reporting Level is defined on the Case ancestor (`/C_CaseReportingLevel`).
- Excel export and inline create/edit updated to match the new Risks columns.

### Files
- `script.md`, `html.md` (CSS and Data unchanged from v2.1)

---

## v2.1 - RAID Editing & Creation Release (2026-06-03)

Builds on v2.0 with full create/update support, cleaner picklists, and a
user-friendly Owner type-ahead.

### What's New
- **Create via inline draft row** — clicking "New risk/issue/request" inserts an
  editable row at the top of the table (name + picklist dropdowns + due date,
  Save/Cancel, Enter to save / Esc to cancel). Replaces the old `prompt()` pop-up.
- **Project linking** — after creating a Risk/Issue/Request, a `RelatedWork`
  record is created (`Case` = new item id, `WorkItem` = project id) so the new
  item appears in the project's RAID view.
- **Canonical Impact / Probability options** — the Impact (`C_Impact`) and
  Probability (`C_Likelihood`) dropdowns always show the full ordered option list
  (1 - Minor … 6 - Severe / 1 - Unlikely … 6 - Very Likely), not just values
  already in use. The "Likelihood" column header was renamed to "Probability".
- **Owner type-ahead** — Owner (`AssignedTo`) is now an editable, debounced
  type-ahead that searches system users server-side (`User WHERE Name LIKE
  '%term%'`, 2+ chars) rather than bulk-loading everyone. Works on existing rows
  and the new-item draft row; sends the `/User/…` id.
- **Clean picklist labels** — `/Type/Value` paths are shown as their value
  (e.g. `New` instead of `/CaseState/New`) while the raw path is preserved for
  round-tripping to the API.
- **Export to Excel** — an "Export" button in the header produces a styled,
  multi-sheet workbook (one sheet per tab: Risks / Issues / Requests) with each
  record's action items listed beneath it. Formatting: bold white-on-burgundy
  headers, bordered grid, Risk Rating cells filled with the on-screen heat-map
  colours, italic-grey action rows, frozen header row. Generated entirely
  in-browser (SpreadsheetML 2003) with no external library or CDN, so it works
  in locked-down environments where external scripts are blocked.

### API / Endpoints
- Create: `PUT /V2.0/services/data/objects/{EntityType}` → `{ id: "/Risk/…" }`
- Update: `POST /V2.0/services/data/objects/{EntityType}/{id}`
- Reference/picklist values sent as the `/Type/Value` path format.

### Files Modified
- `script.md`, `css.md`, `html.md`

---

## v2.0 - RAID Panel with Expanded Risk Model (2026-06)

### What's New
- Expanded Risk model: separate **Impact**, **Likelihood**, and calculated
  **Risk Rating** columns with heat-map colouring (Risk Rating is read-only;
  editing Impact or Likelihood reloads so the recomputed rating shows).
- Two-phase data load: Risks/Issues/Requests, then their Action Items
  (`WHERE Container IN (...)`).
- Inline editing for picklist, text, and date fields with safe round-tripping
  of raw picklist paths.
- Create/update moved to the documented REST `objects` endpoints (fixing the
  earlier HTTP 500 from the wrong `/upsert` body).

### Files
- `html.md`, `css.md`, `data.md`, `script.md`

---

## v1.1 - Bug Fix Release (2026-05-28)

### Bug Fixed
**Status field displaying "[object Object]" instead of actual values**

#### Root Cause
The `extractPicklistValue()` function was not being called when normalizing data from the API response. Picklist fields like `State`, `Priority`, `C_Impact`, and `C_Likelihood` were being returned as wrapped objects from the AdaptiveWork API, which when directly converted to strings displayed as "[object Object]".

#### Solution
1. Enhanced `extractPicklistValue()` function with 10+ fallback property checks:
   - Direct string values
   - `Value` property
   - `value` property
   - `Name` property
   - `name` property
   - `id` property with substring extraction
   - Fallback to first object key if available

2. Integrated `extractPicklistValue()` calls into all data normalization functions:
   - `toRisk()` - applies to C_Impact, C_Likelihood, C_RiskRating, and State fields
   - `toIssue()` - applies to Priority and State fields
   - `toRequest()` - applies to RequestType, Priority, and State fields
   - `toAction()` - applies to State field

#### Testing
- Verified in AdaptiveWork environment with actual project data
- Confirmed inline editing still works with picklist fields
- Validated heat map color visualization displays correct ratings
- Confirmed search and filter functionality works with properly extracted status values

#### Files Modified
- `script.md` - Enhanced extractPicklistValue() and integrated into normalization functions

### What's Included
- 3-tab RAID interface (Risks, Issues, Requests)
- Inline editing support with field-specific input types
- Heat map visualization for Risk Rating
- Search and filter functionality per tab
- Expandable rows for associated action items
- Professional Planview-style UI with burgundy color scheme
- Full AdaptiveWork API integration with CZQL support

---

## v1.0 - Initial Release

### Features
- Three-tab custom panel for AdaptiveWork Project object
- Complete RAID management (Risks, Issues, Enhancement Requests)
- Inline editing for editable fields
- Search and filter by status
- Action items display with expandable rows
- Risk Rating heat map visualization
- Professional UI styling with Tabler icons

### Files
- `html.md` - Semantic HTML with accessibility attributes
- `css.md` - Professional styling with heat map colors
- `data.md` - Template variable configuration
- `script.md` - Full panel functionality

### Known Issues
- Status field displays "[object Object]" for picklist fields (Fixed in v1.1)
