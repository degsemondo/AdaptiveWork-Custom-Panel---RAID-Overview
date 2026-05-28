# Version History

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
