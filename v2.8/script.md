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
var API_QUERY     = '/V2.0/services/data/query';
var API_OBJECTS   = '/V2.0/services/data/objects';
var API_CUSTOMACT = '/V2.0/services/data/executeCustomAction';

/* Risks tab shows only these lifecycle states (others, e.g. Realised, are
   hidden — a Risk becomes Realised once converted to an Issue). */
var RISK_STATES = ['Opened', 'Closed'];
function isShownRiskState(label) {
  return RISK_STATES.some(function (s) { return s.toLowerCase() === String(label || '').toLowerCase(); });
}

/* Name of the Risk custom action that flips the Risk to Realised and spawns a
   linked Issue. If the action's API name differs from its display name, change
   this one string. */
var CREATE_ISSUE_ACTION = 'Create Issue';

/* Reporting Level field API name on the RISK entity — DIFFERS from Issue's.
   Confirmed on eu.clarizentb.com: Risk field = C_ReportingLevelR (picklist class
   C_RiskReportingLevelR); Issue field = C_ReportingLevel (class
   C_IssueReportingLevel). Set to '' to disable for Risk — the Risks "Reporting
   Level" column then shows read-only "—" and is omitted from the risk query /
   create / edit / import. */
var RISK_REPORTING_FIELD = 'C_ReportingLevelR';

/* Impact field API name on the RISK entity — DIFFERS from the panel's original
   C_Impact. Confirmed on eu.clarizentb.com: Risk Impact field = C_ImpactR
   (picklist class C_RiskImpactR). (Issue Impact is C_IssueImpact ->
   C_IssueIssueImpact; Risk Probability is C_Likelihood -> C_RiskLikelihood.) */
var RISK_IMPACT_FIELD = 'C_ImpactR';

/* Impact Date field API name on the ISSUE entity — DIFFERS from Risk's.
   Confirmed on eu.clarizentb.com: Issue = C_IssueImpactDate; Risk = C_ImpactDate.
   Set to '' to disable for Issue (column shows read-only "—" and it's omitted
   from the issue query / create / edit / import). */
var ISSUE_IMPACTDATE_FIELD = 'C_IssueImpactDate';

/* Latest loaded data, retained so "Export to Excel" can use it. */
var DATA = { risks: [], issues: [], requests: [], actionMap: {} };

/* Picklist option lists, collected from the loaded data (field -> [rawPath]). */
var PICK = {};
/* Authoritative picklist options fetched from AdaptiveWork metadata
   (field -> [{ raw, label }]). Preferred over guessing a "/Type" prefix. */
var PICK_META = {};
var PICK_FIELDS = ['C_Impact', 'C_ImpactR', 'C_Likelihood', 'C_IssueImpact', 'State', 'Priority', 'RequestType', 'C_ReportingLevel', 'C_ReportingLevelR', 'ActionItemState'];
function isPick(f) { return PICK_FIELDS.indexOf(f) > -1; }

