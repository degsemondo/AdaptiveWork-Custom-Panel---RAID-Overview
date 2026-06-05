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

/* Latest loaded data, retained so "Export to Excel" can use it. */
var DATA = { risks: [], issues: [], requests: [], actionMap: {} };

/* Picklist option lists, collected from the loaded data (field -> [rawPath]). */
var PICK = {};
/* Authoritative picklist options fetched from AdaptiveWork metadata
   (field -> [{ raw, label }]). Preferred over guessing a "/Type" prefix. */
var PICK_META = {};
var PICK_FIELDS = ['C_Impact', 'C_Likelihood', 'State', 'Priority', 'RequestType', 'C_ReportingLevel', 'ActionItemState'];
function isPick(f) { return PICK_FIELDS.indexOf(f) > -1; }

/* Status-like fields rendered as a coloured pill; date fields as a formatted date. */
function isPillField(f) { return f === 'State' || f === 'Priority'; }
function isDateField(f) { return f === 'DueDate' || f === 'C_ImpactDate'; }
function addPick(field, raw) {
  if (!raw) return;
  if (!PICK[field]) PICK[field] = [];
  if (PICK[field].indexOf(raw) === -1) PICK[field].push(raw);
}

/* User reference fields render as a type-ahead (there can be many users, so we
   search server-side as you type rather than bulk-loading everyone).
   USER_NAME maps a known /User/.. id -> display name (for showing the cell). */
var USER_NAME = {};
var USER_FIELDS = ['AssignedTo', 'Owner'];
function isUserField(f) { return USER_FIELDS.indexOf(f) > -1; }
function rememberUser(id, name) { if (id) USER_NAME[id] = name || USER_NAME[id] || cleanLabel(id); }

