# Script Field - v2.0

Complete, pasteable AdaptiveWork **Script** field. Combines the original working
data-loading logic (CZQL `/query` endpoint, `Session` auth, `json.entities`) with:

- **Clean picklist display** — values like `/C_RiskImpact/(5) Catastrophic` and
  `/CaseState/New` are stripped to their label (`(5) Catastrophic`, `New`).
- **Inline editing** — click a cell to edit (text / date / dropdown), Save persists
  the `objects` REST endpoint, Cancel reverts. Picklist dropdowns are populated from
  the real values present in your project data, and saves send the exact path format
  AdaptiveWork expects.
- **Fixed Risk columns** — Impact, Likelihood and Risk Rating (heat-map) render in
  three separate columns matching the 8-column Risks table in `html.md`. Risk Rating
  is read-only (AdaptiveWork computes it from Impact × Likelihood; editing Impact or
  Likelihood reloads so the new rating shows).
- **Create** — New Risk / Issue / Request buttons create items via
  `PUT /V2.0/services/data/objects/{EntityType}`.

## Paste this into the Script field

```javascript
/* ============================================================
   AdaptiveWork Custom Panel — JavaScript (v2.0)
   - Loads Risks, Issues, Requests + their Actions via CZQL.
   - Clean picklist labels, inline editing, create + update via REST.
   Auth: Data field supplies { sessionId, project }.
   ============================================================ */

/* ---------- 1. Constants ---------- */
var API_QUERY   = '/V2.0/services/data/query';
var API_OBJECTS = '/V2.0/services/data/objects';

/* Picklist option lists, collected from the loaded data (field -> [rawPath]). */
var PICK = {};
var PICK_FIELDS = ['C_Impact', 'C_Likelihood', 'State', 'Priority', 'RequestType'];
function isPick(f) { return PICK_FIELDS.indexOf(f) > -1; }
function addPick(field, raw) {
  if (!raw) return;
  if (!PICK[field]) PICK[field] = [];
  if (PICK[field].indexOf(raw) === -1) PICK[field].push(raw);
}

/* ---------- 2. CZQL fetch wrapper ---------- */
function czql(baseUrl, sessionId, query) {
  var url = baseUrl.replace(/\/$/, '') + API_QUERY;
  return fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Session ' + sessionId
    },
    body: JSON.stringify({ q: query })
  })
  .then(function (res) {
    if (!res.ok) throw new Error('API error ' + res.status + ' for query: ' + query);
    return res.json();
  })
  .then(function (json) {
    if (json.errorCode) throw new Error(json.message || json.errorCode);
    return json.entities || [];
  });
}

/* ---------- 3. Value helpers ---------- */
/* Pull a string out of any picklist shape (string path or wrapped object). */
function extractPicklistValue(v) {
  if (v === null || v === undefined) return '';
  if (typeof v === 'string' || typeof v === 'number') return String(v);
  if (typeof v === 'object') {
    return v.value || v.Value || v.displayValue || v.DisplayValue ||
           v.name  || v.Name  || v.label        || v.Label        ||
           v.text  || v.Text  || v.id            || v.Id           || '';
  }
  return String(v);
}
/* The raw value we send back to the API (keeps the /Type/Value path intact). */
function pickRaw(v) {
  if (v === null || v === undefined) return '';
  if (typeof v === 'object') return extractPicklistValue(v);
  return String(v);
}
/* The clean label we show in the UI ("/CaseState/New" -> "New"). */
function cleanLabel(s) {
  s = (s === null || s === undefined) ? '' : String(s);
  if (s.charAt(0) === '/') { var p = s.split('/'); s = p[p.length - 1]; }
  return s;
}
/* HTML/attribute escaping. */
function esc(s) {
  return String(s === null || s === undefined ? '' : s)
    .replace(/&/g, '&amp;').replace(/"/g, '&quot;')
    .replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

/* ---------- 4. Normalisers ---------- */
function toRisk(e) {
  var impactRaw = pickRaw(e.C_Impact);
  var likeRaw   = pickRaw(e.C_Likelihood);
  var rateRaw   = pickRaw(e.C_RiskRating);
  var stRaw     = pickRaw(e.State);
  return {
    id:            (e.id || '').replace(/\//g, '-'),
    name:          e.Title || '(unnamed)',
    impactRaw:     impactRaw,  impact:     cleanLabel(impactRaw) || '—',
    likelihoodRaw: likeRaw,    likelihood: cleanLabel(likeRaw)   || '—',
    riskRating:    cleanLabel(rateRaw) || '—',
    statusRaw:     stRaw,      status:     cleanLabel(stRaw) || 'Open',
    owner:         (e.AssignedTo && e.AssignedTo.Name) || '—',
    dueDate:       e.DueDate || null,
    rawId:         e.id || ''
  };
}

function toIssue(e) {
  var prRaw = pickRaw(e.Priority);
  var stRaw = pickRaw(e.State);
  return {
    id:          (e.id || '').replace(/\//g, '-'),
    name:        e.Title || '(unnamed)',
    priorityRaw: prRaw, priority: cleanLabel(prRaw) || 'Medium',
    statusRaw:   stRaw, status:   cleanLabel(stRaw) || 'Open',
    owner:       (e.AssignedTo && e.AssignedTo.Name) || '—',
    raised:      e.CreatedOn || null,
    dueDate:     e.DueDate || null,
    rawId:       e.id || ''
  };
}

function toRequest(e) {
  var tyRaw = pickRaw(e.RequestType) || pickRaw(e.Type);
  var stRaw = pickRaw(e.State);
  return {
    id:         (e.id || '').replace(/\//g, '-'),
    name:       e.Title || '(unnamed)',
    typeRaw:    tyRaw, type:   cleanLabel(tyRaw) || 'Change request',
    statusRaw:  stRaw, status: cleanLabel(stRaw) || 'New',
    requestor:  (e.CreatedBy && e.CreatedBy.Name) || '—',
    submitted:  e.CreatedOn || null,
    decisionBy: e.DueDate || null,
    rawId:      e.id || ''
  };
}

function toAction(e) {
  var today  = new Date();
  var due    = e.DueDate ? new Date(e.DueDate) : null;
  var stateV = cleanLabel(extractPicklistValue(e.ActionItemState));
  var isDone = stateV === 'Complete' || stateV === 'Completed';
  var isLate = !isDone && due && due < today;
  var statusLabel = isDone ? 'Completed'
                 : isLate  ? 'Overdue'
                 : due     ? 'Due ' + fmtDate(e.DueDate)
                 :           'No due date';
  return {
    name:     e.Name || '(unnamed)',
    assignee: (e.EntityOwner && e.EntityOwner.Name) || '—',
    dueDate:  e.DueDate || null,
    status:   statusLabel,
    parentId: (e.Container && e.Container.id) || ''
  };
}

/* ---------- 5. Date helpers ---------- */
function fmtDate(iso) {
  if (!iso) return '—';
  return new Date(iso).toLocaleDateString('en-GB', { day: 'numeric', month: 'short', year: 'numeric' });
}
function dueCls(iso) {
  if (!iso) return '';
  return new Date(iso) < new Date() ? 'overdue' : 'ok';
}

/* ---------- 6. Action lookup map ---------- */
function buildActionMap(actions) {
  var map = {};
  actions.forEach(function (a) {
    var pid = a.parentId;
    if (!pid) return;
    (map[pid] = map[pid] || []).push(a);
  });
  return map;
}

/* ---------- 7. Pill helper ---------- */
var PILL_MAP = {
  Critical: 'pill-critical', High: 'pill-high', Medium: 'pill-medium', Low: 'pill-low',
  Open: 'pill-open', 'In progress': 'pill-inprog', Active: 'pill-inprog',
  Resolved: 'pill-resolved', Complete: 'pill-resolved', Completed: 'pill-resolved',
  Pending: 'pill-pending', New: 'pill-new', Submitted: 'pill-new',
  Approved: 'pill-approved', Rejected: 'pill-rejected', Cancelled: 'pill-rejected', Realised: 'pill-critical'
};
function pill(label) {
  var cls = PILL_MAP[label] || 'pill-medium';
  return '<span class="pill ' + cls + '">' + esc(label) + '</span>';
}

/* ---------- 8. Risk Rating heat-map ---------- */
function riskRatingHeatMap(val) {
  var s = String(val == null ? '' : val).toLowerCase();
  if (!s || s === '—' || s === 'n/a') return 'heatmap-neutral';
  var num = parseFloat(s.replace(/[^0-9.]/g, ''));
  if (!isNaN(num) && /\d/.test(s)) {
    if (num >= 20) return 'heatmap-critical';
    if (num >= 15) return 'heatmap-very-high';
    if (num >= 10) return 'heatmap-high';
    if (num >= 6)  return 'heatmap-medium';
    if (num >= 3)  return 'heatmap-low';
    return 'heatmap-very-low';
  }
  if (s.indexOf('critical') > -1 || s.indexOf('severe') > -1)      return 'heatmap-critical';
  if (s.indexOf('very high') > -1 || s.indexOf('extreme') > -1)    return 'heatmap-very-high';
  if (s.indexOf('high') > -1 || s.indexOf('major') > -1)          return 'heatmap-high';
  if (s.indexOf('medium') > -1 || s.indexOf('moderate') > -1)     return 'heatmap-medium';
  if (s.indexOf('very low') > -1 || s.indexOf('negligible') > -1) return 'heatmap-very-low';
  if (s.indexOf('low') > -1 || s.indexOf('minor') > -1)          return 'heatmap-low';
  return 'heatmap-neutral';
}

/* ---------- 9. Action HTML helpers ---------- */
function actionItemHtml(a) {
  var isDone = a.status === 'Completed';
  var isLate = a.status === 'Overdue';
  var iconCol = isDone ? '#3B6D11' : '#185FA5';
  var dCls = isLate ? 'overdue' : 'ok';
  return '<div class="action-item">' +
    '<i class="ti ti-circle-check" style="font-size:14px;color:' + iconCol + '" aria-hidden="true"></i>' +
    '<span class="action-name">' + esc(a.name) + '</span>' +
    '<span class="action-meta">' + esc(a.assignee) + '</span>' +
    '<span class="action-due ' + dCls + '">· ' + a.status + '</span>' +
    '</div>';
}
function actionsBlock(actions) {
  if (!actions || !actions.length) {
    return '<div class="actions-block"><div class="actions-label">Associated actions</div>' +
           '<p style="font-size:12px;color:#5f6368;margin:4px 0 0">No actions recorded.</p></div>';
  }
  return '<div class="actions-block"><div class="actions-label">Associated actions</div>' +
    actions.map(actionItemHtml).join('') + '</div>';
}

/* ---------- 10. Editable-cell HTML helper ----------
   field   = AdaptiveWork API field name (Title, C_Impact, State, DueDate ...)
   id      = full entity id (e.g. /Risk/guid)
   rawVal  = value to store for editing/sending (raw path for picklists)
   display = inner HTML to show when not editing
   extra   = extra <td> classes */
function editCell(field, id, rawVal, display, extra) {
  return '<td class="editable' + (extra ? ' ' + extra : '') + '"' +
    ' data-field="' + field + '"' +
    ' data-id="' + esc(id) + '"' +
    ' data-value="' + esc(rawVal) + '">' + display + '</td>';
}
/* Inner HTML for a cell given its field + raw value (used after save/cancel). */
function cellDisplay(field, raw) {
  if (field === 'DueDate') return '<span class="action-due ' + dueCls(raw) + '">' + fmtDate(raw) + '</span>';
  if (field === 'State' || field === 'Priority') return pill(cleanLabel(raw));
  if (isPick(field)) return esc(cleanLabel(raw));
  return esc(raw);
}

/* ---------- 11. Renderers ---------- */
function renderRisks(risks, actionMap) {
  var tbody = document.getElementById('risks-body');
  if (!risks.length) {
    tbody.innerHTML = '<tr><td colspan="8"><div class="no-data">No risks found for this project.</div></td></tr>';
    document.getElementById('badge-risks').textContent = '0';
    return;
  }
  tbody.innerHTML = risks.map(function (r) {
    var actions = actionMap[r.rawId] || [];
    var hm = riskRatingHeatMap(r.riskRating);
    return '<tr class="data-row" data-status="' + esc(r.status) + '">' +
      '<td><button class="expand-btn" data-target="' + r.id + '-actions" aria-label="Show actions"></button></td>' +
      editCell('Title',        r.rawId, r.name,          esc(r.name)) +
      editCell('C_Impact',     r.rawId, r.impactRaw,     esc(r.impact)) +
      editCell('C_Likelihood', r.rawId, r.likelihoodRaw, esc(r.likelihood)) +
      '<td><span class="heatmap ' + hm + '">' + esc(r.riskRating) + '</span></td>' +
      editCell('State',        r.rawId, r.statusRaw,     pill(r.status)) +
      '<td>' + esc(r.owner) + '</td>' +
      editCell('DueDate',      r.rawId, r.dueDate || '', '<span class="action-due ' + dueCls(r.dueDate) + '">' + fmtDate(r.dueDate) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + r.id + '-actions"><td colspan="8">' + actionsBlock(actions) + '</td></tr>';
  }).join('');
  document.getElementById('badge-risks').textContent = risks.length;
}

function renderIssues(issues, actionMap) {
  var tbody = document.getElementById('issues-body');
  if (!issues.length) {
    tbody.innerHTML = '<tr><td colspan="7"><div class="no-data">No issues found for this project.</div></td></tr>';
    document.getElementById('badge-issues').textContent = '0';
    return;
  }
  tbody.innerHTML = issues.map(function (issue) {
    var actions = actionMap[issue.rawId] || [];
    return '<tr class="data-row" data-status="' + esc(issue.status) + '">' +
      '<td><button class="expand-btn" data-target="' + issue.id + '-actions" aria-label="Show actions"></button></td>' +
      editCell('Title',    issue.rawId, issue.name,        esc(issue.name)) +
      editCell('Priority', issue.rawId, issue.priorityRaw, pill(issue.priority)) +
      editCell('State',    issue.rawId, issue.statusRaw,   pill(issue.status)) +
      '<td>' + esc(issue.owner) + '</td>' +
      '<td>' + fmtDate(issue.raised) + '</td>' +
      editCell('DueDate',  issue.rawId, issue.dueDate || '', '<span class="action-due ' + dueCls(issue.dueDate) + '">' + fmtDate(issue.dueDate) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + issue.id + '-actions"><td colspan="7">' + actionsBlock(actions) + '</td></tr>';
  }).join('');
  document.getElementById('badge-issues').textContent = issues.length;
}

function renderRequests(requests, actionMap) {
  var tbody = document.getElementById('requests-body');
  if (!requests.length) {
    tbody.innerHTML = '<tr><td colspan="7"><div class="no-data">No requests found for this project.</div></td></tr>';
    document.getElementById('badge-requests').textContent = '0';
    return;
  }
  tbody.innerHTML = requests.map(function (req) {
    var actions = actionMap[req.rawId] || [];
    return '<tr class="data-row" data-status="' + esc(req.status) + '">' +
      '<td><button class="expand-btn" data-target="' + req.id + '-actions" aria-label="Show actions"></button></td>' +
      editCell('Title',       req.rawId, req.name,    esc(req.name)) +
      editCell('RequestType', req.rawId, req.typeRaw, esc(req.type)) +
      editCell('State',       req.rawId, req.statusRaw, pill(req.status)) +
      '<td>' + esc(req.requestor) + '</td>' +
      '<td>' + fmtDate(req.submitted) + '</td>' +
      editCell('DueDate',     req.rawId, req.decisionBy || '', '<span class="action-due ' + dueCls(req.decisionBy) + '">' + fmtDate(req.decisionBy) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + req.id + '-actions"><td colspan="7">' + actionsBlock(actions) + '</td></tr>';
  }).join('');
  document.getElementById('badge-requests').textContent = requests.length;
}

/* Collect picklist options from loaded data, then render everything. */
function renderAll(risks, issues, requests, actionMap) {
  PICK = {};
  risks.forEach(function (r) {
    addPick('C_Impact', r.impactRaw);
    addPick('C_Likelihood', r.likelihoodRaw);
    addPick('State', r.statusRaw);
  });
  issues.forEach(function (i) {
    addPick('Priority', i.priorityRaw);
    addPick('State', i.statusRaw);
  });
  requests.forEach(function (q) {
    addPick('RequestType', q.typeRaw);
    addPick('State', q.statusRaw);
  });
  renderRisks(risks, actionMap);
  renderIssues(issues, actionMap);
  renderRequests(requests, actionMap);
}

/* ---------- 12. Loading / error helpers ---------- */
function showLoading(tbodyId, cols) {
  document.getElementById(tbodyId).innerHTML =
    '<tr><td colspan="' + cols + '"><div class="no-data">Loading…</div></td></tr>';
}
function showError(tbodyId, cols, msg) {
  document.getElementById(tbodyId).innerHTML =
    '<tr><td colspan="' + cols + '"><div class="no-data" style="color:#A32D2D">' + esc(msg) + '</div></td></tr>';
}

/* ---------- 13. Context + API host ---------- */
function getContext() {
  var ctx = API.Context.getData();
  if (!ctx || !ctx.sessionId || !ctx.project || !ctx.project.id) {
    throw new Error('Data field must supply sessionId and project object.');
  }
  var appHost = window.location.hostname;
  var apiHost = appHost
    .replace(/^app2\./, 'api2.')
    .replace(/^app\./, 'api.')
    .replace(/^eu1\./, 'apie1.')
    .replace(/^eu\./, 'apie.');
  return { sid: ctx.sessionId, base: 'https://' + apiHost, projId: ctx.project.id };
}

/* ---------- 14. Create + update via the objects REST endpoint ----------
   Create: PUT  /V2.0/services/data/objects/{EntityType}  body = { field: value }
           -> { "id": "/Risk/..." }
   Update: POST /V2.0/services/data/objects/{EntityType}/{id}  body = { field: value }
   Reference / picklist values use the "/Type/Value" path format (as returned). */
function createObject(base, sid, entityType, fields) {
  var url = base.replace(/\/$/, '') + API_OBJECTS + '/' + entityType;
  return fetch(url, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Session ' + sid
    },
    body: JSON.stringify(fields)
  })
  .then(function (res) {
    if (!res.ok) throw new Error('HTTP ' + res.status);
    return res.json().catch(function () { return {}; });
  });
}

function updateObject(base, sid, entityId, fields) {
  /* entityId is the full path e.g. "/Risk/guid" -> .../objects/Risk/guid */
  var url = base.replace(/\/$/, '') + API_OBJECTS + '/' + String(entityId).replace(/^\//, '');
  return fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Session ' + sid
    },
    body: JSON.stringify(fields)
  })
  .then(function (res) {
    if (!res.ok) throw new Error('HTTP ' + res.status);
    return res.json().catch(function () { return {}; });
  });
}

function updateField(entityId, field, value) {
  var c = getContext();
  var fields = {};
  fields[field] = value;
  return updateObject(c.base, c.sid, entityId, fields);
}

function createEntity(entityType, fields, label) {
  var c;
  try { c = getContext(); } catch (e) { alert('Cannot create ' + label + ': ' + e.message); return; }
  fields.PlannedFor = c.projId;
  createObject(c.base, c.sid, entityType, fields)
    .then(function () { location.reload(); })
    .catch(function (err) { alert('Failed to create ' + label + ': ' + err.message); });
}

function createNewRisk() {
  var name = prompt('Enter risk name:');
  if (!name) return;
  createEntity('Risk', { Title: name }, 'risk');
}
function createNewIssue() {
  var name = prompt('Enter issue name:');
  if (!name) return;
  createEntity('Issue', { Title: name }, 'issue');
}
function createNewRequest() {
  var name = prompt('Enter request name:');
  if (!name) return;
  createEntity('EnhancementRequest', { Title: name }, 'request');
}

/* ---------- 15. Inline editing ---------- */
function startEdit(cell) {
  if (cell.classList.contains('editing')) return;
  var field = cell.getAttribute('data-field');
  var value = cell.getAttribute('data-value') || '';
  var inputHtml;

  if (isPick(field)) {
    var opts = (PICK[field] || []).slice();
    if (value && opts.indexOf(value) === -1) opts.push(value);
    inputHtml = '<select class="edit-input">' + opts.map(function (o) {
      return '<option value="' + esc(o) + '"' + (o === value ? ' selected' : '') + '>' + esc(cleanLabel(o)) + '</option>';
    }).join('') + '</select>';
  } else if (field === 'DueDate') {
    inputHtml = '<input type="date" class="edit-input" value="' + (value ? value.split('T')[0] : '') + '" />';
  } else {
    inputHtml = '<input type="text" class="edit-input" value="' + esc(value) + '" />';
  }

  cell.classList.add('editing');
  cell.innerHTML = inputHtml +
    '<button class="edit-save">Save</button>' +
    '<button class="edit-cancel">Cancel</button>';
  var inp = cell.querySelector('.edit-input');
  if (inp) inp.focus();
}

function cancelEdit(cell) {
  cell.classList.remove('editing');
  cell.innerHTML = cellDisplay(cell.getAttribute('data-field'), cell.getAttribute('data-value') || '');
}

function saveEdit(cell) {
  var input = cell.querySelector('.edit-input');
  if (!input) return;
  var newValue = input.value;
  var field = cell.getAttribute('data-field');
  var id = cell.getAttribute('data-id');
  if (!newValue) { alert('Field cannot be empty'); return; }

  var btn = cell.querySelector('.edit-save');
  if (btn) { btn.disabled = true; btn.textContent = 'Saving…'; }

  updateField(id, field, newValue)
    .then(function () {
      /* Impact/Likelihood drive the computed Risk Rating — reload to refresh it. */
      if (field === 'C_Impact' || field === 'C_Likelihood') { location.reload(); return; }
      cell.setAttribute('data-value', newValue);
      cell.classList.remove('editing');
      cell.innerHTML = cellDisplay(field, newValue);
    })
    .catch(function (err) {
      alert('Save failed: ' + err.message);
      if (btn) { btn.disabled = false; btn.textContent = 'Save'; }
    });
}

/* ---------- 16. Filtering ---------- */
function filterTable(tbodyId, query) {
  var q = query.toLowerCase();
  document.querySelectorAll('#' + tbodyId + ' .data-row').forEach(function (row) {
    row.style.display = row.innerText.toLowerCase().indexOf(q) > -1 ? '' : 'none';
  });
}
function filterByStatus(tbodyId, val) {
  document.querySelectorAll('#' + tbodyId + ' .data-row').forEach(function (row) {
    var v = row.getAttribute('data-status') || '';
    row.style.display = (!val || v === val) ? '' : 'none';
  });
}

/* ---------- 17. UI wiring ---------- */
function initUI() {
  document.querySelectorAll('.aw-tab[data-tab]').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var name = btn.getAttribute('data-tab');
      document.querySelectorAll('.aw-tab').forEach(function (t) { t.classList.remove('active'); });
      document.querySelectorAll('.aw-tab-pane').forEach(function (p) { p.classList.remove('active'); });
      btn.classList.add('active');
      document.getElementById('pane-' + name).classList.add('active');
    });
  });

  ['risks-body', 'issues-body', 'requests-body'].forEach(function (id) {
    var tbody = document.getElementById(id);
    if (!tbody) return;
    tbody.addEventListener('click', function (e) {
      var expBtn = e.target.closest('.expand-btn');
      if (expBtn) {
        var row = document.getElementById(expBtn.getAttribute('data-target'));
        if (row) {
          var isOpen = row.classList.contains('show');
          row.classList.toggle('show', !isOpen);
          expBtn.classList.toggle('open', !isOpen);
        }
        return;
      }
      var saveBtn = e.target.closest('.edit-save');
      if (saveBtn) { saveEdit(saveBtn.closest('.editable')); return; }
      var cancelBtn = e.target.closest('.edit-cancel');
      if (cancelBtn) { cancelEdit(cancelBtn.closest('.editable')); return; }
      var editable = e.target.closest('.editable');
      if (editable && !editable.classList.contains('editing')) { startEdit(editable); return; }
    });
  });

  [
    { input: 'input[placeholder="Search risks…"]',    tbody: 'risks-body' },
    { input: 'input[placeholder="Search issues…"]',   tbody: 'issues-body' },
    { input: 'input[placeholder="Search requests…"]', tbody: 'requests-body' }
  ].forEach(function (s) {
    var el = document.querySelector(s.input);
    if (el) el.addEventListener('input', function () { filterTable(s.tbody, el.value); });
  });

  [
    { pane: 'pane-risks',    tbody: 'risks-body' },
    { pane: 'pane-issues',   tbody: 'issues-body' },
    { pane: 'pane-requests', tbody: 'requests-body' }
  ].forEach(function (s) {
    var pane = document.getElementById(s.pane);
    if (!pane) return;
    var sel = pane.querySelector('select');
    if (sel) sel.addEventListener('change', function () { filterByStatus(s.tbody, sel.value); });
  });

  var nr = document.getElementById('btn-new-risk');
  var ni = document.getElementById('btn-new-issue');
  var nq = document.getElementById('btn-new-request');
  if (nr) nr.addEventListener('click', createNewRisk);
  if (ni) ni.addEventListener('click', createNewIssue);
  if (nq) nq.addEventListener('click', createNewRequest);
}

/* ---------- 18. Bootstrap ---------- */
setTimeout(function () {
  initUI();

  var c;
  try {
    c = getContext();
  } catch (e) {
    showError('risks-body', 8, e.message);
    showError('issues-body', 7, e.message);
    showError('requests-body', 7, e.message);
    return;
  }

  showLoading('risks-body', 8);
  showLoading('issues-body', 7);
  showLoading('requests-body', 7);

  var qRisks =
    "SELECT Title, C_Impact, C_Likelihood, C_RiskRating, State, AssignedTo.Name, DueDate " +
    "FROM Risk WHERE PlannedFor = '" + c.projId + "'";
  var qIssues =
    "SELECT Title, Priority, State, AssignedTo.Name, CreatedOn, DueDate " +
    "FROM Issue WHERE PlannedFor = '" + c.projId + "'";
  var qRequests =
    "SELECT Title, RequestType, State, CreatedBy.Name, CreatedOn, DueDate " +
    "FROM EnhancementRequest WHERE PlannedFor = '" + c.projId + "'";

  Promise.all([
    czql(c.base, c.sid, qRisks),
    czql(c.base, c.sid, qIssues),
    czql(c.base, c.sid, qRequests)
  ])
  .then(function (results) {
    var risks    = results[0].map(toRisk);
    var issues   = results[1].map(toIssue);
    var requests = results[2].map(toRequest);

    var caseIds = []
      .concat(results[0].map(function (e) { return e.id; }))
      .concat(results[1].map(function (e) { return e.id; }))
      .concat(results[2].map(function (e) { return e.id; }));

    if (!caseIds.length) {
      renderAll(risks, issues, requests, {});
      return;
    }

    var inList = caseIds.map(function (id) { return "'" + id + "'"; }).join(',');
    var qActions =
      "SELECT Name, EntityOwner.Name, DueDate, ActionItemState, Container.id " +
      "FROM ActionItem WHERE Container IN (" + inList + ")";

    return czql(c.base, c.sid, qActions).then(function (actionEntities) {
      var actionMap = buildActionMap(actionEntities.map(toAction));
      renderAll(risks, issues, requests, actionMap);
    });
  })
  .catch(function (err) {
    console.error('AdaptiveWork panel fetch error:', err);
    var msg = 'Failed to load data: ' + err.message;
    showError('risks-body', 8, msg);
    showError('issues-body', 7, msg);
    showError('requests-body', 7, msg);
  });
}, 100);
```

## What changed in this revision

| Problem | Fix |
|---------|-----|
| Cells showed raw paths (`/CaseState/New`, `/C_RiskImpact/(5) Catastrophic`) | `cleanLabel()` strips the `/Type/` prefix; `pickRaw()` keeps the full path for sending back |
| Could no longer edit fields | Restored `startEdit` / `saveEdit` / `cancelEdit` + click delegation in `initUI()` |
| Picklist edits could send invalid values | Dropdowns are built from the real values present in your data (`PICK`), and the original path is sent on save |
| Risk Rating editable / stale | Risk Rating is read-only; editing Impact or Likelihood reloads so the recomputed rating shows |
| Create returned HTTP 500 (wrong `/upsert` body) | Create now uses `PUT /objects/{EntityType}`; update uses `POST /objects/{EntityType}/{id}` — the documented REST endpoints |
```