/* Status-like fields rendered as a coloured pill; date fields as a formatted date. */
function isPillField(f) { return f === 'State' || f === 'Priority'; }
function isDateField(f) { return f === 'DueDate' || f === 'C_ImpactDate' || (!!ISSUE_IMPACTDATE_FIELD && f === ISSUE_IMPACTDATE_FIELD); }
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
function attachUserTypeahead(input, menu, onPick) {
  var timer = null;
  function close() { menu.style.display = 'none'; menu.innerHTML = ''; }
  /* Position the menu with viewport-fixed coordinates anchored to the input, so
     it floats above the table's overflow container instead of being clipped
     (which also stops a stray scrollbar appearing on that container). */
  function place() {
    var r = input.getBoundingClientRect();
    menu.style.position = 'fixed';
    menu.style.left = r.left + 'px';
    menu.style.top = (r.bottom + 2) + 'px';
    menu.style.minWidth = r.width + 'px';
    menu.style.zIndex = '1000';
  }
  function pick(item) {
    input.value = item.getAttribute('data-name');
    input.setAttribute('data-user-id', item.getAttribute('data-id'));
    rememberUser(item.getAttribute('data-id'), item.getAttribute('data-name'));
    close();
    if (onPick) onPick();
  }
  function render(list) {
    if (!list.length) { menu.innerHTML = '<div class="ta-empty">No matches</div>'; menu.style.display = 'block'; place(); return; }
    menu.innerHTML = list.map(function (u) {
      return '<div class="ta-item" data-id="' + esc(u.id) + '" data-name="' + esc(u.name) + '">' + esc(u.name) + '</div>';
    }).join('');
    menu.style.display = 'block';
    place();
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
  C_ImpactR:        ['1 Minor', '2 Moderate', '3 Moderate+', '4 Significant', '5 Significant+', '6 Severe'],
  C_IssueImpact:    ['1 - Minor', '2 - Moderate', '3 - Moderate +', '4 - Significant', '5 - Significant +', '6 - Severe'],
  C_Likelihood:     ['1 - Unlikely', '2 - Possible', '3 - Possible +', '4 - Likely', '5 - Likely +', '6 - Very Likely'],
  C_ReportingLevel:  ['1 - EFDC', '2 - Portfolio', '3 - Project Board', '4 - Project Team'],
  C_ReportingLevelR: ['1 - EFDC', '2 - Portfolio', '3 - Project Board', '4 - Project Team']
};

/* Custom picklists are their own entities whose INSTANCES are the values, so we
   read the current environment's real options with a plain query against the
   picklist type (its "Class API Name"). Each row's id is the exact "/Type/Value"
   path to save; its Name is the label. Map: field -> CANDIDATE type entity names
   (tried in order; first one that exists + returns values wins).
   All confirmed from each field's "Class API Name" on eu.clarizentb.com:
     C_Impact -> C_RiskImpact, C_Likelihood -> C_RiskProbability,
     C_IssueImpact -> C_IssueIssueImpact,
     C_ReportingLevel (Issue) -> C_IssueReportingLevel,
     C_ReportingLevelR (Risk) -> C_RiskReportingLevelR.
   (State / Priority / RequestType / ActionItemState are system enums, sourced
   from the loaded data instead, so they aren't listed here.) */
var PICK_LIST_TYPES = {
  C_ImpactR:         ['C_RiskImpactR'],
  C_Likelihood:      ['C_RiskLikelihood'],
  C_IssueImpact:     ['C_IssueIssueImpact'],
  C_ReportingLevel:  ['C_IssueReportingLevel'],
  C_ReportingLevelR: ['C_RiskReportingLevelR']
};

/* Build [{ raw, label }] options for a picklist field — ENVIRONMENT-DRIVEN.
   We never fabricate a "/Type/Value" path. Those paths differ per environment
   (they weren't migrated 1:1), and sending a made-up path fails to save with a
   500. The real, saveable option paths come only from:
     1. AdaptiveWork metadata (describeEntities) for THIS environment, else
     2. the distinct raw values actually present in the loaded data.
   PICK_OPTIONS is used ONLY to order/relabel options that genuinely exist here —
   it can never introduce a value the environment doesn't have. `current` (the
   row's own raw value) is always kept selectable, even if it is a stale or
   unmigrated path, so the cell still shows and the user can pick a valid one. */
function pickOptionList(field, current) {
  var out = [], seenRaw = {};
  function push(raw, label) {
    if (raw === null || raw === undefined || raw === '') return;
    if (seenRaw[raw]) return;
    seenRaw[raw] = 1;
    out.push({ raw: raw, label: label || cleanLabel(raw) });
  }

  /* Real, saveable options for this environment (metadata preferred). */
  var real = [];
  var meta = PICK_META[field];
  if (meta && meta.length) {
    real = meta.slice();
  } else {
    (PICK[field] || []).forEach(function (r) { real.push({ raw: r, label: cleanLabel(r) }); });
  }

  /* Optional canonical ordering, matched by label — real options only. */
  var canon = PICK_OPTIONS[field];
  if (canon && real.length) {
    var byLabel = {};
    real.forEach(function (o) { byLabel[cleanLabel(o.raw).toLowerCase()] = o; });
    canon.forEach(function (lbl) {
      var m = byLabel[lbl.toLowerCase()];
      if (m) push(m.raw, lbl);
    });
  }
  real.forEach(function (o) { push(o.raw, o.label); });   /* everything real, in order */

  if (current) push(current, cleanLabel(current));        /* keep the row's own value */
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

/* Load real option values for the custom picklists by querying each picklist
   type entity (its instances ARE the values). Populates PICK_META[field] with
   the exact env "/Type/Value" ids + labels — so dropdowns and saves use values
   that actually exist here, even when no record uses them yet. Per-field and
   best-effort: one type failing (or being empty) doesn't block the others, and
   a summary is logged so an empty picklist (not configured in this env) is
   visible. Does not overwrite options already resolved from metadata. */
function loadPicklistValues() {
  var c = getContext();
  var jobs = Object.keys(PICK_LIST_TYPES).map(function (field) {
    var candidates = PICK_LIST_TYPES[field];
    var notes = [];
    /* Try each candidate type in turn; stop at the first that returns values. */
    function tryNext(i) {
      if (i >= candidates.length) {
        return { field: field, type: candidates.join(' | '), error: notes.join('; ') || 'no candidates' };
      }
      var type = candidates[i];
      return czql(c.base, c.sid, 'SELECT Name FROM ' + type).then(
        function (rows) {
          if (rows && rows.length) {
            if (!PICK_META[field]) {
              PICK_META[field] = rows.map(function (r) { return { raw: r.id, label: r.Name || cleanLabel(r.id) }; });
            }
            return { field: field, type: type, count: rows.length };
          }
          notes.push(type + ': 0');
          return tryNext(i + 1);
        },
        function (err) {
          var m = (err && err.message) || String(err);
          notes.push(type + ': ' + (m.indexOf('InvalidType') > -1 ? 'InvalidType' : m.slice(0, 60)));
          return tryNext(i + 1);
        }
      );
    }
    return Promise.resolve().then(function () { return tryNext(0); });
  });
  return Promise.all(jobs).then(function (results) {
    console.info('[RAID panel] picklist values:', results);
    results.forEach(function (r) {
      var sample = (PICK_META[r.field] && PICK_META[r.field][0]) ? PICK_META[r.field][0].raw : '?';
      diag('picklist ' + r.field + ' [' + r.type + ']: ' +
        (r.error ? 'none — tried ' + r.error : (r.count + ' value(s); sends e.g. ' + sample)));
    });
    return results;
  });
}

/* ---------- 2. CZQL fetch wrapper ---------- */
/* Build a detailed error from a failed API response. Reads the response BODY —
   which is where the real cause lives: a Clarizen JSON error (e.g. errorCode
   "FieldNotFound"/"EntityNotFound") OR a WAF/gateway block page (e.g. Akamai
   "Access Denied / Reference #18.xxxx"). Also shows the endpoint host so we can
   confirm the eu.->apie. mapping resolved correctly. */
function apiError(what, res, body, url) {
  var host = '';
  try { host = String(url).split('/')[2] || ''; } catch (e) {}
  var snippet = String(body || '').replace(/\s+/g, ' ').trim().slice(0, 400);
  return new Error(what + ' failed: HTTP ' + res.status + (host ? ' @ ' + host : '') +
    (snippet ? ' — ' + snippet : ' (empty response body)'));
}

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
    if (!res.ok) return res.text().then(function (body) { throw apiError('Query', res, body, url); });
    return res.json();
  })
  .then(function (json) {
    if (json && json.errorCode) throw new Error(json.message || json.errorCode);
    return (json && json.entities) || [];
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
  var impRaw  = pickRaw(e[RISK_IMPACT_FIELD]);
  var probRaw = pickRaw(e.C_Likelihood);
  var rateRaw = pickRaw(e.C_RiskRating);
  var stRaw   = pickRaw(e.State);
  var rlRaw   = RISK_REPORTING_FIELD ? pickRaw(e[RISK_REPORTING_FIELD]) : '';
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
  var impRaw = pickRaw(e.C_IssueImpact);
  var stRaw  = pickRaw(e.State);
  var rlRaw  = pickRaw(e.C_ReportingLevel);
  return {
    id:                (e.id || '').replace(/\//g, '-'),
    sysId:             e.SYSID || '',
    name:              e.Title || '(unnamed)',
    impactRaw:         impRaw, impact: cleanLabel(impRaw) || '—',
    score:             cleanLabel(pickRaw(e.C_IssueScore)) || '—',
    statusRaw:         stRaw,  status: cleanLabel(stRaw) || 'Open',
    ownerRaw:          (e.Owner && e.Owner.id) || '',
    owner:             (e.Owner && e.Owner.Name) || '—',
    reportingLevelRaw: rlRaw,  reportingLevel: cleanLabel(rlRaw) || '—',
    impactDate:        ISSUE_IMPACTDATE_FIELD ? (e[ISSUE_IMPACTDATE_FIELD] || null) : null,
    rawId:             e.id || ''
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
/* Today as yyyy-mm-dd, for pre-filling <input type="date"> on new rows. */
function todayISO() {
  var d = new Date();
  var m = String(d.getMonth() + 1); if (m.length < 2) m = '0' + m;
  var day = String(d.getDate());    if (day.length < 2) day = '0' + day;
  return d.getFullYear() + '-' + m + '-' + day;
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
  Open: 'pill-open', Opened: 'pill-open', 'In progress': 'pill-inprog', Active: 'pill-inprog',
  Closed: 'pill-resolved', Resolved: 'pill-resolved', Complete: 'pill-resolved', Completed: 'pill-resolved',
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
  var stateLbl = cleanLabel(stateRaw);
  var statePill = stateLbl ? '<span class="pill ' + (PILL_MAP[stateLbl] || 'pill-medium') + '">' + esc(stateLbl) + '</span>' : '';
  return '<i class="ti ti-circle-check" style="font-size:14px;color:' + iconCol + '" aria-hidden="true"></i>' +
    '<span class="action-name">' + esc(name) + '</span>' +
    '<span class="action-meta">' + esc(ownerName && ownerName !== '—' ? ownerName : '—') + '</span>' +
    statePill +
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

/* ---------- 10b. Detail-page deep link ----------
   AdaptiveWork detail pages use the Clarizen URL scheme on the app host:
   https://<appHost>/Clarizen/<EntityType>/<id>. The record's rawId is already
   "/Risk/<id>", so we just prefix it. Opens in a new tab. */
function detailUrl(rawId) {
  if (!rawId) return '';
  return 'https://' + window.location.hostname + '/Clarizen/' + String(rawId).replace(/^\//, '');
}
function idLink(rawId, sysId) {
  var label = esc(sysId || '—');
  var url = detailUrl(rawId);
  if (!url) return label;
  return '<a class="id-link" href="' + esc(url) + '" target="_blank" rel="noopener" ' +
    'title="Open in AdaptiveWork">' + label + '</a>';
}

/* ---------- 11. Renderers ---------- */
function renderRisks(risks, actionMap) {
  var tbody = document.getElementById('risks-body');
  if (!risks.length) {
    tbody.innerHTML = '<tr><td colspan="11"><div class="no-data">No risks found for this project.</div></td></tr>';
    document.getElementById('badge-risks').textContent = '0';
    return;
  }
  tbody.innerHTML = risks.map(function (r) {
    var actions = actionMap[r.rawId] || [];
    var hm = riskRatingHeatMap(r.riskRating);
    return '<tr class="data-row" data-status="' + esc(r.status) + '">' +
      '<td><button class="expand-btn" data-target="' + r.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td>' + idLink(r.rawId, r.sysId) + '</td>' +
      editCell('Title',            r.rawId, r.name,              esc(r.name)) +
      editCell('C_Likelihood',     r.rawId, r.probabilityRaw,    esc(r.probability)) +
      editCell(RISK_IMPACT_FIELD,  r.rawId, r.impactRaw,         esc(r.impact)) +
      '<td><span class="heatmap ' + hm + '">' + esc(r.riskRating) + '</span></td>' +
      editCell('State',            r.rawId, r.statusRaw,         pill(r.status)) +
      editCell('Owner',            r.rawId, r.ownerRaw,          esc(r.owner)) +
      (RISK_REPORTING_FIELD
        ? editCell(RISK_REPORTING_FIELD, r.rawId, r.reportingLevelRaw, esc(r.reportingLevel))
        : '<td>' + esc(r.reportingLevel || '—') + '</td>') +
      editCell('C_ImpactDate',     r.rawId, r.impactDate || '',  '<span class="action-due ' + dueCls(r.impactDate) + '">' + fmtDate(r.impactDate) + '</span>') +
      '<td class="row-action-cell" data-id="' + esc(r.rawId) + '" data-name="' + esc(r.name) + '">' +
        createIssueButtonHtml() +
      '</td>' +
      '</tr>' +
      '<tr class="actions-row" id="' + r.id + '-actions"><td colspan="11">' + actionsBlock(actions, r.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-risks').textContent = risks.length;
}

function renderIssues(issues, actionMap) {
  var tbody = document.getElementById('issues-body');
  if (!issues.length) {
    tbody.innerHTML = '<tr><td colspan="9"><div class="no-data">No issues found for this project.</div></td></tr>';
    document.getElementById('badge-issues').textContent = '0';
    return;
  }
  tbody.innerHTML = issues.map(function (issue) {
    var actions = actionMap[issue.rawId] || [];
    var hm = riskRatingHeatMap(issue.score);
    return '<tr class="data-row" data-status="' + esc(issue.status) + '">' +
      '<td><button class="expand-btn" data-target="' + issue.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td>' + idLink(issue.rawId, issue.sysId) + '</td>' +
      editCell('Title',            issue.rawId, issue.name,              esc(issue.name)) +
      editCell('C_IssueImpact',    issue.rawId, issue.impactRaw,         esc(issue.impact)) +
      '<td><span class="heatmap ' + hm + '">' + esc(issue.score) + '</span></td>' +
      editCell('State',            issue.rawId, issue.statusRaw,         pill(issue.status)) +
      editCell('Owner',            issue.rawId, issue.ownerRaw,          esc(issue.owner)) +
      editCell('C_ReportingLevel', issue.rawId, issue.reportingLevelRaw, esc(issue.reportingLevel)) +
      (ISSUE_IMPACTDATE_FIELD
        ? editCell(ISSUE_IMPACTDATE_FIELD, issue.rawId, issue.impactDate || '', '<span class="action-due ' + dueCls(issue.impactDate) + '">' + fmtDate(issue.impactDate) + '</span>')
        : '<td><span class="action-due">' + fmtDate(issue.impactDate) + '</span></td>') +
      '</tr>' +
      '<tr class="actions-row" id="' + issue.id + '-actions"><td colspan="9">' + actionsBlock(actions, issue.rawId) + '</td></tr>';
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
      '<td>' + idLink(req.rawId, req.sysId) + '</td>' +
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
    addPick(RISK_IMPACT_FIELD, r.impactRaw);
    addPick('C_Likelihood', r.probabilityRaw);
    addPick('State', r.statusRaw);
    if (RISK_REPORTING_FIELD) addPick(RISK_REPORTING_FIELD, r.reportingLevelRaw);
    rememberUser(r.ownerRaw, r.owner);
  });
  issues.forEach(function (i) {
    addPick('C_IssueImpact', i.impactRaw);
    addPick('State', i.statusRaw);
    addPick('C_ReportingLevel', i.reportingLevelRaw);
    rememberUser(i.ownerRaw, i.owner);
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

  /* DIAGNOSTIC: show the exact value format an existing record already stores,
     so we can match the create/save payload to what each field really expects. */
  if (risks[0]) diag('sample Risk raw: prob=' + (risks[0].probabilityRaw || '(empty)') +
    '  impact=' + (risks[0].impactRaw || '(empty)') + '  reporting=' + (risks[0].reportingLevelRaw || '(empty)'));
  if (issues[0]) diag('sample Issue raw: impact=' + (issues[0].impactRaw || '(empty)') +
    '  reporting=' + (issues[0].reportingLevelRaw || '(empty)'));
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

/* On-screen diagnostics — because console.info is often hidden by the console's
   level filter (esp. on locked-down machines). Renders a dismissible box at the
   top of the panel so load/picklist results are readable (and pasteable). */
var DIAG_LINES = [];
function diag(line) {
  DIAG_LINES.push(line);
  var box = document.getElementById('aw-diag');
  if (!box) {
    box = document.createElement('div');
    box.id = 'aw-diag';
    box.style.cssText = 'margin:8px 12px;padding:8px 24px 8px 10px;border:1px solid #dadce0;' +
      'border-radius:4px;background:#fffbe6;color:#202124;font:12px/1.45 monospace;' +
      'white-space:pre-wrap;position:relative;';
    var close = document.createElement('button');
    close.textContent = '×';
    close.title = 'Dismiss diagnostics';
    close.setAttribute('aria-label', 'Dismiss diagnostics');
    close.style.cssText = 'position:absolute;top:2px;right:6px;border:none;background:none;' +
      'font-size:16px;line-height:1;cursor:pointer;color:#5f6368;';
    close.addEventListener('click', function () { box.parentNode && box.parentNode.removeChild(box); });
    var body = document.createElement('div');
    body.id = 'aw-diag-body';
    box.appendChild(close);
    box.appendChild(body);
    var panel = document.querySelector('.aw-panel') || document.body;
    panel.insertBefore(box, panel.firstChild);
  }
  var bodyEl = document.getElementById('aw-diag-body');
  if (bodyEl) bodyEl.textContent = 'RAID panel diagnostics\n' + DIAG_LINES.join('\n');
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
    if (!res.ok) return res.text().then(function (b) { throw apiError('Create', res, b, url); });
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
    if (!res.ok) return res.text().then(function (b) { throw apiError('Update', res, b, url); });
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
      if (!res.ok) return res.text().then(function (b) { throw apiError('Delete', res, b, url); });
      return res.json().catch(function () { return {}; });
    });
}

/* Run a custom (workflow) action on an entity:
   POST /V2.0/services/data/executeCustomAction
   body = { targetId: '/Risk/..', customAction: 'Create Issue', values?: [{fieldName,value}] } */
function executeCustomAction(base, sid, targetId, actionName, values) {
  var url = base.replace(/\/$/, '') + API_CUSTOMACT;
  var body = { targetId: targetId, customAction: actionName };
  if (values && values.length) body.values = values;
  return fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': 'Session ' + sid },
    body: JSON.stringify(body)
  })
  .then(function (res) {
    if (!res.ok) return res.text().then(function (b) { throw apiError('Custom action', res, b, url); });
    return res.json().catch(function () { return {}; });
  });
}

/* Row-level "Create Issue" action on a Risk — runs the custom action (which sets
   the Risk to Realised and creates a linked Issue), then reloads so the now-
   Realised Risk drops out of the list and the new Issue appears.

   No browser dialog: the bug button arms an inline ✓/✗ confirm in the cell.
   The Risk id/name live on the enclosing .row-action-cell. */
function createIssueButtonHtml() {
  return '<button class="row-action create-issue"' +
    ' title="Create Issue — sets this Risk to Realised and creates a linked Issue"' +
    ' aria-label="Create Issue from this risk"><i class="ti ti-bug" aria-hidden="true"></i></button>';
}
/* First click: swap the bug button for an inline confirm (✓) / cancel (✗) pair. */
function armCreateIssue(btn) {
  var cell = btn.closest('.row-action-cell');
  if (!cell) return;
  cell.innerHTML =
    '<span class="ci-confirm-wrap">' +
      '<button class="row-action ci-confirm" title="Confirm — create Issue" aria-label="Confirm create Issue"><i class="ti ti-check" aria-hidden="true"></i></button>' +
      '<button class="row-action ci-cancel" title="Cancel" aria-label="Cancel create Issue"><i class="ti ti-x" aria-hidden="true"></i></button>' +
    '</span>';
}
/* ✗ — restore the original bug button. */
function disarmCreateIssue(cell) {
  if (cell) cell.innerHTML = createIssueButtonHtml();
}
/* ✓ — run the custom action against the Risk on this cell. */
function runCreateIssue(cell) {
  if (!cell) return;
  var id = cell.getAttribute('data-id');
  if (!id) return;
  var c;
  try { c = getContext(); }
  catch (e) {
    cell.innerHTML = '<button class="row-action create-issue ci-error" title="Cannot run action: ' +
      esc(e.message) + ' — click to retry" aria-label="Create Issue failed, click to retry">' +
      '<i class="ti ti-alert-triangle" aria-hidden="true"></i></button>';
    return;
  }
  cell.innerHTML = '<span class="row-action" title="Creating Issue…"><i class="ti ti-loader-2 spin" aria-hidden="true"></i></span>';
  executeCustomAction(c.base, c.sid, id, CREATE_ISSUE_ACTION)
    .then(function () { return completeRiskActions(c, id); })
    .then(function () { location.reload(); })
    .catch(function (err) {
      console.error('Create Issue failed:', err);
      cell.innerHTML = '<button class="row-action create-issue ci-error" title="Create Issue failed: ' +
        esc(err.message) + ' — click to retry" aria-label="Create Issue failed, click to retry">' +
        '<i class="ti ti-alert-triangle" aria-hidden="true"></i></button>';
    });
}

/* After a Risk is converted to an Issue, mark all of its associated actions
   Completed. Best-effort: a failure to complete one action doesn't abort the
   flow. Uses the environment's real ActionItemState "Completed" path. */
function completeRiskActions(c, riskId) {
  var actions = (DATA.actionMap && DATA.actionMap[riskId]) || [];
  if (!actions.length) return Promise.resolve();
  var doneRaw = pickRawByLabel('ActionItemState', ['Completed', 'Complete']);
  if (!doneRaw) return Promise.resolve();   /* can't resolve a Completed value in this env */
  var doneLbl = cleanLabel(doneRaw).toLowerCase();
  return Promise.all(actions.map(function (a) {
    if (cleanLabel(a.stateRaw).toLowerCase() === doneLbl) return true;   /* already complete */
    return updateObject(c.base, c.sid, a.rawId, { ActionItemState: doneRaw })
      .then(function () { return true; }, function () { return false; });
  }));
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
      { type: 'pick',   field: RISK_IMPACT_FIELD },
      { type: 'static', html: '—' },                       /* Score (auto-calculated) */
      { type: 'static', html: 'Opened' },                  /* new risks are always Opened */
      { type: 'user',   field: 'Owner' },
      (RISK_REPORTING_FIELD ? { type: 'pick', field: RISK_REPORTING_FIELD } : { type: 'static', html: '—' }),
      { type: 'date',   field: 'C_ImpactDate', actions: true }
    ]
  },
  Issue: {
    tbody: 'issues-body', label: 'issue',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'text',   field: 'Title', placeholder: 'Issue name' },
      { type: 'pick',   field: 'C_IssueImpact' },
      { type: 'static', html: '—' },                       /* Score (auto-calculated) */
      { type: 'pick',   field: 'State' },
      { type: 'user',   field: 'Owner' },
      { type: 'pick',   field: 'C_ReportingLevel' },
      { type: 'date',   field: ISSUE_IMPACTDATE_FIELD, actions: true }
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
    /* Only render the date input if this entity actually has the field; the
       Save/Cancel buttons still render so a fieldless date cell stays usable.
       New rows default the date to today. */
    var input = cell.field
      ? '<input type="date" class="new-input edit-input" data-field="' + cell.field + '" value="' + todayISO() + '" />'
      : '';
    return '<td>' + input + actions + '</td>';
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
  /* New Risks are always created with State = Opened. */
  if (entityType === 'Risk' && !fields.State) {
    var opened = pickRawByLabel('State', ['Opened']);
    if (opened) fields.State = opened;
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

/* Resolve a picklist field's real "/Type/Value" path for one of the given
   labels (case-insensitive), sourced from THIS environment — metadata first,
   then values present in the data. Returns '' if none match (never fabricates a
   path, since those differ per environment). */
function pickRawByLabel(field, labels) {
  var wanted = labels.map(function (l) { return String(l).toLowerCase(); });
  function findIn(list) {
    for (var i = 0; i < (list || []).length; i++) {
      var raw = (list[i] && list[i].raw !== undefined) ? list[i].raw : list[i];
      if (raw && wanted.indexOf(cleanLabel(raw).toLowerCase()) > -1) return raw;
    }
    return '';
  }
  return findIn(PICK_META[field]) || findIn(PICK[field]) || '';
}

/* Resolve the real "/Type/Value" path for RequestType = Change Request, so new
   change requests are created with the right type (and show in this tab). */
function changeRequestTypeRaw() {
  return pickRawByLabel('RequestType', ['Change Request']) || '/RequestType/Change Request';
}

/* ---------- 15. Inline editing (auto-commit, AdaptiveWork-style) ----------
   Click a cell to edit; the change is committed automatically — picklists and
   dates on change, free text on blur/Enter, the owner type-ahead on pick.
   Esc cancels. There are no Save/Cancel buttons. */
function startEdit(cell) {
  if (cell.classList.contains('editing') || cell.classList.contains('saving')) return;
  var field = cell.getAttribute('data-field');
  var value = cell.getAttribute('data-value') || '';
  cell.classList.add('editing');

  /* Revert (cancel) on blur if nothing was committed. */
  function blurRevert() { setTimeout(function () { if (cell.classList.contains('editing')) revertEdit(cell); }, 200); }
  function escHandler(e) { if (e.key === 'Escape') { e.preventDefault(); revertEdit(cell); } }

  if (isUserField(field)) {
    var curName = value ? (USER_NAME[value] || cleanLabel(value)) : '';
    cell.innerHTML = '<input type="text" class="edit-input ta-input" autocomplete="off"' +
      ' data-user-id="' + esc(value) + '" value="' + esc(curName) + '" placeholder="Type a name…" />' +
      '<div class="ta-menu" style="display:none"></div>';
    var uinp = cell.querySelector('.ta-input');
    attachUserTypeahead(uinp, cell.querySelector('.ta-menu'), function () { commitEdit(cell); });
    uinp.addEventListener('keydown', escHandler);
    uinp.addEventListener('blur', blurRevert);   /* commit only happens on pick */
    uinp.focus();
    return;
  }

  if (isPick(field)) {
    cell.innerHTML = '<select class="edit-input">' + pickOptionsHtml(field, value, false) + '</select>';
    var sel = cell.querySelector('select');
    sel.addEventListener('change', function () { commitEdit(cell); });
    sel.addEventListener('keydown', escHandler);
    sel.addEventListener('blur', blurRevert);
    sel.focus();
    return;
  }

  if (isDateField(field)) {
    cell.innerHTML = '<input type="date" class="edit-input" value="' + (value ? value.split('T')[0] : '') + '" />';
    var dinp = cell.querySelector('.edit-input');
    dinp.addEventListener('change', function () { commitEdit(cell); });
    dinp.addEventListener('keydown', function (e) { escHandler(e); if (e.key === 'Enter') { e.preventDefault(); commitEdit(cell); } });
    dinp.addEventListener('blur', function () { setTimeout(function () { if (cell.classList.contains('editing')) commitEdit(cell); }, 200); });
    dinp.focus();
    return;
  }

  /* free text */
  cell.innerHTML = '<input type="text" class="edit-input" value="' + esc(value) + '" />';
  var tinp = cell.querySelector('.edit-input');
  tinp.addEventListener('keydown', function (e) { escHandler(e); if (e.key === 'Enter') { e.preventDefault(); commitEdit(cell); } });
  tinp.addEventListener('blur', function () { commitEdit(cell); });
  tinp.focus();
  tinp.select();
}

/* Restore the cell's display from its last committed value. */
function revertEdit(cell) {
  if (!cell) return;
  cell.classList.remove('editing');
  cell.innerHTML = cellDisplay(cell.getAttribute('data-field'), cell.getAttribute('data-value') || '');
}

/* Commit the cell's current input value (no-op if unchanged / nothing picked). */
function commitEdit(cell) {
  if (!cell || !cell.classList.contains('editing')) return;
  var field = cell.getAttribute('data-field');
  var id = cell.getAttribute('data-id');
  var oldVal = cell.getAttribute('data-value') || '';
  var input = cell.querySelector('.edit-input');
  if (!input) return;

  var newValue;
  if (isUserField(field)) {
    newValue = input.getAttribute('data-user-id') || '';
    if (!newValue) { revertEdit(cell); return; }          /* no user picked */
  } else {
    newValue = input.value;
    if (!newValue && field === 'Title') { revertEdit(cell); return; }  /* don't blank the title */
  }
  if (String(newValue) === String(oldVal)) { revertEdit(cell); return; }  /* unchanged */

  cell.classList.remove('editing');
  cell.classList.add('saving');
  cell.innerHTML = '<span class="cell-saving">Saving…</span>';

  updateField(id, field, newValue)
    .then(function () {
      /* Impact/Probability drive the computed Score — reload to refresh it. */
      if (field === RISK_IMPACT_FIELD || field === 'C_Likelihood' || field === 'C_IssueImpact') { location.reload(); return; }
      cell.setAttribute('data-value', newValue);
      cell.classList.remove('saving');
      cell.innerHTML = cellDisplay(field, newValue);
    })
    .catch(function (err) {
      alert('Save failed: ' + err.message);
      cell.classList.remove('saving');
      cell.innerHTML = cellDisplay(field, oldVal);
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
  risks:    { tbody: 'risks-body',    placeholder: 'Search risks…',    newLabel: 'New risk',    create: createNewRisk,    statuses: RISK_STATES },
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

      /* Row-level Create Issue action (Risks tab) — inline ✓/✗ confirm */
      var ciOk = e.target.closest('.ci-confirm');
      if (ciOk) { runCreateIssue(ciOk.closest('.row-action-cell')); return; }
      var ciNo = e.target.closest('.ci-cancel');
      if (ciNo) { disarmCreateIssue(ciNo.closest('.row-action-cell')); return; }
      var ci = e.target.closest('.create-issue');
      if (ci) { armCreateIssue(ci); return; }

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
    rows.push([i.sysId, i.name, i.impact, i.score, i.status, i.owner, i.reportingLevel, exportDateCell(i.impactDate)]);
    kinds.push('record');
    (DATA.actionMap[i.rawId] || []).forEach(function (a) { rows.push(exportActionRow(a, 8)); kinds.push('action'); });
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
      ['ID', 'Issue name', 'Impact', 'Score', 'Status', 'Owner', 'Reporting Level', 'Impact Date'],
      exportRowsIssues(), 3) +
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
      { hdr: 'Impact',          api: 'C_ImpactR',        kind: 'pick' },
      { hdr: 'Status',          api: 'State',            kind: 'pick' },
      { hdr: 'Owner',           api: 'Owner',            kind: 'user' },
      { hdr: 'Reporting Level', api: 'C_ReportingLevel', kind: 'pick' },
      { hdr: 'Impact Date',     api: 'C_ImpactDate',     kind: 'date' }
    ]
  },
  'Issues': {
    entityType: 'Issue', list: 'issues',
    fields: [
      { hdr: 'Issue name',      api: 'Title',            kind: 'text' },
      { hdr: 'Impact',          api: 'C_IssueImpact',    kind: 'pick' },
      { hdr: 'Status',          api: 'State',            kind: 'pick' },
      { hdr: 'Owner',           api: 'Owner',            kind: 'user' },
      { hdr: 'Reporting Level', api: 'C_ReportingLevel', kind: 'pick' },
      { hdr: 'Impact Date',     api: 'C_ImpactDate',     kind: 'date' }
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

/* Risk's Reporting Level field API name differs from Issue's — point the Risk
   import column at RISK_REPORTING_FIELD, or drop it if Risk has no such field. */
(function () {
  var rf = IMPORT_CFG.Risks.fields;
  for (var i = rf.length - 1; i >= 0; i--) {
    if (rf[i].hdr === 'Reporting Level') {
      if (RISK_REPORTING_FIELD) rf[i].api = RISK_REPORTING_FIELD; else rf.splice(i, 1);
    }
  }
})();

/* Issue's Impact Date field differs / is absent in this tenant — point the
   Issue import column at ISSUE_IMPACTDATE_FIELD, or drop it if there is none. */
(function () {
  var iff = IMPORT_CFG.Issues.fields;
  for (var i = iff.length - 1; i >= 0; i--) {
    if (iff[i].hdr === 'Impact Date') {
      if (ISSUE_IMPACTDATE_FIELD) iff[i].api = ISSUE_IMPACTDATE_FIELD; else iff.splice(i, 1);
    }
  }
})();

/* Current raw value of an API field on a loaded record (for change detection). */
function recCurrent(entityType, rec, api) {
  if (api === 'Title') return rec.name === '(unnamed)' ? '' : rec.name;
  if (entityType === 'Risk') {
    if (api === 'C_Likelihood')     return rec.probabilityRaw;
    if (api === RISK_IMPACT_FIELD)  return rec.impactRaw;
    if (api === 'State')            return rec.statusRaw;
    if (api === 'Owner')            return rec.ownerRaw;
    if (RISK_REPORTING_FIELD && api === RISK_REPORTING_FIELD) return rec.reportingLevelRaw;
    if (api === 'C_ImpactDate')     return rec.impactDate || '';
  } else if (entityType === 'Issue') {
    if (api === 'C_IssueImpact')    return rec.impactRaw;
    if (api === 'State')            return rec.statusRaw;
    if (api === 'Owner')            return rec.ownerRaw;
    if (api === 'C_ReportingLevel') return rec.reportingLevelRaw;
    if (ISSUE_IMPACTDATE_FIELD && api === ISSUE_IMPACTDATE_FIELD) return rec.impactDate || '';
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
    showError('risks-body', 11, e.message);
    showError('issues-body', 9, e.message);
    showError('requests-body', 7, e.message);
    return;
  }

  /* DIAGNOSTIC: confirm the resolved API host and the project id injected by the
     Data field. After moving environments, a wrong host or a stale/empty project
     id from the Data macro is a prime suspect — check these in the console. */
  console.info('[RAID panel] app host:', window.location.hostname,
    '| API base:', c.base, '| project (PlannedFor):', c.projId,
    '| session set:', !!c.sid);
  diag('host ' + window.location.hostname + '  |  api ' + c.base);
  diag('project ' + c.projId + '  |  session ' + (!!c.sid));

  showLoading('risks-body', 11);
  showLoading('issues-body', 9);
  showLoading('requests-body', 7);

  /* Load real picklist option paths (non-blocking). Query the custom picklist
     type entities for their values, and also try describeEntities metadata. */
  loadPicklistValues().catch(function (err) {
    console.warn('Picklist value query failed:', err && err.message);
  });
  loadPicklistMeta().catch(function (err) {
    console.warn('Picklist metadata unavailable, falling back to data-derived options:', err && err.message);
  });

  var qRisks =
    "SELECT SYSID, Title, C_Likelihood, " + RISK_IMPACT_FIELD + ", C_RiskRating, State, Owner.Name, " +
    (RISK_REPORTING_FIELD ? RISK_REPORTING_FIELD + ", " : "") +
    "C_ImpactDate FROM Risk WHERE PlannedFor = '" + c.projId + "'";
  var qIssues =
    "SELECT SYSID, Title, C_IssueImpact, C_IssueScore, State, Owner.Name, C_ReportingLevel" +
    (ISSUE_IMPACTDATE_FIELD ? ", " + ISSUE_IMPACTDATE_FIELD : "") +
    " FROM Issue WHERE PlannedFor = '" + c.projId + "'";
  var qRequests =
    "SELECT SYSID, Title, RequestType, State, CreatedBy.Name, CreatedOn, DueDate " +
    "FROM EnhancementRequest WHERE PlannedFor = '" + c.projId + "'";

  Promise.all([
    czql(c.base, c.sid, qRisks),
    czql(c.base, c.sid, qIssues),
    czql(c.base, c.sid, qRequests)
  ])
  .then(function (results) {
    /* Risks tab shows only the Opened / Closed states (hide Realised, etc.). */
    var riskRows = results[0].filter(function (e) {
      return isShownRiskState(cleanLabel(pickRaw(e.State)));
    });
    var risks    = riskRows.map(toRisk);
    var issues   = results[1].map(toIssue);
    /* Requests tab shows Change Requests only (RequestType = 'Change Request'). */
    var reqRows  = results[2].filter(function (e) {
      return cleanLabel(pickRaw(e.RequestType) || pickRaw(e.Type)).toLowerCase() === 'change request';
    });
    var requests = reqRows.map(toRequest);

    var caseIds = []
      .concat(riskRows.map(function (e) { return e.id; }))
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
    showError('risks-body', 11, msg);
    showError('issues-body', 9, msg);
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
