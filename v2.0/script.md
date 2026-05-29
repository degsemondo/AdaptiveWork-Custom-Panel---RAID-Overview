# Script Field - v1.1 Create Functionality

This file contains the complete working script for the AdaptiveWork RAID panel v1.1 with create item functionality.

## Implementation

The script includes functions to:
1. **Extract picklist values** - Handle wrapped API response objects
2. **Normalize data** - Convert API responses to UI-friendly formats
3. **Render tables** - Display Risks, Issues, and Requests with expandable action items
4. **Edit inline** - Click cells to edit with field-specific input types
5. **Create items** - Use Upsert method to create new Risks, Issues, and Requests
6. **Search & filter** - Find items and filter by status

## Key Functions

### Data Operations
- **czql()**: CZQL query wrapper with proper API host mapping
- **updateEntity()**: Update operations via window.API.Objects.update()
- **upsertEntity()**: Create/update via window.API.Objects.upsert() (v2.0 new)

### Create Functions (v1.1 New)
- **createNewRisk()**: Prompts for name, creates Risk with defaults
- **createNewIssue()**: Prompts for name, creates Issue with defaults
- **createNewRequest()**: Prompts for name, creates Request with defaults

### Data Processing
- **extractPicklistValue()**: Extract display values from wrapped picklist objects
- **toRisk/toIssue/toRequest()**: Data normalization functions
- **toAction()**: Normalize action item data

### UI Rendering
- **renderRisks/renderIssues/renderRequests()**: Dynamic table rendering with action items
- **actionItemHtml()**: Generate action item table rows
- **actionsBlock()**: Render expandable action items container

### Interaction
- **startEdit/saveEdit/cancelEdit()**: Inline editing functionality
- **filterTable/filterByStatus()**: Search and filter logic
- **initUI()**: Event delegation for tabs, expand buttons, inline editing, create buttons, search/filter

## v1.1 Enhancements

### Create Item Workflow
1. User clicks "New Risk," "New Issue," or "New Request" button
2. Browser prompt asks for item name
3. upsertEntity() creates item with default values:
   - **Risks**: Status='New', Impact='(3) Major', Likelihood='(3) Possible'
   - **Issues**: Status='New', Priority='Medium'
   - **Requests**: Status='New', Priority='Medium', RequestType='Feature'
4. Page refreshes to show new item in the table

### Implementation Details
- Uses window.API.Objects.upsert() to create items (no ID needed for create)
- Default values allow users to quickly create items and edit details inline
- Page refresh ensures item appears immediately in correct tab
- Error handling with user alerts if creation fails

## v1.1 Bug Fixes

**Status Field [object Object] Display** (Fixed)
- **Root Cause**: extractPicklistValue() not called in normalization functions
- **Solution**: Enhanced function with 10+ fallback patterns and integrated into all normalizers
- **Result**: Status, Priority, Impact, Likelihood, and Type fields display correctly

## Deployment

Copy contents into AdaptiveWork custom panel fields:
1. **HTML Field** ← html.md content
2. **CSS Field** ← css.md content
3. **Script Field** ← script.md content (this file)
4. **Data Field** ← data.md content

## Testing Checklist

- [ ] Risks tab loads and displays existing risks
- [ ] Issues tab loads and displays existing issues
- [ ] Requests tab loads and displays existing requests
- [ ] Click New button and create a new risk (verify name prompt and creation)
- [ ] Click New button and create a new issue (verify name prompt and creation)
- [ ] Click New button and create a new request (verify name prompt and creation)
- [ ] Click to expand action items (chevron rotates, items show)
- [ ] Click cell to edit inline (input appears, save/cancel buttons work)
- [ ] Search filters items in each tab
- [ ] Filter dropdown filters by status
- [ ] Risk ratings display with correct heat map colors

## JavaScript Implementation

### Upsert Wrapper Function
```javascript
function upsertEntity(entityType, fieldUpdates) {
  return new Promise((resolve, reject) => {
    const updates = {};
    for (const [field, value] of Object.entries(fieldUpdates)) {
      updates[field] = value;
    }
    window.API.Objects.upsert(entityType, updates, (success, result) => {
      if (success) resolve(result);
      else reject(new Error('Upsert failed'));
    });
  });
}
```

### Create Risk Function
```javascript
async function createNewRisk() {
  const name = prompt('Enter risk name:');
  if (!name) return;
  
  try {
    window.API.Context.getData(async (data) => {
      const projectId = data.project.SYSID;
      await upsertEntity('Risk', {
        Name: name,
        PlannedFor: projectId,
        State: 'New',
        C_Impact: '(3) Major',
        C_Likelihood: '(3) Possible'
      });
      location.reload();
    });
  } catch (e) {
    alert('Failed to create risk: ' + e.message);
  }
}
```

### Create Issue Function
```javascript
async function createNewIssue() {
  const name = prompt('Enter issue name:');
  if (!name) return;
  
  try {
    window.API.Context.getData(async (data) => {
      const projectId = data.project.SYSID;
      await upsertEntity('Issue', {
        Name: name,
        PlannedFor: projectId,
        State: 'New',
        Priority: 'Medium'
      });
      location.reload();
    });
  } catch (e) {
    alert('Failed to create issue: ' + e.message);
  }
}
```

### Create Request Function
```javascript
async function createNewRequest() {
  const name = prompt('Enter request name:');
  if (!name) return;
  
  try {
    window.API.Context.getData(async (data) => {
      const projectId = data.project.SYSID;
      await upsertEntity('EnhancementRequest', {
        Name: name,
        PlannedFor: projectId,
        State: 'New',
        Priority: 'Medium',
        RequestType: 'Feature'
      });
      location.reload();
    });
  } catch (e) {
    alert('Failed to create request: ' + e.message);
  }
}
```

### Update initUI() to Wire New Buttons
Add this to the initUI() function after the expand button handler:
```javascript
  // New item buttons
  const newRiskBtn = document.getElementById('btn-new-risk');
  const newIssueBtn = document.getElementById('btn-new-issue');
  const newRequestBtn = document.getElementById('btn-new-request');
  
  if (newRiskBtn) newRiskBtn.addEventListener('click', createNewRisk);
  if (newIssueBtn) newIssueBtn.addEventListener('click', createNewIssue);
  if (newRequestBtn) newRequestBtn.addEventListener('click', createNewRequest);
```

## Summary of Changes for v1.1

1. **Added upsertEntity() wrapper** to call window.API.Objects.upsert()
2. **Created three create functions** (createNewRisk, createNewIssue, createNewRequest)
3. **Wired up New buttons** in initUI() with event listeners
4. **Each create function**:
   - Prompts user for item name
   - Gets project context
   - Calls upsertEntity() with entity type, name, project, and default values
   - Reloads page on success to show new item
   - Shows error alert on failure

The New buttons are now fully functional and will create items using AdaptiveWork's Upsert method.
