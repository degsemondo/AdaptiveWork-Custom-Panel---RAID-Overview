# HTML Field - v2.0

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>AdaptiveWork — Project panel</title>

  <!-- Tabler outline icon font -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@latest/tabler-icons.min.css" />

  <!-- Panel stylesheet -->
  <link rel="stylesheet" href="panel.css" />
</head>
<body>

<div class="aw-panel">

  <!-- ── Single-row header: tabs + toolbar + export ── -->
  <div class="aw-header">
    <div class="aw-tabs" role="tablist">
      <button class="aw-tab active" role="tab" aria-selected="true"
              aria-controls="pane-risks" data-tab="risks">
        <i class="ti ti-alert-triangle" style="font-size:14px" aria-hidden="true"></i>
        Risks
        <span class="aw-badge badge-risk" id="badge-risks">0</span>
      </button>

      <button class="aw-tab" role="tab" aria-selected="false"
              aria-controls="pane-issues" data-tab="issues">
        <i class="ti ti-bug" style="font-size:14px" aria-hidden="true"></i>
        Issues
        <span class="aw-badge badge-issue" id="badge-issues">0</span>
      </button>

      <button class="aw-tab" role="tab" aria-selected="false"
              aria-controls="pane-requests" data-tab="requests">
        <i class="ti ti-clipboard-list" style="font-size:14px" aria-hidden="true"></i>
        Requests
        <span class="aw-badge badge-request" id="badge-requests">0</span>
      </button>
    </div>

    <!-- Shared toolbar — acts on the active tab -->
    <div class="aw-toolbar">
      <input type="text" id="aw-search" placeholder="Search risks…" aria-label="Search" />

      <select id="aw-filter" aria-label="Filter by status">
        <option value="">All statuses</option>
        <option value="New">New</option>
        <option value="Closed">Closed</option>
        <option value="Realised">Realised</option>
      </select>

      <button class="aw-btn aw-btn-primary" id="btn-new" aria-label="Add new item">
        <i class="ti ti-plus" style="font-size:13px" aria-hidden="true"></i>
        <span id="btn-new-label">New risk</span>
      </button>
    </div>

    <button class="aw-btn" id="btn-export" aria-label="Export all tabs to Excel">
      <i class="ti ti-file-spreadsheet" style="font-size:14px" aria-hidden="true"></i>
      Export
    </button>
  </div>

  <div class="aw-body">

    <!-- ══════════════════════════════════════
         TAB 1 — RISKS
    ══════════════════════════════════════ -->
    <div class="aw-tab-pane active" id="pane-risks" role="tabpanel">

      <div class="aw-table-wrap">
        <table class="aw-table" role="grid" aria-label="Risks">
          <thead>
            <tr>
              <th style="width:28px"></th>
              <th>ID</th>
              <th>Title</th>
              <th>Probability</th>
              <th>Impact</th>
              <th>Score</th>
              <th>Status</th>
              <th>Owner</th>
              <th>Reporting Level</th>
              <th>Impact Date</th>
            </tr>
          </thead>
          <tbody id="risks-body">
            <!-- Populated dynamically by panel.js -->
          </tbody>
        </table>
      </div>
    </div>

    <!-- ══════════════════════════════════════
         TAB 2 — ISSUES
    ══════════════════════════════════════ -->
    <div class="aw-tab-pane" id="pane-issues" role="tabpanel">

      <div class="aw-table-wrap">
        <table class="aw-table" role="grid" aria-label="Issues">
          <thead>
            <tr>
              <th style="width:28px"></th>
              <th>Issue name</th>
              <th>Priority</th>
              <th>Status</th>
              <th>Owner</th>
              <th>Raised</th>
              <th>Due date</th>
            </tr>
          </thead>
          <tbody id="issues-body">
            <!-- Populated dynamically by panel.js -->
          </tbody>
        </table>
      </div>
    </div>

    <!-- ══════════════════════════════════════
         TAB 3 — REQUESTS
    ══════════════════════════════════════ -->
    <div class="aw-tab-pane" id="pane-requests" role="tabpanel">

      <div class="aw-table-wrap">
        <table class="aw-table" role="grid" aria-label="Requests">
          <thead>
            <tr>
              <th style="width:28px"></th>
              <th>Request name</th>
              <th>Type</th>
              <th>Status</th>
              <th>Requestor</th>
              <th>Submitted</th>
              <th>Decision by</th>
            </tr>
          </thead>
          <tbody id="requests-body">
            <!-- Populated dynamically by panel.js -->
          </tbody>
        </table>
      </div>
    </div>

  </div><!-- /.aw-body -->
</div><!-- /.aw-panel -->

</body>
</html>
```

## Features
- Three-tab RAID interface (Risks, Issues, Requests)
- Semantic HTML with ARIA labels for accessibility
- Tabler icon font integration
- Search input for each tab
- Status filter dropdown for each tab
- New Risk/Issue/Request buttons
- Expandable rows for associated actions
- Badge counters showing item counts per tab
