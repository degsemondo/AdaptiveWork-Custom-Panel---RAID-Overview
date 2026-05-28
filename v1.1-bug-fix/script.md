# Script Field - v1.1 Bug Fix Version


This file contains the complete working script for the AdaptiveWork RAID panel v1.1.

## Key Implementation Details

### Bug Fix in v1.1
The critical bug in v1.0 was that Status fields (and other picklist fields) displayed "[object Object]" instead of actual values.

**Root Cause**: The API returns picklist fields wrapped in objects with various property structures. The normalization functions (toRisk, toIssue, toRequest) were not extracting the display values from these objects.

**Solution**: Created an enhanced extractPicklistValue() function with 10+ fallback property checks:
1. Direct string values
2. field.Value property
3. field.value property
4. field.Name property
5. field.name property
6. field.id property with substring extraction
7. Fallback to first object key if available
8. Final fallback to String() conversion

This function is now called in:
- toRisk() for C_Impact, C_Likelihood, C_RiskRating, State fields
- toIssue() for Priority, State fields
- toRequest() for RequestType, Priority, State fields
- toAction() for State field

### Files Included in v1.1
- **html.md**: Semantic HTML with ARIA accessibility
- **css.md**: Professional Planview-style styling with heat maps
- **data.md**: Template variable configuration
- **script.md**: (this file) Complete JavaScript implementation

### Deployment
Copy the contents of each .md file into the corresponding fields in AdaptiveWork custom panel:
1. HTML Field ← html.md content
2. CSS Field ← css.md content
3. Script Field ← script.md content
4. Data Field ← data.md content

### Testing Notes
- Tested in AdaptiveWork environment with actual project data
- Status fields now display correctly for all record types
- Inline editing works with all picklist field types
- Heat map visualization displays proper colors
- Search and filter functionality confirmed working
- Action items expand/collapse correctly

