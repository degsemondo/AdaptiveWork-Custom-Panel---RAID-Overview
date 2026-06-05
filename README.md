# AdaptiveWork RAID Panel

A professional custom panel for AdaptiveWork that displays Risks, Issues, and Requests (RAID) in a three-tab interface with inline editing, search, filtering, and action item tracking.

## Version

**Current Stable Version: v2.3** (Layout, Change Requests & Excel round-trip)

## Features

### Core Functionality
- **Three-Tab Interface**: Manage Risks, Issues, and Requests in separate organized tabs
- **Inline Editing**: Click any cell to edit with field-specific input types (text, date, select dropdowns)
- **Heat Map Visualization**: Risk ratings displayed with color-coded heat map (green to red)
- **Action Item Tracking**: Expandable rows showing associated action items for each record
- **Search & Filter**: Full-text search and status-based filtering per tab
- **Professional UI**: Planview-inspired design with burgundy color scheme

### Supported Fields
- **Risks**: Name, Impact, Likelihood, Risk Rating, Status, Owner, Due Date
- **Issues**: Name, Priority, Status, Owner, Due Date, Description
- **Requests**: Name, Type, Priority, Status, Owner, Due Date

### Data Integration
- Uses AdaptiveWork REST API v2.0 with CZQL (Clarizen Query Language)
- Session-based authentication
- Automatic API host mapping for different deployments (app, app2, eu1)
- Proper picklist field extraction with fallback handling

## Installation

Deploy to your AdaptiveWork instance as a custom panel on the Project object:

1. **HTML Field**: Copy contents of `v2.3/html.md` into the HTML field
2. **CSS Field**: Copy contents of `v2.3/css.md` into the CSS field
3. **Script Field**: Copy contents of `v2.3/script.md` into the Script field
4. **Data Field**: Copy contents of `v2.3/data.md` into the Data field (JSON)

## Technical Details

### Architecture
- Vanilla JavaScript (ES5 compatible) with no external dependencies
- Event delegation pattern for efficient DOM handling
- Asynchronous API calls with Promise-based wrappers
- Data normalization functions for type conversion

### API Endpoints
- **Query Endpoint**: `/v2.0/search` (CZQL queries)
- **Update Endpoint**: `window.API.Objects.update()` (inline editing)

### Key Components
- `czql()` - Async CZQL query wrapper with proper headers
- `extractPicklistValue()` - Robust picklist field extraction with 10+ fallback patterns
- `toRisk/toIssue/toRequest()` - Data normalization functions
- `renderRisks/renderIssues/renderRequests()` - Dynamic table rendering
- `startEdit/saveEdit/cancelEdit()` - Inline editing system
- `filterTable/filterByStatus()` - Search and filter logic

## Version History

### v2.3 (2026-06-05)
- **New**: Single-row header (title removed; tabs + shared toolbar on one line)
- **New**: Change Requests tab (filtered to `RequestType = Change Request`); Type column removed
- **New**: ID column on all tabs; **Import from Excel** to update/create records from an exported workbook

### v2.2 (2026-06-03)
- **Reworked Risks tab**: ID, Title, Probability, Impact, Score, Status, Owner, Reporting Level, Impact Date
- **New**: Reporting Level picklist; picklist option paths resolved via metadata (no prefix guessing)
- Status uses `State`, Owner uses `Owner`, date column is `C_ImpactDate` (Issues/Requests unchanged)

### v2.1 (2026-06-03)
- **New**: Inline "new item" draft row for creating Risks/Issues/Requests (no pop-up)
- **New**: `RelatedWork` link created on add so new items show in the project view
- **New**: Owner field is a server-side **type-ahead** of system users
- **New**: Export all tabs (with action items) to a multi-sheet Excel workbook
- **Enhancement**: Canonical, ordered Impact/Probability dropdowns; "Likelihood" → "Probability"
- **Enhancement**: Clean picklist labels with raw-path round-tripping

### v2.0 (2026-06)
- Expanded Risk model (Impact, Probability, calculated Risk Rating with heat map)
- Action items loaded per record; inline editing; REST `objects` create/update

### v1.1 (2026-05-28)
- **Fixed**: Status field displaying "[object Object]" 
- **Enhancement**: Improved picklist field extraction with multiple fallback patterns
- **Result**: All picklist fields (Status, Priority, Impact, Likelihood, etc.) now display correctly

### v1.0
- Initial release with full RAID panel functionality

## Browser Support
- Modern browsers with ES6 support
- Requires AdaptiveWork environment with window.API access

## License
Internal use only for AdaptiveWork deployments