/* Search system users by (partial) name. Returns [{ id, name }] (capped). */
function searchUsers(term) {
  var c = getContext();
  var safe = String(term).trim().replace(/'/g, "''");   /* escape single quotes */
  var q = "SELECT Name FROM User WHERE Name LIKE '%" + safe + "%'";
  return czql(c.base, c.sid, q).then(function (rows) {
    return rows.map(function (u) { return { id: u.id, name: u.Name || cleanLabel(u.id) }; })
               .sort(function (a, b) { return a.name.localeCompare(b.name); })
               .slice(0, 25);
  });
}

/* Wire type-ahead behaviour onto a text input + its sibling suggestions menu.
   The chosen user's id is stored on the input as data-user-id (read on save). */
function attachUserTypeahead(input, menu) {
  var timer = null;
  function close() { menu.style.display = 'none'; menu.innerHTML = ''; }
  function pick(item) {
    input.value = item.getAttribute('data-name');
    input.setAttribute('data-user-id', item.getAttribute('data-id'));
    rememberUser(item.getAttribute('data-id'), item.getAttribute('data-name'));
    close();
  }
  function render(list) {
    if (!list.length) { menu.innerHTML = '<div class="ta-empty">No matches</div>'; menu.style.display = 'block'; return; }
    menu.innerHTML = list.map(function (u) {
      return '<div class="ta-item" data-id="' + esc(u.id) + '" data-name="' + esc(u.name) + '">' + esc(u.name) + '</div>';
    }).join('');
    menu.style.display = 'block';
  }
  input.addEventListener('input', function () {
    input.setAttribute('data-user-id', '');   /* typing invalidates any prior pick */
    var term = input.value.trim();
    if (timer) clearTimeout(timer);
    if (term.length < 2) { close(); return; }
    timer = setTimeout(function () { searchUsers(term).then(render).catch(close); }, 250);
  });
  input.addEventListener('keydown', function (e) {
    if (e.key === 'Enter' && menu.style.display === 'block') {
      var first = menu.querySelector('.ta-item');
      if (first) { e.preventDefault(); e.stopPropagation(); pick(first); }
    }
  });
  menu.addEventListener('mousedown', function (e) {
    var item = e.target.closest('.ta-item');
    if (item) { e.preventDefault(); pick(item); }
  });
  input.addEventListener('blur', function () { setTimeout(close, 150); });
}

/* Canonical, fully-ordered option labels for the calculated-risk picklists.
   These always appear in the dropdowns (Impact / Probability) in this order,
   regardless of which values existing records happen to use. The API read/write
   is unchanged — we still send the "/Type/Value" raw path. */
var PICK_OPTIONS = {
  C_Impact:         ['1 - Minor', '2 - Moderate', '3 - Moderate +', '4 - Significant', '5 - Significant +', '6 - Severe'],
  C_Likelihood:     ['1 - Unlikely', '2 - Possible', '3 - Possible +', '4 - Likely', '5 - Likely +', '6 - Very Likely'],
  C_ReportingLevel: ['1 - EFDC', '2 - Portfolio', '3 - Project Board', '4 - Project Team']
};
/* Fallback "/Type" prefix used only if no existing record reveals it.
   Picklist TYPE names are "C_" + the DEFINING entity + the field's label
   (Impact/Probability are defined on Risk -> /C_RiskImpact, /C_RiskProbability;
   Reporting Level is defined on the Case ancestor -> /C_CaseReportingLevel). */
var PICK_PREFIX_DEFAULT = {
  C_Impact: '/C_RiskImpact', C_Likelihood: '/C_RiskProbability', C_ReportingLevel: '/C_CaseReportingLevel'
};

/* Detect the "/Type" prefix for a field from any raw value present in the data. */
function pickPrefix(field) {
  var arr = PICK[field] || [];
  for (var i = 0; i < arr.length; i++) {
    var v = arr[i];
    if (v && v.charAt(0) === '/') { var p = v.split('/'); p.pop(); return p.join('/'); }
  }
  return PICK_PREFIX_DEFAULT[field] || '';
}

/* Build [{ raw, label }] options for a picklist field.
   - Fields with a canonical list (Impact / Probability) show all options in
     order; the raw path is reused from the data when available, else built from
     the detected/default "/Type" prefix.
   - Other fields fall back to the values present in the loaded data.
   - `current` (the row's raw value) is always included so it stays selectable. */
function pickOptionList(field, current) {
  var out = [], seen = {};

  /* Prefer real option paths from metadata — no prefix guessing needed. */
  var meta = PICK_META[field];
  if (meta && meta.length) {
    var seenM = {};
    meta.forEach(function (o) { if (o.raw && !seenM[o.raw]) { seenM[o.raw] = 1; out.push(o); } });
    if (current && !seenM[current]) out.push({ raw: current, label: cleanLabel(current) });
    return out;
  }

  function add(raw) {
    if (raw === null || raw === undefined || raw === '') return;
    var lbl = cleanLabel(raw);
    if (seen[lbl]) return;
    seen[lbl] = true;
    out.push({ raw: raw, label: lbl });
  }

  var canon = PICK_OPTIONS[field];
  if (canon) {
    var prefix = pickPrefix(field);
    var byLabel = {};
    (PICK[field] || []).forEach(function (r) { byLabel[cleanLabel(r)] = r; });
    canon.forEach(function (lbl) {
      add(byLabel[lbl] || (prefix ? prefix + '/' + lbl : lbl));
    });
    (PICK[field] || []).forEach(add);   /* keep any non-standard legacy values */
  } else {
    (PICK[field] || []).forEach(add);
  }
  add(current);
  return out;
}

/* <option> HTML for a picklist field. */
function pickOptionsHtml(field, current, includeBlank) {
  var html = includeBlank ? '<option value="">—</option>' : '';
  return html + pickOptionList(field, current).map(function (o) {
    return '<option value="' + esc(o.raw) + '"' +
      (o.raw === current ? ' selected' : '') + '>' + esc(o.label) + '</option>';
  }).join('');
}

/* ---------- 1b. Picklist options from metadata ----------
   describeEntities returns each field's real option paths, so we don't have to
   guess the "/Type" prefix (which differs from the field name — e.g. C_Impact's
   values live under /C_RiskImpact/...). Best-effort: schema-agnostic parsing and
   graceful fallback to data-derived options if anything is unavailable. */
function parsePicklistOptions(fd) {
  /* Find the first array property whose items look like picklist options
     (either "/Type/Value" strings, or objects carrying a value/id + label). */
  for (var k in fd) {
    if (k === 'flags') continue;
    var v = fd[k];
    if (!Array.isArray(v) || !v.length) continue;
    var first = v[0];
    if (typeof first === 'string') {
      if (first.charAt(0) === '/') {
        return v.map(function (s) { return { raw: s, label: cleanLabel(s) }; });
      }
      continue;
    }
    if (first && typeof first === 'object') {
      var probe = first.value || first.id || first.Value || first.Id;
      if (probe) {
        return v.map(function (o) {
          var raw = o.value || o.id || o.Value || o.Id || '';
          return { raw: raw, label: o.label || o.name || o.Label || o.Name || cleanLabel(raw) };
        });
      }
    }
  }
  return null;
}
function loadPicklistMeta() {
  var c = getContext();
  var url = c.base.replace(/\/$/, '') +
    '/V2.0/services/metadata/describeEntities?typeNames=Risk,Issue,EnhancementRequest,ActionItem';
  return fetch(url, { headers: { 'Authorization': 'Session ' + c.sid } })
    .then(function (res) { if (!res.ok) throw new Error('HTTP ' + res.status); return res.json(); })
    .then(function (json) {
      var descs = json.entityDescriptions || json.descriptions || [];
      descs.forEach(function (d) {
        (d.fields || []).forEach(function (fd) {
          if (!fd || PICK_FIELDS.indexOf(fd.name) === -1) return;
          if (PICK_META[fd.name]) return;            /* first definition wins */
          var opts = parsePicklistOptions(fd);
          if (opts && opts.length) PICK_META[fd.name] = opts;
        });
      });
    });
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
  var impRaw  = pickRaw(e.C_Impact);
  var probRaw = pickRaw(e.C_Likelihood);
  var rateRaw = pickRaw(e.C_RiskRating);
  var stRaw   = pickRaw(e.State);
  var rlRaw   = pickRaw(e.C_ReportingLevel);
  return {
    id:                (e.id || '').replace(/\//g, '-'),
    sysId:             e.SYSID || '',
    name:              e.Title || '(unnamed)',
    probabilityRaw:    probRaw, probability: cleanLabel(probRaw) || '—',
    impactRaw:         impRaw,  impact:      cleanLabel(impRaw)  || '—',
    riskRating:        cleanLabel(rateRaw) || '—',
    statusRaw:         stRaw,   status:      cleanLabel(stRaw) || 'Open',
    ownerRaw:          (e.Owner && e.Owner.id) || '',
    owner:             (e.Owner && e.Owner.Name) || '—',
    reportingLevelRaw: rlRaw,   reportingLevel: cleanLabel(rlRaw) || '—',
    impactDate:        e.C_ImpactDate || null,
    rawId:             e.id || ''
  };
}

function toIssue(e) {
  var prRaw = pickRaw(e.Priority);
  var stRaw = pickRaw(e.State);
  return {
    id:          (e.id || '').replace(/\//g, '-'),
    sysId:       e.SYSID || '',
    name:        e.Title || '(unnamed)',
    priorityRaw: prRaw, priority: cleanLabel(prRaw) || 'Medium',
    statusRaw:   stRaw, status:   cleanLabel(stRaw) || 'Open',
    ownerRaw:    (e.AssignedTo && e.AssignedTo.id) || '',
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
    sysId:      e.SYSID || '',
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
    rawId:    e.id || '',
    name:     e.Name || '(unnamed)',
    ownerRaw: (e.EntityOwner && e.EntityOwner.id) || '',
    assignee: (e.EntityOwner && e.EntityOwner.Name) || '—',
    dueDate:  e.DueDate || null,
    stateRaw: pickRaw(e.ActionItemState),
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
/* Status label + flags from an action's state + due date (used for display). */
function actionStatusInfo(stateRaw, dueIso) {
  var due = dueIso ? new Date(dueIso) : null;
  var stateV = cleanLabel(stateRaw);
  var isDone = stateV === 'Complete' || stateV === 'Completed';
  var isLate = !isDone && due && due < new Date();
  var label = isDone ? 'Completed'
            : isLate  ? 'Overdue'
            : due     ? 'Due ' + fmtDate(dueIso)
            :           'No due date';
  return { label: label, done: isDone, late: isLate };
}
/* Inner HTML of an action row in display mode (icon, name, owner, status, tools). */
function actionInnerHtml(name, ownerName, stateRaw, dueIso) {
  var s = actionStatusInfo(stateRaw, dueIso);
  var iconCol = s.done ? '#3B6D11' : '#185FA5';
  var dCls = s.late ? 'overdue' : 'ok';
  return '<i class="ti ti-circle-check" style="font-size:14px;color:' + iconCol + '" aria-hidden="true"></i>' +
    '<span class="action-name">' + esc(name) + '</span>' +
    '<span class="action-meta">' + esc(ownerName && ownerName !== '—' ? ownerName : '—') + '</span>' +
    '<span class="action-due ' + dCls + '">· ' + esc(s.label) + '</span>' +
    '<span class="action-tools">' +
      '<button class="act-edit" title="Edit action" aria-label="Edit action"><i class="ti ti-pencil" aria-hidden="true"></i></button>' +
      '<button class="act-del" title="Delete action" aria-label="Delete action"><i class="ti ti-trash" aria-hidden="true"></i></button>' +
    '</span>';
}
/* One action row, carrying its data on data-* attributes for in-place editing. */
function actionItemHtml(a) {
  return '<div class="action-item"' +
    ' data-action-id="' + esc(a.rawId) + '"' +
    ' data-name="' + esc(a.name) + '"' +
    ' data-owner="' + esc(a.ownerRaw) + '"' +
    ' data-owner-name="' + esc(a.assignee) + '"' +
    ' data-due="' + esc(a.dueDate || '') + '"' +
    ' data-state="' + esc(a.stateRaw) + '">' +
    actionInnerHtml(a.name, a.assignee, a.stateRaw, a.dueDate) +
    '</div>';
}
/* Edit form for an action (used for both edit and add). */
function actionFormHtml(a) {
  a = a || {};
  var ownerName = (a.assignee && a.assignee !== '—') ? a.assignee : '';
  return '<div class="action-edit">' +
    '<input type="text" class="edit-input act-f-name" value="' + esc(a.name || '') + '" placeholder="Action name" />' +
    '<span class="ta-cell"><input type="text" class="edit-input ta-input act-f-owner" autocomplete="off"' +
      ' data-user-id="' + esc(a.ownerRaw || '') + '" value="' + esc(ownerName) + '" placeholder="Owner" />' +
      '<div class="ta-menu" style="display:none"></div></span>' +
    '<input type="date" class="edit-input act-f-due" value="' + ((a.dueDate || '').split('T')[0]) + '" />' +
    '<select class="edit-input act-f-state">' + pickOptionsHtml('ActionItemState', a.stateRaw || '', true) + '</select>' +
    '<button class="act-save">Save</button>' +
    '<button class="act-cancel">Cancel</button>' +
    '</div>';
}
function actionsBlock(actions, parentId) {
  var head = '<div class="actions-head">' +
    '<span class="actions-label">Associated actions</span>' +
    '<button class="act-add" aria-label="Add action"><i class="ti ti-plus" aria-hidden="true"></i> Add</button>' +
    '</div>';
  var body = (actions && actions.length)
    ? actions.map(actionItemHtml).join('')
    : '<p class="action-empty">No actions recorded.</p>';
  return '<div class="actions-block" data-parent="' + esc(parentId || '') + '">' + head + body + '</div>';
}

/* Rebuild an action row's display from its data-* attributes (after cancel). */
function actionDataFromEl(el) {
  return {
    rawId:    el.getAttribute('data-action-id') || '',
    name:     el.getAttribute('data-name') || '',
    ownerRaw: el.getAttribute('data-owner') || '',
    assignee: el.getAttribute('data-owner-name') || '',
    dueDate:  el.getAttribute('data-due') || '',
    stateRaw: el.getAttribute('data-state') || ''
  };
}
function startActionEdit(el) {
  if (!el || el.querySelector('.action-edit')) return;
  el.innerHTML = actionFormHtml(actionDataFromEl(el));
  var ta = el.querySelector('.ta-input');
  if (ta) attachUserTypeahead(ta, el.querySelector('.ta-menu'));
  var n = el.querySelector('.act-f-name'); if (n) n.focus();
}
function cancelActionEdit(el) {
  if (!el) return;
  if (!el.getAttribute('data-action-id')) { el.remove(); return; }   /* discard new draft */
  var a = actionDataFromEl(el);
  el.innerHTML = actionInnerHtml(a.name, a.assignee, a.stateRaw, a.dueDate);
}
function addActionRow(addBtn) {
  var block = addBtn.closest('.actions-block');
  if (!block || block.querySelector('.action-edit')) return;
  var empty = block.querySelector('.action-empty');
  if (empty) empty.remove();
  var el = document.createElement('div');
  el.className = 'action-item';
  el.setAttribute('data-action-id', '');
  el.setAttribute('data-parent', block.getAttribute('data-parent') || '');
  el.innerHTML = actionFormHtml({});
  block.appendChild(el);
  var ta = el.querySelector('.ta-input');
  if (ta) attachUserTypeahead(ta, el.querySelector('.ta-menu'));
  var n = el.querySelector('.act-f-name'); if (n) n.focus();
}
function saveAction(el) {
  if (!el) return;
  var nameEl = el.querySelector('.act-f-name');
  var name = nameEl ? String(nameEl.value).trim() : '';
  if (!name) { alert('Action name is required.'); if (nameEl) nameEl.focus(); return; }
  var ownerEl = el.querySelector('.act-f-owner');
  var ownerId = ownerEl ? (ownerEl.getAttribute('data-user-id') || '') : '';
  var dueEl = el.querySelector('.act-f-due');
  var due = dueEl ? dueEl.value : '';
  var stateEl = el.querySelector('.act-f-state');
  var state = stateEl ? stateEl.value : '';

  var fields = { Name: name };
  if (ownerId) fields.EntityOwner = ownerId;
  if (due) fields.DueDate = due;
  if (state) fields.ActionItemState = state;

  var c;
  try { c = getContext(); } catch (e) { alert('Cannot save action: ' + e.message); return; }
  var btn = el.querySelector('.act-save');
  if (btn) { btn.disabled = true; btn.textContent = 'Saving…'; }

  var actionId = el.getAttribute('data-action-id');
  var p;
  if (actionId) {
    p = updateObject(c.base, c.sid, actionId, fields);
  } else {
    fields.Container = el.getAttribute('data-parent') || '';
    p = createObject(c.base, c.sid, 'ActionItem', fields);
  }
  p.then(function () { location.reload(); })
   .catch(function (err) {
     alert('Failed to save action: ' + err.message);
     if (btn) { btn.disabled = false; btn.textContent = 'Save'; }
   });
}
function deleteAction(el) {
  if (!el) return;
  var actionId = el.getAttribute('data-action-id');
  if (!actionId) { el.remove(); return; }
  if (!window.confirm('Delete this action?')) return;
  var c;
  try { c = getContext(); } catch (e) { alert('Cannot delete action: ' + e.message); return; }
  deleteObject(c.base, c.sid, actionId)
    .then(function () { location.reload(); })
    .catch(function (err) { alert('Failed to delete action: ' + err.message); });
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
  if (isDateField(field)) return '<span class="action-due ' + dueCls(raw) + '">' + fmtDate(raw) + '</span>';
  if (isUserField(field)) return esc(raw ? (USER_NAME[raw] || cleanLabel(raw)) : '—');
  if (isPillField(field)) return pill(cleanLabel(raw));
  if (isPick(field)) return esc(cleanLabel(raw));
  return esc(raw);
}

/* ---------- 11. Renderers ---------- */
function renderRisks(risks, actionMap) {
  var tbody = document.getElementById('risks-body');
  if (!risks.length) {
    tbody.innerHTML = '<tr><td colspan="10"><div class="no-data">No risks found for this project.</div></td></tr>';
    document.getElementById('badge-risks').textContent = '0';
    return;
  }
  tbody.innerHTML = risks.map(function (r) {
    var actions = actionMap[r.rawId] || [];
    var hm = riskRatingHeatMap(r.riskRating);
    return '<tr class="data-row" data-status="' + esc(r.status) + '">' +
      '<td><button class="expand-btn" data-target="' + r.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td>' + esc(r.sysId) + '</td>' +
      editCell('Title',            r.rawId, r.name,              esc(r.name)) +
      editCell('C_Likelihood',     r.rawId, r.probabilityRaw,    esc(r.probability)) +
      editCell('C_Impact',         r.rawId, r.impactRaw,         esc(r.impact)) +
      '<td><span class="heatmap ' + hm + '">' + esc(r.riskRating) + '</span></td>' +
      editCell('State',            r.rawId, r.statusRaw,         pill(r.status)) +
      editCell('Owner',            r.rawId, r.ownerRaw,          esc(r.owner)) +
      editCell('C_ReportingLevel', r.rawId, r.reportingLevelRaw, esc(r.reportingLevel)) +
      editCell('C_ImpactDate',     r.rawId, r.impactDate || '',  '<span class="action-due ' + dueCls(r.impactDate) + '">' + fmtDate(r.impactDate) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + r.id + '-actions"><td colspan="10">' + actionsBlock(actions, r.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-risks').textContent = risks.length;
}

function renderIssues(issues, actionMap) {
  var tbody = document.getElementById('issues-body');
  if (!issues.length) {
    tbody.innerHTML = '<tr><td colspan="8"><div class="no-data">No issues found for this project.</div></td></tr>';
    document.getElementById('badge-issues').textContent = '0';
    return;
  }
  tbody.innerHTML = issues.map(function (issue) {
    var actions = actionMap[issue.rawId] || [];
    return '<tr class="data-row" data-status="' + esc(issue.status) + '">' +
      '<td><button class="expand-btn" data-target="' + issue.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td>' + esc(issue.sysId) + '</td>' +
      editCell('Title',    issue.rawId, issue.name,        esc(issue.name)) +
      editCell('Priority', issue.rawId, issue.priorityRaw, pill(issue.priority)) +
      editCell('State',    issue.rawId, issue.statusRaw,   pill(issue.status)) +
      editCell('AssignedTo', issue.rawId, issue.ownerRaw,  esc(issue.owner)) +
      '<td>' + fmtDate(issue.raised) + '</td>' +
      editCell('DueDate',  issue.rawId, issue.dueDate || '', '<span class="action-due ' + dueCls(issue.dueDate) + '">' + fmtDate(issue.dueDate) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + issue.id + '-actions"><td colspan="8">' + actionsBlock(actions, issue.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-issues').textContent = issues.length;
}

function renderRequests(requests, actionMap) {
  var tbody = document.getElementById('requests-body');
  if (!requests.length) {
    tbody.innerHTML = '<tr><td colspan="7"><div class="no-data">No change requests found for this project.</div></td></tr>';
    document.getElementById('badge-requests').textContent = '0';
    return;
  }
  tbody.innerHTML = requests.map(function (req) {
    var actions = actionMap[req.rawId] || [];
    return '<tr class="data-row" data-status="' + esc(req.status) + '">' +
      '<td><button class="expand-btn" data-target="' + req.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td>' + esc(req.sysId) + '</td>' +
      editCell('Title',       req.rawId, req.name,    esc(req.name)) +
      editCell('State',       req.rawId, req.statusRaw, pill(req.status)) +
      '<td>' + esc(req.requestor) + '</td>' +
      '<td>' + fmtDate(req.submitted) + '</td>' +
      editCell('DueDate',     req.rawId, req.decisionBy || '', '<span class="action-due ' + dueCls(req.decisionBy) + '">' + fmtDate(req.decisionBy) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + req.id + '-actions"><td colspan="7">' + actionsBlock(actions, req.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-requests').textContent = requests.length;
}

/* Collect picklist options from loaded data, then render everything. */
function renderAll(risks, issues, requests, actionMap) {
  DATA = { risks: risks, issues: issues, requests: requests, actionMap: actionMap };
  PICK = {};
  risks.forEach(function (r) {
    addPick('C_Impact', r.impactRaw);
    addPick('C_Likelihood', r.probabilityRaw);
    addPick('State', r.statusRaw);
    addPick('C_ReportingLevel', r.reportingLevelRaw);
    rememberUser(r.ownerRaw, r.owner);
  });
  issues.forEach(function (i) { rememberUser(i.ownerRaw, i.owner); });
  issues.forEach(function (i) {
    addPick('Priority', i.priorityRaw);
    addPick('State', i.statusRaw);
  });
  requests.forEach(function (q) {
    addPick('State', q.statusRaw);
  });
  Object.keys(actionMap || {}).forEach(function (pid) {
    actionMap[pid].forEach(function (a) { addPick('ActionItemState', a.stateRaw); });
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

/* Delete an object: DELETE /V2.0/services/data/objects/{Type}/{id}. */
function deleteObject(base, sid, entityId) {
  var url = base.replace(/\/$/, '') + API_OBJECTS + '/' + String(entityId).replace(/^\//, '');
  return fetch(url, { method: 'DELETE', headers: { 'Authorization': 'Session ' + sid } })
    .then(function (res) {
      if (!res.ok) throw new Error('HTTP ' + res.status);
      return res.json().catch(function () { return {}; });
    });
}

/* RelatedWork ties a Case (Risk/Issue/Request) to a WorkItem (the project).
   Without it the new item is created but never appears in the project view. */
function linkToProject(base, sid, caseId, projId) {
  return createObject(base, sid, 'RelatedWork', {
    Case: caseId,
    WorkItem: projId
  });
}

function createEntity(entityType, fields, label) {
  var c;
  try { c = getContext(); } catch (e) { alert('Cannot create ' + label + ': ' + e.message); return Promise.reject(e); }
  fields.PlannedFor = c.projId;
  return createObject(c.base, c.sid, entityType, fields)
    .then(function (res) {
      var caseId = res && res.id;
      if (!caseId) { location.reload(); return; }   /* no id returned — nothing to link */
      return linkToProject(c.base, c.sid, caseId, c.projId)
        .then(function () { location.reload(); });
    })
    .catch(function (err) { alert('Failed to create ' + label + ': ' + err.message); throw err; });
}

/* ---------- 14b. Inline "new item" draft row ----------
   Clicking a "New …" button inserts an editable draft row at the top of the
   table (no pop-up). The user fills it in and clicks Save (or presses Enter). */
var NEW_ROW_DEFS = {
  Risk: {
    tbody: 'risks-body', label: 'risk',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'text',   field: 'Title', placeholder: 'Risk name' },
      { type: 'pick',   field: 'C_Likelihood' },            /* Probability */
      { type: 'pick',   field: 'C_Impact' },
      { type: 'static', html: '—' },                       /* Score (auto-calculated) */
      { type: 'pick',   field: 'State' },
      { type: 'user',   field: 'Owner' },
      { type: 'pick',   field: 'C_ReportingLevel' },
      { type: 'date',   field: 'C_ImpactDate', actions: true }
    ]
  },
  Issue: {
    tbody: 'issues-body', label: 'issue',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'text',   field: 'Title', placeholder: 'Issue name' },
      { type: 'pick',   field: 'Priority' },
      { type: 'pick',   field: 'State' },
      { type: 'user',   field: 'AssignedTo' },
      { type: 'static', html: '—' },
      { type: 'date',   field: 'DueDate', actions: true }
    ]
  },
  EnhancementRequest: {
    tbody: 'requests-body', label: 'change request',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'text',   field: 'Title', placeholder: 'Change request name' },
      { type: 'pick',   field: 'State' },
      { type: 'static', html: '—' },
      { type: 'static', html: '—' },
      { type: 'date',   field: 'DueDate', actions: true }
    ]
  }
};

function newRowCellHtml(cell) {
  if (cell.type === 'static') return '<td>' + cell.html + '</td>';
  if (cell.type === 'text') {
    return '<td><input type="text" class="new-input edit-input" data-field="' + cell.field +
      '" placeholder="' + esc(cell.placeholder || '') + '" /></td>';
  }
  if (cell.type === 'date') {
    var actions = cell.actions
      ? '<div class="new-row-actions">' +
          '<button class="new-save">Save</button>' +
          '<button class="new-cancel">Cancel</button>' +
        '</div>'
      : '';
    return '<td><input type="date" class="new-input edit-input" data-field="' + cell.field + '" />' + actions + '</td>';
  }
  if (cell.type === 'user') {
    return '<td class="ta-cell"><input type="text" class="new-input ta-input" autocomplete="off"' +
      ' data-field="' + cell.field + '" data-user-id="" placeholder="Type a name…" />' +
      '<div class="ta-menu" style="display:none"></div></td>';
  }
  /* pick — canonical list for Impact/Probability, else values present in data */
  return '<td><select class="new-input edit-input" data-field="' + cell.field + '">' +
    pickOptionsHtml(cell.field, '', true) +
    '</select></td>';
}

function openNewRow(entityType) {
  var def = NEW_ROW_DEFS[entityType];
  if (!def) return;
  var tbody = document.getElementById(def.tbody);
  if (!tbody) return;

  /* If a draft is already open, just refocus it. */
  var existing = tbody.querySelector('.new-row');
  if (existing) { var f = existing.querySelector('.new-input'); if (f) f.focus(); return; }

  /* Drop a "no data" placeholder row if present. */
  var noData = tbody.querySelector('.no-data');
  if (noData) { var ndRow = noData.closest('tr'); if (ndRow) ndRow.remove(); }

  var tr = document.createElement('tr');
  tr.className = 'new-row';
  tr.setAttribute('data-entity', entityType);
  tr.innerHTML = '<td><i class="ti ti-plus" style="font-size:13px;color:#8B3A62" aria-hidden="true"></i></td>' +
    def.cells.map(newRowCellHtml).join('');
  tbody.insertBefore(tr, tbody.firstChild);

  tr.querySelectorAll('.ta-input').forEach(function (inp) {
    var menu = inp.parentNode.querySelector('.ta-menu');
    if (menu) attachUserTypeahead(inp, menu);
  });

  var saveB = tr.querySelector('.new-save');
  var cancB = tr.querySelector('.new-cancel');
  if (saveB) saveB.addEventListener('click', function () { saveNewRow(tr); });
  if (cancB) cancB.addEventListener('click', function () { cancelNewRow(tr); });
  tr.addEventListener('keydown', function (e) {
    if (e.key === 'Enter')  { e.preventDefault(); saveNewRow(tr); }
    if (e.key === 'Escape') { e.preventDefault(); cancelNewRow(tr); }
  });

  var title = tr.querySelector('input[data-field="Title"]');
  if (title) title.focus();
}

function cancelNewRow(row) {
  if (row) row.remove();
}

function saveNewRow(row) {
  var entityType = row.getAttribute('data-entity');
  var def = NEW_ROW_DEFS[entityType];
  var fields = {};
  row.querySelectorAll('.new-input').forEach(function (inp) {
    var f = inp.getAttribute('data-field');
    var v = isUserField(f) ? (inp.getAttribute('data-user-id') || '') : inp.value;
    if (v) fields[f] = v;
  });
  /* New items on the Change Requests tab are always Change Requests. */
  if (entityType === 'EnhancementRequest' && !fields.RequestType) {
    var rt = changeRequestTypeRaw();
    if (rt) fields.RequestType = rt;
  }
  if (!fields.Title) {
    alert('Please enter a name.');
    var t = row.querySelector('input[data-field="Title"]');
    if (t) t.focus();
    return;
  }

  var btn = row.querySelector('.new-save');
  if (btn) { btn.disabled = true; btn.textContent = 'Saving…'; }

  createEntity(entityType, fields, def.label)
    .catch(function () { if (btn) { btn.disabled = false; btn.textContent = 'Save'; } });
}

function createNewRisk()    { openNewRow('Risk'); }
function createNewIssue()   { openNewRow('Issue'); }
function createNewRequest() { openNewRow('EnhancementRequest'); }

/* Resolve the real "/Type/Value" path for RequestType = Change Request, so new
   change requests are created with the right type (and show in this tab). Uses
   metadata first, then a value already present in the data, else a best guess. */
function changeRequestTypeRaw() {
  function findIn(list) {
    for (var i = 0; i < (list || []).length; i++) {
      var raw = (list[i] && list[i].raw !== undefined) ? list[i].raw : list[i];
      if (raw && cleanLabel(raw).toLowerCase() === 'change request') return raw;
    }
    return '';
  }
  return findIn(PICK_META.RequestType) || findIn(PICK.RequestType) || '/RequestType/Change Request';
}

/* ---------- 15. Inline editing ---------- */
function startEdit(cell) {
  if (cell.classList.contains('editing')) return;
  var field = cell.getAttribute('data-field');
  var value = cell.getAttribute('data-value') || '';
  var inputHtml;

  if (isUserField(field)) {
    var curName = value ? (USER_NAME[value] || cleanLabel(value)) : '';
    inputHtml = '<input type="text" class="edit-input ta-input" autocomplete="off"' +
      ' data-user-id="' + esc(value) + '" value="' + esc(curName) + '" placeholder="Type a name…" />' +
      '<div class="ta-menu" style="display:none"></div>';
  } else if (isPick(field)) {
    inputHtml = '<select class="edit-input">' + pickOptionsHtml(field, value, false) + '</select>';
  } else if (isDateField(field)) {
    inputHtml = '<input type="date" class="edit-input" value="' + (value ? value.split('T')[0] : '') + '" />';
  } else {
    inputHtml = '<input type="text" class="edit-input" value="' + esc(value) + '" />';
  }

  cell.classList.add('editing');
  cell.innerHTML = inputHtml +
    '<button class="edit-save">Save</button>' +
    '<button class="edit-cancel">Cancel</button>';
  var taInput = cell.querySelector('.ta-input');
  if (taInput) {
    attachUserTypeahead(taInput, cell.querySelector('.ta-menu'));
    taInput.focus();
  } else {
    var inp = cell.querySelector('.edit-input');
    if (inp) inp.focus();
  }
}

function cancelEdit(cell) {
  cell.classList.remove('editing');
  cell.innerHTML = cellDisplay(cell.getAttribute('data-field'), cell.getAttribute('data-value') || '');
}

function saveEdit(cell) {
  var input = cell.querySelector('.edit-input');
  if (!input) return;
  var field = cell.getAttribute('data-field');
  var id = cell.getAttribute('data-id');
  var newValue;
  if (isUserField(field)) {
    newValue = input.getAttribute('data-user-id') || '';
    if (!newValue) { alert('Please pick a user from the list.'); return; }
  } else {
    newValue = input.value;
    if (!newValue) { alert('Field cannot be empty'); return; }
  }

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
/* Per-tab config for the single shared toolbar (search / status / New). */
var TAB_CFG = {
  risks:    { tbody: 'risks-body',    placeholder: 'Search risks…',    newLabel: 'New risk',    create: createNewRisk,    statuses: ['New', 'Closed', 'Realised'] },
  issues:   { tbody: 'issues-body',   placeholder: 'Search issues…',   newLabel: 'New issue',   create: createNewIssue,   statuses: ['Open', 'In progress', 'Resolved'] },
  requests: { tbody: 'requests-body', placeholder: 'Search change requests…', newLabel: 'New change request', create: createNewRequest, statuses: ['New', 'Pending', 'Approved', 'Rejected'] }
};
var ACTIVE_TAB = 'risks';

/* Switch tabs and point the shared toolbar at the newly active tab. */
function setActiveTab(name) {
  if (!TAB_CFG[name]) return;
  ACTIVE_TAB = name;
  var cfg = TAB_CFG[name];

  document.querySelectorAll('.aw-tab').forEach(function (t) {
    var on = t.getAttribute('data-tab') === name;
    t.classList.toggle('active', on);
    t.setAttribute('aria-selected', on ? 'true' : 'false');
  });
  document.querySelectorAll('.aw-tab-pane').forEach(function (p) { p.classList.remove('active'); });
  var pane = document.getElementById('pane-' + name);
  if (pane) pane.classList.add('active');

  var search = document.getElementById('aw-search');
  if (search) { search.value = ''; search.placeholder = cfg.placeholder; }

  var filter = document.getElementById('aw-filter');
  if (filter) {
    filter.innerHTML = '<option value="">All statuses</option>' +
      cfg.statuses.map(function (s) { return '<option value="' + esc(s) + '">' + esc(s) + '</option>'; }).join('');
    filter.value = '';
  }

  var lbl = document.getElementById('btn-new-label');
  if (lbl) lbl.textContent = cfg.newLabel;

  /* Clear any prior search/status filtering on the now-visible tab. */
  document.querySelectorAll('#' + cfg.tbody + ' .data-row').forEach(function (row) { row.style.display = ''; });
}

function initUI() {
  document.querySelectorAll('.aw-tab[data-tab]').forEach(function (btn) {
    btn.addEventListener('click', function () { setActiveTab(btn.getAttribute('data-tab')); });
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
      /* Action item add / edit / delete / save / cancel */
      var aAdd = e.target.closest('.act-add');
      if (aAdd) { addActionRow(aAdd); return; }
      var aSave = e.target.closest('.act-save');
      if (aSave) { saveAction(aSave.closest('.action-item')); return; }
      var aCancel = e.target.closest('.act-cancel');
      if (aCancel) { cancelActionEdit(aCancel.closest('.action-item')); return; }
      var aEdit = e.target.closest('.act-edit');
      if (aEdit) { startActionEdit(aEdit.closest('.action-item')); return; }
      var aDel = e.target.closest('.act-del');
      if (aDel) { deleteAction(aDel.closest('.action-item')); return; }

      var saveBtn = e.target.closest('.edit-save');
      if (saveBtn) { saveEdit(saveBtn.closest('.editable')); return; }
      var cancelBtn = e.target.closest('.edit-cancel');
      if (cancelBtn) { cancelEdit(cancelBtn.closest('.editable')); return; }
      var editable = e.target.closest('.editable');
      if (editable && !editable.classList.contains('editing')) { startEdit(editable); return; }
    });
  });

  var search = document.getElementById('aw-search');
  if (search) search.addEventListener('input', function () { filterTable(TAB_CFG[ACTIVE_TAB].tbody, search.value); });

  var filter = document.getElementById('aw-filter');
  if (filter) filter.addEventListener('change', function () { filterByStatus(TAB_CFG[ACTIVE_TAB].tbody, filter.value); });

  var nb = document.getElementById('btn-new');
  if (nb) nb.addEventListener('click', function () { TAB_CFG[ACTIVE_TAB].create(); });

  var ex = document.getElementById('btn-export');
  if (ex) ex.addEventListener('click', exportToExcel);

  var imp = document.getElementById('btn-import');
  var impFile = document.getElementById('aw-import-file');
  if (imp && impFile) {
    imp.addEventListener('click', function () { impFile.value = ''; impFile.click(); });
    impFile.addEventListener('change', function () {
      if (impFile.files && impFile.files[0]) importFromExcel(impFile.files[0]);
    });
  }

  setActiveTab('risks');   /* initialise the toolbar for the default tab */
}

/* ---------- 17b. Export to Excel ----------
   Builds a styled multi-sheet SpreadsheetML 2003 workbook entirely in-browser
   (no external library / no CDN, so it works in locked-down environments).
   One sheet per tab (Risks / Issues / Requests); each record's action items are
   listed as rows beneath it. Formatting: bold white-on-burgundy headers, a
   bordered grid, Risk Rating cells filled with the on-screen heat-map colours,
   italic-grey action rows, and a frozen header row. */
function exportDateCell(iso) { return iso ? fmtDate(iso) : ''; }
function exportActionRow(a, len) {
  var row = ['', 'Action', a.name || '', a.assignee || '', a.status || '', exportDateCell(a.dueDate)];
  while (row.length < len) row.push('');
  return row.slice(0, len);
}
/* Each builder returns { rows: [[...]], kinds: ['record'|'action', ...] }. */
function exportRowsRisks() {
  var rows = [], kinds = [];
  DATA.risks.forEach(function (r) {
    rows.push([r.sysId, r.name, r.probability, r.impact, r.riskRating, r.status, r.owner, r.reportingLevel, exportDateCell(r.impactDate)]);
    kinds.push('record');
    (DATA.actionMap[r.rawId] || []).forEach(function (a) { rows.push(exportActionRow(a, 9)); kinds.push('action'); });
  });
  return { rows: rows, kinds: kinds };
}
function exportRowsIssues() {
  var rows = [], kinds = [];
  DATA.issues.forEach(function (i) {
    rows.push([i.sysId, i.name, i.priority, i.status, i.owner, exportDateCell(i.raised), exportDateCell(i.dueDate)]);
    kinds.push('record');
    (DATA.actionMap[i.rawId] || []).forEach(function (a) { rows.push(exportActionRow(a, 7)); kinds.push('action'); });
  });
  return { rows: rows, kinds: kinds };
}
function exportRowsRequests() {
  var rows = [], kinds = [];
  DATA.requests.forEach(function (q) {
    rows.push([q.sysId, q.name, q.status, q.requestor, exportDateCell(q.submitted), exportDateCell(q.decisionBy)]);
    kinds.push('record');
    (DATA.actionMap[q.rawId] || []).forEach(function (a) { rows.push(exportActionRow(a, 6)); kinds.push('action'); });
  });
  return { rows: rows, kinds: kinds };
}

/* SpreadsheetML helpers ------------------------------------------------- */
function xmlEsc(s) {
  return String(s == null ? '' : s)
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}
function xlCell(v, styleId) {
  return '<Cell' + (styleId ? ' ss:StyleID="' + styleId + '"' : '') +
    '><Data ss:Type="String">' + xmlEsc(v) + '</Data></Cell>';
}
/* Risk Rating value -> style id (mirrors the on-screen heat map). */
function exportHeatStyle(value) {
  var map = {
    'heatmap-very-low': 'hmVeryLow', 'heatmap-low': 'hmLow', 'heatmap-medium': 'hmMedium',
    'heatmap-high': 'hmHigh', 'heatmap-very-high': 'hmVeryHigh', 'heatmap-critical': 'hmCritical'
  };
  return map[riskRatingHeatMap(value)] || 'rec';   /* neutral/blank -> plain record cell */
}
function workbookStyles() {
  function border() {
    return '<Borders>' +
      '<Border ss:Position="Bottom" ss:LineStyle="Continuous" ss:Weight="1" ss:Color="#E0E0E0"/>' +
      '<Border ss:Position="Top" ss:LineStyle="Continuous" ss:Weight="1" ss:Color="#E0E0E0"/>' +
      '<Border ss:Position="Left" ss:LineStyle="Continuous" ss:Weight="1" ss:Color="#E0E0E0"/>' +
      '<Border ss:Position="Right" ss:LineStyle="Continuous" ss:Weight="1" ss:Color="#E0E0E0"/>' +
      '</Borders>';
  }
  function heat(id, color) {
    return '<Style ss:ID="' + id + '"><Font ss:Bold="1" ss:Color="#FFFFFF"/>' +
      '<Interior ss:Color="' + color + '" ss:Pattern="Solid"/>' +
      '<Alignment ss:Vertical="Center" ss:Horizontal="Center"/>' + border() + '</Style>';
  }
  return '<Styles>' +
    '<Style ss:ID="Default" ss:Name="Normal"><Alignment ss:Vertical="Center"/></Style>' +
    '<Style ss:ID="hdr"><Font ss:Bold="1" ss:Color="#FFFFFF"/>' +
      '<Interior ss:Color="#8B3A62" ss:Pattern="Solid"/><Alignment ss:Vertical="Center"/>' + border() + '</Style>' +
    '<Style ss:ID="rec"><Alignment ss:Vertical="Center"/>' + border() + '</Style>' +
    '<Style ss:ID="act"><Font ss:Italic="1" ss:Color="#5F6368"/>' +
      '<Interior ss:Color="#FAFBFC" ss:Pattern="Solid"/>' + border() + '</Style>' +
    heat('hmVeryLow', '#4CAF50') + heat('hmLow', '#8BC34A') + heat('hmMedium', '#FF9800') +
    heat('hmHigh', '#FF7043') + heat('hmVeryHigh', '#F4511E') + heat('hmCritical', '#D32F2F') +
    '</Styles>';
}
/* Build one <Worksheet>. heatCol = column to colour by Risk Rating (null = none). */
function buildSheetXml(name, header, built, heatCol) {
  var safe = String(name).replace(/[:\\\/?*\[\]]/g, ' ').slice(0, 31);
  var cols = header.map(function (h, i) {
    return '<Column ss:Width="' + (i === 0 ? 200 : 110) + '"/>';
  }).join('');
  var headerRow = '<Row>' + header.map(function (h) { return xlCell(h, 'hdr'); }).join('') + '</Row>';
  var bodyRows = built.rows.map(function (row, idx) {
    var kind = built.kinds[idx];
    return '<Row>' + row.map(function (val, c) {
      var sid = kind === 'action' ? 'act'
        : (heatCol != null && c === heatCol) ? exportHeatStyle(val)
        : 'rec';
      return xlCell(val, sid);
    }).join('') + '</Row>';
  }).join('');
  return '<Worksheet ss:Name="' + xmlEsc(safe) + '">' +
    '<Table>' + cols + headerRow + bodyRows + '</Table>' +
    '<WorksheetOptions xmlns="urn:schemas-microsoft-com:office:excel">' +
      '<FreezePanes/><FrozenNoSplit/><SplitHorizontal>1</SplitHorizontal>' +
      '<TopRowBottomPane>1</TopRowBottomPane><ActivePane>2</ActivePane>' +
    '</WorksheetOptions></Worksheet>';
}
function exportToExcel() {
  if (!DATA.risks.length && !DATA.issues.length && !DATA.requests.length) {
    alert('Nothing to export yet — the panel is still loading.');
    return;
  }
  var workbook =
    '<?xml version="1.0"?>\n<?mso-application progid="Excel.Sheet"?>\n' +
    '<Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet"' +
    ' xmlns:o="urn:schemas-microsoft-com:office:office"' +
    ' xmlns:x="urn:schemas-microsoft-com:office:excel"' +
    ' xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet">' +
    workbookStyles() +
    buildSheetXml('Risks',
      ['ID', 'Risk name', 'Probability', 'Impact', 'Score', 'Status', 'Owner', 'Reporting Level', 'Impact Date'],
      exportRowsRisks(), 4) +
    buildSheetXml('Issues',
      ['ID', 'Issue name', 'Priority', 'Status', 'Owner', 'Raised', 'Due date'],
      exportRowsIssues(), null) +
    buildSheetXml('Change Requests',
      ['ID', 'Request name', 'Status', 'Requestor', 'Submitted', 'Decision by'],
      exportRowsRequests(), null) +
    '</Workbook>';

  var blob = new Blob(['﻿' + workbook], { type: 'application/vnd.ms-excel' });
  var url = URL.createObjectURL(blob);
  var a = document.createElement('a');
  a.href = url;
  a.download = 'RAID_Export_' + new Date().toISOString().slice(0, 10) + '.xls';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  setTimeout(function () { URL.revokeObjectURL(url); }, 1000);
}

/* ---------- 17c. Import from Excel ----------
   Reads a workbook previously produced by Export (SpreadsheetML 2003 XML),
   matches rows back to records by the ID (SYSID) column, and updates changed
   fields / creates rows with no ID. Read-only columns (ID, Score, Requestor,
   Submitted, Raised) are ignored; empty cells are left untouched (never clears).
   Owner is matched by name, picklists by label, dates by parsing the shown date. */
var IMPORT_CFG = {
  'Risks': {
    entityType: 'Risk', list: 'risks',
    fields: [
      { hdr: 'Risk name',       api: 'Title',            kind: 'text' },
      { hdr: 'Probability',     api: 'C_Likelihood',     kind: 'pick' },
      { hdr: 'Impact',          api: 'C_Impact',         kind: 'pick' },
      { hdr: 'Status',          api: 'State',            kind: 'pick' },
      { hdr: 'Owner',           api: 'Owner',            kind: 'user' },
      { hdr: 'Reporting Level', api: 'C_ReportingLevel', kind: 'pick' },
      { hdr: 'Impact Date',     api: 'C_ImpactDate',     kind: 'date' }
    ]
  },
  'Issues': {
    entityType: 'Issue', list: 'issues',
    fields: [
      { hdr: 'Issue name', api: 'Title',      kind: 'text' },
      { hdr: 'Priority',   api: 'Priority',   kind: 'pick' },
      { hdr: 'Status',     api: 'State',      kind: 'pick' },
      { hdr: 'Owner',      api: 'AssignedTo', kind: 'user' },
      { hdr: 'Due date',   api: 'DueDate',    kind: 'date' }
    ]
  },
  'Change Requests': {
    entityType: 'EnhancementRequest', list: 'requests',
    fields: [
      { hdr: 'Request name', api: 'Title',   kind: 'text' },
      { hdr: 'Status',       api: 'State',   kind: 'pick' },
      { hdr: 'Decision by',  api: 'DueDate', kind: 'date' }
    ],
    createExtra: function (fields) { var rt = changeRequestTypeRaw(); if (rt) fields.RequestType = rt; }
  }
};

/* Current raw value of an API field on a loaded record (for change detection). */
function recCurrent(entityType, rec, api) {
  if (api === 'Title') return rec.name === '(unnamed)' ? '' : rec.name;
  if (entityType === 'Risk') {
    if (api === 'C_Likelihood')     return rec.probabilityRaw;
    if (api === 'C_Impact')         return rec.impactRaw;
    if (api === 'State')            return rec.statusRaw;
    if (api === 'Owner')            return rec.ownerRaw;
    if (api === 'C_ReportingLevel') return rec.reportingLevelRaw;
    if (api === 'C_ImpactDate')     return rec.impactDate || '';
  } else if (entityType === 'Issue') {
    if (api === 'Priority')   return rec.priorityRaw;
    if (api === 'State')      return rec.statusRaw;
    if (api === 'AssignedTo') return rec.ownerRaw;
    if (api === 'DueDate')    return rec.dueDate || '';
  } else if (entityType === 'EnhancementRequest') {
    if (api === 'State')   return rec.statusRaw;
    if (api === 'DueDate') return rec.decisionBy || '';
  }
  return '';
}

/* Convert a displayed cell value back to the raw API value.
   Resolves to: a value to send, '' (empty -> skip, never clears), or null (could
   not resolve -> skip + warn). Returns a Promise (user lookup may be async). */
function importConvert(field, kind, display) {
  display = (display == null ? '' : String(display)).trim();
  if (kind === 'text') return Promise.resolve(display);
  if (!display || display === '—') return Promise.resolve('');
  if (kind === 'date') {
    var d = new Date(display);
    return Promise.resolve(isNaN(d.getTime()) ? null : d.toISOString().slice(0, 10));
  }
  if (kind === 'pick') {
    var opts = pickOptionList(field, '');
    for (var i = 0; i < opts.length; i++) {
      if (String(opts[i].label).toLowerCase() === display.toLowerCase()) return Promise.resolve(opts[i].raw);
    }
    return Promise.resolve(null);
  }
  if (kind === 'user') {
    for (var id in USER_NAME) {
      if (USER_NAME[id] && USER_NAME[id].toLowerCase() === display.toLowerCase()) return Promise.resolve(id);
    }
    return searchUsers(display).then(function (list) {
      for (var j = 0; j < list.length; j++) {
        if (list[j].name.toLowerCase() === display.toLowerCase()) { rememberUser(list[j].id, list[j].name); return list[j].id; }
      }
      return null;
    }).catch(function () { return null; });
  }
  return Promise.resolve(display);
}

/* Parse a SpreadsheetML workbook into { sheetName: [ [cell,...], ... ] }. */
function parseWorkbook(text) {
  var doc = new DOMParser().parseFromString(text, 'application/xml');
  if (doc.getElementsByTagName('parsererror').length) throw new Error('not a valid Excel XML workbook');
  function wsName(ws) {
    var a = ws.attributes;
    for (var i = 0; i < a.length; i++) { if (a[i].name === 'ss:Name' || a[i].localName === 'Name') return a[i].value; }
    return '';
  }
  var out = {}, sheets = doc.getElementsByTagName('Worksheet');
  for (var s = 0; s < sheets.length; s++) {
    var rowsOut = [], rowEls = sheets[s].getElementsByTagName('Row');
    for (var r = 0; r < rowEls.length; r++) {
      var cells = rowEls[r].getElementsByTagName('Cell'), vals = [];
      for (var c = 0; c < cells.length; c++) {
        var data = cells[c].getElementsByTagName('Data')[0];
        vals.push(data ? (data.textContent || '') : '');
      }
      rowsOut.push(vals);
    }
    out[wsName(sheets[s])] = rowsOut;
  }
  return out;
}

function importFromExcel(file) {
  var reader = new FileReader();
  reader.onload = function () {
    var book;
    try { book = parseWorkbook(String(reader.result)); }
    catch (e) { alert('Could not read the file: ' + e.message); return; }
    processImport(book);
  };
  reader.onerror = function () { alert('Could not read the file.'); };
  reader.readAsText(file);
}

function processImport(book) {
  var ops = [], warnings = [], pending = [];

  Object.keys(IMPORT_CFG).forEach(function (sheetName) {
    var cfg = IMPORT_CFG[sheetName];
    var rows = book[sheetName];
    if (!rows || rows.length < 2) return;

    var colOf = {};
    rows[0].forEach(function (h, idx) { colOf[String(h).trim()] = idx; });
    var idCol = colOf.ID;

    var bySys = {};
    (DATA[cfg.list] || []).forEach(function (rec) { if (rec.sysId) bySys[String(rec.sysId)] = rec; });

    for (var r = 1; r < rows.length; r++) {
      var row = rows[r];
      if ((row[0] || '') === '' && (row[1] || '') === 'Action') continue;   /* action sub-row */

      (function (row, rowNum) {
        var idVal = (idCol != null) ? String(row[idCol] || '').trim() : '';
        var rec = idVal ? bySys[idVal] : null;
        var proms = cfg.fields.map(function (f) {
          var ci = colOf[f.hdr];
          var disp = (ci != null) ? row[ci] : '';
          return importConvert(f.api, f.kind, disp).then(function (val) { return { f: f, val: val, disp: disp }; });
        });
        pending.push(Promise.all(proms).then(function (resolved) {
          var fields = {};
          resolved.forEach(function (rv) {
            if (rv.val === null) { warnings.push(sheetName + ' row ' + rowNum + ': could not resolve "' + rv.disp + '" for ' + rv.f.hdr); return; }
            if (rv.val === '') return;                       /* empty -> leave untouched */
            if (rec) {
              var cur = recCurrent(cfg.entityType, rec, rv.f.api);
              var curCmp = (rv.f.kind === 'date') ? String(cur).slice(0, 10) : cur;
              if (String(curCmp) === String(rv.val)) return;  /* unchanged */
            }
            fields[rv.f.api] = rv.val;
          });
          if (rec) {
            if (Object.keys(fields).length) ops.push({ kind: 'update', rawId: rec.rawId, fields: fields });
          } else if (idVal) {
            warnings.push(sheetName + ' row ' + rowNum + ': ID "' + idVal + '" not found — skipped.');
          } else if (fields.Title) {
            if (cfg.createExtra) cfg.createExtra(fields);
            ops.push({ kind: 'create', entityType: cfg.entityType, fields: fields });
          }
        }));
      })(row, r + 1);
    }
  });

  Promise.all(pending).then(function () {
    var upd = ops.filter(function (o) { return o.kind === 'update'; }).length;
    var cre = ops.filter(function (o) { return o.kind === 'create'; }).length;
    if (!upd && !cre) {
      alert('No changes to import.' + (warnings.length ? '\n\nNotes:\n' + warnings.slice(0, 15).join('\n') : ''));
      return;
    }
    var msg = 'Import will update ' + upd + ' and create ' + cre + ' record(s).';
    if (warnings.length) msg += '\n\nSkipped / warnings (' + warnings.length + '):\n' + warnings.slice(0, 12).join('\n');
    msg += '\n\nProceed?';
    if (window.confirm(msg)) executeImport(ops);
  });
}

function executeImport(ops) {
  var c;
  try { c = getContext(); } catch (e) { alert('Import failed: ' + e.message); return; }
  var jobs = ops.map(function (op) {
    var p;
    if (op.kind === 'update') {
      p = updateObject(c.base, c.sid, op.rawId, op.fields);
    } else {
      op.fields.PlannedFor = c.projId;
      p = createObject(c.base, c.sid, op.entityType, op.fields).then(function (res) {
        var caseId = res && res.id;
        return caseId ? linkToProject(c.base, c.sid, caseId, c.projId) : null;
      });
    }
    return p.then(function () { return true; }, function () { return false; });
  });
  Promise.all(jobs).then(function (results) {
    var failed = results.filter(function (ok) { return !ok; }).length;
    if (failed) alert(failed + ' of ' + results.length + ' operation(s) failed; the rest were applied.');
    location.reload();
  });
}

/* ---------- 18. Bootstrap ---------- */
setTimeout(function () {
  initUI();

  var c;
  try {
    c = getContext();
  } catch (e) {
    showError('risks-body', 10, e.message);
    showError('issues-body', 8, e.message);
    showError('requests-body', 7, e.message);
    return;
  }

  showLoading('risks-body', 10);
  showLoading('issues-body', 8);
  showLoading('requests-body', 7);

  /* Load real picklist option paths from metadata (non-blocking). */
  loadPicklistMeta().catch(function (err) {
    console.warn('Picklist metadata unavailable, falling back to data-derived options:', err && err.message);
  });

  var qRisks =
    "SELECT SYSID, Title, C_Likelihood, C_Impact, C_RiskRating, State, Owner.Name, " +
    "C_ReportingLevel, C_ImpactDate FROM Risk WHERE PlannedFor = '" + c.projId + "'";
  var qIssues =
    "SELECT SYSID, Title, Priority, State, AssignedTo.Name, CreatedOn, DueDate " +
    "FROM Issue WHERE PlannedFor = '" + c.projId + "'";
  var qRequests =
    "SELECT SYSID, Title, RequestType, State, CreatedBy.Name, CreatedOn, DueDate " +
    "FROM EnhancementRequest WHERE PlannedFor = '" + c.projId + "'";

  Promise.all([
    czql(c.base, c.sid, qRisks),
    czql(c.base, c.sid, qIssues),
    czql(c.base, c.sid, qRequests)
  ])
  .then(function (results) {
    var risks    = results[0].map(toRisk);
    var issues   = results[1].map(toIssue);
    /* Requests tab shows Change Requests only (RequestType = 'Change Request'). */
    var reqRows  = results[2].filter(function (e) {
      return cleanLabel(pickRaw(e.RequestType) || pickRaw(e.Type)).toLowerCase() === 'change request';
    });
    var requests = reqRows.map(toRequest);

    var caseIds = []
      .concat(results[0].map(function (e) { return e.id; }))
      .concat(results[1].map(function (e) { return e.id; }))
      .concat(reqRows.map(function (e) { return e.id; }));

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
    showError('risks-body', 10, msg);
    showError('issues-body', 8, msg);
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
| New item created but not linked to the project | After creating the Risk/Issue/Request, a `RelatedWork` record is created (`Case` = new item id, `WorkItem` = project id) to tie it to the project view |
| "New …" used an ugly `prompt()` pop-up | Clicking a "New …" button now inserts an inline editable draft row at the top of the table (name + picklists + due date, Save/Cancel, Enter to save / Esc to cancel) |
| Impact / Probability dropdowns only listed values already in use | Canonical, fully-ordered option lists for `C_Impact` (1 - Minor … 6 - Severe) and `C_Likelihood` (1 - Unlikely … 6 - Very Likely) always show; raw `/Type/Value` path is reused from data or rebuilt from the detected prefix. "Likelihood" column header renamed to "Probability" |
| Owner was read-only text | Owner (`AssignedTo`) is now an editable **type-ahead** of system users — searches server-side (`User WHERE Name LIKE '%…%'`, debounced, 2+ chars) instead of bulk-loading everyone. Works on existing rows and the new-item draft row; sends the `/User/…` id |
```
