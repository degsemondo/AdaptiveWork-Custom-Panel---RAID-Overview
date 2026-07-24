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
var RISK_STATES = ['Draft', 'Open'];
function isShownRiskState(label) {
  return RISK_STATES.some(function (s) { return s.toLowerCase() === String(label || '').toLowerCase(); });
}

/* Selectable Status values when EDITING a Risk — the inline Status dropdown on a
   risk row offers only these, and new risks start as Draft. Distinct from
   RISK_STATES, which controls which risks are DISPLAYED. "Closed" isn't present
   in the loaded data, so its path is derived from the environment's own State
   prefix (e.g. /CaseState/Closed). */
var RISK_STATE_OPTIONS = ['Draft', 'Open', 'Closed'];

/* Selectable states when editing an ActionItem; new actions default to Open.
   This tenant's ActionItemState enum is Open / Closed / Stuck (confirmed with the
   group). These are offered (current value kept if it's another state); the done
   state is Closed. */
var ACTION_STATE_OPTIONS = ['Open', 'Closed', 'Stuck'];

/* Name of the Risk custom action that flips the Risk to Realised and spawns a
   linked Issue. If the action's API name differs from its display name, change
   this one string. */
var CREATE_ISSUE_ACTION = 'Create Issue from Risk';

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
   C_IssueIssueImpact; Risk Probability is C_Probability -> C_RiskProbability.) */
var RISK_IMPACT_FIELD = 'C_ImpactR';

/* Issue Impact Date has been removed from the panel — kept as '' so the (now
   unused) impact-date conditionals stay inert. */
var ISSUE_IMPACTDATE_FIELD = '';

/* The Risks tab shows Due Date (standard DueDate) + Next Review Date
   (C_NextReviewDateR); it no longer shows the old C_ImpactDate. */
var RISK_NEXTREVIEW_FIELD = 'C_NextReviewDateR';

/* Latest loaded data, retained so "Export to Excel" can use it. */
var DATA = { risks: [], issues: [], requests: [], actionMap: {} };

/* Picklist option lists, collected from the loaded data (field -> [rawPath]). */
var PICK = {};
/* Authoritative picklist options fetched from AdaptiveWork metadata
   (field -> [{ raw, label }]). Preferred over guessing a "/Type" prefix. */
var PICK_META = {};
var PICK_FIELDS = ['C_Impact', 'C_ImpactR', 'C_Probability', 'C_IssueImpact', 'State', 'Priority', 'RequestType', 'C_ReportingLevel', 'C_ReportingLevelR', 'ActionItemState'];
function isPick(f) { return PICK_FIELDS.indexOf(f) > -1; }

/* Status-like fields rendered as a coloured pill; date fields as a formatted date. */
function isPillField(f) { return f === 'State' || f === 'Priority'; }
function isDateField(f) { return f === 'DueDate' || f === RISK_NEXTREVIEW_FIELD; }
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

/* The signed-in user — used to default the Owner on new rows. Resolved at load
   via getSessionInfo (a custom PAGE has no CurrentObject to read the user from).
   Tries the data + authentication service paths; best-effort, non-blocking. */
var CURRENT_USER = { id: '', name: '' };
function fetchUserName(base, sid, userId) {
  var url = base.replace(/\/$/, '') + API_OBJECTS + '/' + String(userId).replace(/^\//, '') + '?fields=Name';
  return fetch(url, { headers: { 'Authorization': 'Session ' + sid } })
    .then(function (res) { if (!res.ok) throw new Error('HTTP ' + res.status); return res.json(); })
    .then(function (j) { return (j && (j.Name || (j.entity && j.entity.Name))) || cleanLabel(userId); })
    .catch(function () { return cleanLabel(userId); });
}
function loadCurrentUser() {
  var c = getContext();
  var paths = ['/V2.0/services/data/getSessionInfo', '/V2.0/services/authentication/getSessionInfo'];
  function tryPath(i) {
    if (i >= paths.length) { return Promise.resolve(); }
    var url = c.base.replace(/\/$/, '') + paths[i];
    return fetch(url, { headers: { 'Authorization': 'Session ' + c.sid } })
      .then(function (res) { if (!res.ok) throw new Error('HTTP ' + res.status); return res.json(); })
      .then(function (j) {
        var uid = j.userId || j.UserId || (j.user && (j.user.id || j.user.Id)) ||
                  (j.sessionInfo && (j.sessionInfo.userId || j.sessionInfo.UserId)) || '';
        if (!uid) throw new Error('no userId in response');
        /* If the response carries the user's display name, use it directly and
           skip the per-user object lookup. */
        var directName = j.userName || j.UserName || j.name || j.Name ||
                         (j.user && (j.user.Name || j.user.name)) || '';
        /* getSessionInfo returns a bare GUID; the object path + Owner reference
           both need the full "/User/<id>" form. */
        if (String(uid).charAt(0) !== '/') uid = '/User/' + uid;
        CURRENT_USER.id = uid;
        if (directName) {
          CURRENT_USER.name = directName;
          rememberUser(uid, directName);
          return;
        }
        return fetchUserName(c.base, c.sid, uid).then(function (name) {
          CURRENT_USER.name = name;
          rememberUser(uid, name);
        });
      })
      .catch(function () { return tryPath(i + 1); });
  }
  return tryPath(0);
}

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
  C_ImpactR:        ['1 - Minor', '2 - Moderate', '3 - Moderate+', '4 - Significant', '5 - Significant+', '6 - Severe'],
  C_IssueImpact:    ['1 - Minor', '2 - Moderate', '3 - Moderate +', '4 - Significant', '5 - Significant +', '6 - Severe'],
  C_Probability:     ['1 - Unlikely', '2 - Possible', '3 - Possible +', '4 - Likely', '5 - Likely +', '6 - Very Likely'],
  C_ReportingLevel:  ['1 - EFDC', '2 - Portfolio', '3 - Project Board', '4 - Project Team'],
  C_ReportingLevelR: ['1 - EFDC', '2 - Portfolio', '3 - Project Board', '4 - Project Team']
};

/* Custom picklists are their own entities whose INSTANCES are the values, so we
   read the current environment's real options with a plain query against the
   picklist type (its "Class API Name"). Each row's id is the exact "/Type/Value"
   path to save; its Name is the label. Map: field -> CANDIDATE type entity names
   (tried in order; first one that exists + returns values wins).
   All confirmed from each field's "Class API Name" on eu.clarizentb.com:
     C_Impact -> C_RiskImpact, C_Probability -> C_RiskProbability,
     C_IssueImpact -> C_IssueIssueImpact,
     C_ReportingLevel (Issue) -> C_IssueReportingLevel,
     C_ReportingLevelR (Risk) -> C_RiskReportingLevelR.
   (State / Priority / RequestType / ActionItemState are system enums, sourced
   from the loaded data instead, so they aren't listed here.) */
var PICK_LIST_TYPES = {
  C_ImpactR:         ['C_RiskImpactR'],
  C_Probability:      ['C_RiskProbability', 'C_RiskLikelihood'],
  C_IssueImpact:     ['C_IssueIssueImpact'],
  C_ReportingLevel:  ['C_IssueReportingLevel'],
  C_ReportingLevelR: ['C_RiskReportingLevelR'],
  ActionItemState:   ['ActionItemState']   /* discover the real action states */
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

/* Restrict a picklist to a fixed set of labels (e.g. a Risk's Status to
   Open/Closed). Prefer a value already present in this environment
   (metadata/data); otherwise derive the path from the prefix the environment
   itself uses for that field — so "Closed" becomes /CaseState/Closed when the
   tenant's states live under /CaseState/. `fallback` is used only if no existing
   value reveals the prefix. */
function fieldPrefix(field, fallback) {
  var lists = [PICK_META[field], PICK[field]];
  for (var li = 0; li < lists.length; li++) {
    var list = lists[li] || [];
    for (var i = 0; i < list.length; i++) {
      var raw = (list[i] && list[i].raw !== undefined) ? list[i].raw : list[i];
      if (raw && String(raw).charAt(0) === '/') { var p = String(raw).split('/'); p.pop(); return p.join('/') + '/'; }
    }
  }
  return fallback;
}
function rawForLabel(field, label, fallback) {
  return pickRawByLabel(field, [label]) || (fieldPrefix(field, fallback) + label);
}
/* Options limited to `labels`, plus the row's current value if it isn't one of
   them (so an existing legacy value still shows and can be changed). */
function restrictedOptionList(field, labels, fallback, current) {
  var out = [], seen = {};
  labels.forEach(function (lbl) {
    var raw = rawForLabel(field, lbl, fallback);
    if (raw && !seen[raw]) { seen[raw] = 1; out.push({ raw: raw, label: lbl }); }
  });
  if (current && !seen[current]) out.push({ raw: current, label: cleanLabel(current) });
  return out;
}
function restrictedOptionsHtml(field, labels, fallback, current, includeBlank) {
  var html = includeBlank ? '<option value="">—</option>' : '';
  return html + restrictedOptionList(field, labels, fallback, current).map(function (o) {
    return '<option value="' + esc(o.raw) + '"' + (o.raw === current ? ' selected' : '') + '>' + esc(o.label) + '</option>';
  }).join('');
}
/* Risk Status dropdown (Open / Closed). */
function riskStateOptionsHtml(current, includeBlank) {
  return restrictedOptionsHtml('State', RISK_STATE_OPTIONS, '/CaseState/', current, includeBlank);
}
/* ActionItem status dropdown. Offer Open / Closed ONLY IF the tenant's
   ActionItemState enum actually has them (real values only — never fabricate a
   path, which 500s with "Entity 'X' of type 'ActionItemState' was not found").
   If it has neither, fall back to the enum's real values so the dropdown is
   always saveable. */
function actionStateOptionsHtml(current, includeBlank) {
  var out = [], seen = {};
  ACTION_STATE_OPTIONS.forEach(function (lbl) {
    var raw = pickRawByLabel('ActionItemState', [lbl]);
    if (raw && !seen[raw]) { seen[raw] = 1; out.push({ raw: raw, label: lbl }); }
  });
  if (!out.length) return pickOptionsHtml('ActionItemState', current, includeBlank);
  if (current && !seen[current]) out.push({ raw: current, label: cleanLabel(current) });
  var html = includeBlank ? '<option value="">—</option>' : '';
  return html + out.map(function (o) {
    return '<option value="' + esc(o.raw) + '"' + (o.raw === current ? ' selected' : '') + '>' + esc(o.label) + '</option>';
  }).join('');
}
/* Real ActionItemState path for a label, or '' if the enum doesn't have it. */
function actionStateRaw(label) {
  return pickRawByLabel('ActionItemState', [label]);
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
  return Promise.all(jobs);
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
  var probRaw = pickRaw(e.C_Probability);
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
    dueDate:           e.DueDate || null,
    nextReviewDate:    e.C_NextReviewDateR || null,
    description:       e.Description || '',
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
    dueDate:           e.DueDate || null,
    description:       e.Description || '',
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
    description: e.Description || '',
    rawId:      e.id || ''
  };
}

function toAction(e) {
  var today  = new Date();
  var due    = e.DueDate ? new Date(e.DueDate) : null;
  var stateV = cleanLabel(extractPicklistValue(e.ActionItemState));
  var isDone = /^(complete|completed|closed|done)$/i.test(stateV);
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
    statusAction: e.C_SummaryUpdateAction || '',
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
  Pending: 'pill-pending', New: 'pill-new', Draft: 'pill-new', Submitted: 'pill-new',
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
  var isDone = /^(complete|completed|closed|done)$/i.test(stateV);
  var isLate = !isDone && due && due < new Date();
  /* When done, the Status pill already says so — no due/overdue text. */
  var label = isDone ? ''
            : isLate  ? 'Overdue'
            : due     ? 'Due ' + fmtDate(dueIso)
            :           'No due date';
  return { label: label, done: isDone, late: isLate };
}
/* Inner HTML of an action row in display mode (icon, name, owner, status, tools).
   statusAction = free-text C_SummaryUpdateAction value (drives the status pop-out icon's tint). */
function actionInnerHtml(name, ownerName, stateRaw, dueIso, statusAction) {
  var s = actionStatusInfo(stateRaw, dueIso);
  var iconCol = s.done ? '#3B6D11' : '#185FA5';
  var dCls = s.late ? 'overdue' : 'ok';
  var stateLbl = cleanLabel(stateRaw);
  var statePill = stateLbl ? '<span class="pill ' + (PILL_MAP[stateLbl] || 'pill-medium') + '">' + esc(stateLbl) + '</span>' : '';
  var hasStatus = !!(statusAction && String(statusAction).trim());
  var statusBtn = '<button class="act-status" title="' + (hasStatus ? 'View / edit status update' : 'Add status update') +
    '" aria-label="Action status"' + ' style="color:' + (hasStatus ? '#8B3A62' : '#c9ccd1') + '">' +
    '<i class="ti ti-notes" aria-hidden="true"></i></button>';
  return '<i class="ti ti-circle-check" style="font-size:14px;color:' + iconCol + '" aria-hidden="true"></i>' +
    '<span class="action-name">' + esc(name) + '</span>' +
    '<span class="action-meta">' + esc(ownerName && ownerName !== '—' ? ownerName : '—') + '</span>' +
    statePill +
    (s.label ? '<span class="action-due ' + dCls + '">· ' + esc(s.label) + '</span>' : '') +
    '<span class="action-tools">' +
      statusBtn +
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
    ' data-state="' + esc(a.stateRaw) + '"' +
    ' data-status-action="' + esc(a.statusAction || '') + '">' +
    actionInnerHtml(a.name, a.assignee, a.stateRaw, a.dueDate, a.statusAction) +
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
    '<select class="edit-input act-f-state">' + actionStateOptionsHtml(a.stateRaw || '', false) + '</select>' +
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
    stateRaw: el.getAttribute('data-state') || '',
    statusAction: el.getAttribute('data-status-action') || ''
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
  el.innerHTML = actionInnerHtml(a.name, a.assignee, a.stateRaw, a.dueDate, a.statusAction);
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
  /* New actions default to: Owner = signed-in user, Due = today, State = Open. */
  el.innerHTML = actionFormHtml({
    ownerRaw: CURRENT_USER.id,
    assignee: CURRENT_USER.name,
    dueDate:  todayISO(),
    stateRaw: actionStateRaw('Open')
  });
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
  p.then(function () { return loadAndRender(); })
   .catch(function (err) {
     alert('Failed to save action: ' + err.message);
     if (btn) { btn.disabled = false; btn.textContent = 'Save'; }
   });
}
function deleteAction(el) {
  if (!el) return;
  var actionId = el.getAttribute('data-action-id');
  if (!actionId) { el.remove(); return; }
  showConfirm({
    anchor: el.querySelector('.act-del'),
    title: 'Delete action',
    message: 'Delete this action? This cannot be undone.',
    confirmLabel: 'Delete',
    danger: true,
    onConfirm: function () {
      var c;
      try { c = getContext(); } catch (e) { alert('Cannot delete action: ' + e.message); return; }
      deleteObject(c.base, c.sid, actionId)
        .then(function () { return loadAndRender(); })
        .catch(function (err) { alert('Failed to delete action: ' + err.message); });
    }
  });
}
/* Pop-out editor for an action's free-text Status update (C_SummaryUpdateAction) — the
   same centered, non-dimming textarea dialog used for a Risk's Description.
   Reads/writes the value from the row's data-status-action attribute + the
   ActionItem's C_SummaryUpdateAction field, then soft-refreshes. */
function openActionStatus(el) {
  if (!el) return;
  var actionId = el.getAttribute('data-action-id');
  if (!actionId) { alert('Save the action first, then add a status update.'); return; }
  var current = el.getAttribute('data-status-action') || '';
  var name = el.getAttribute('data-name') || '';
  showDescriptionDialog({
    title: 'Status update' + (name ? ' — ' + name : ''),
    value: current,
    onSave: function (text) {
      if (String(text) === String(current)) return;   /* unchanged */
      var c;
      try { c = getContext(); } catch (e) { alert('Cannot save status: ' + e.message); return; }
      updateObject(c.base, c.sid, actionId, { C_SummaryUpdateAction: text })
        .then(function () { el.setAttribute('data-status-action', text); return loadAndRender(); })
        .catch(function (err) { alert('Failed to save status: ' + err.message); });
    }
  });
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

/* Small count badge shown on a record row when it has associated actions, so
   users can see there are items to expand. Inline-styled (no CSS-field change). */
function actionBadge(n) {
  if (!n) return '';
  return ' <span class="action-badge" title="' + n + ' associated action' + (n === 1 ? '' : 's') + '"' +
    ' style="display:inline-block;min-width:14px;height:14px;line-height:14px;padding:0 4px;' +
    'border-radius:7px;background:#8B3A62;color:#fff;font-size:9px;font-weight:600;' +
    'text-align:center;vertical-align:middle;">' + n + '</span>';
}

/* Description icon for a record row — burgundy when the record has a Description,
   faint when empty. Clicking opens a plain-textarea dialog to view/edit it
   (Description is a Text Area field, so no rich-text handling needed). */
function descIcon(rec) {
  var has = !!(rec.description && String(rec.description).trim());
  return '<button class="desc-btn" data-id="' + esc(rec.rawId) + '"' +
    ' title="' + (has ? 'View / edit description' : 'Add description') + '" aria-label="Description"' +
    ' style="border:none;background:none;cursor:pointer;padding:0 0 0 3px;vertical-align:middle;line-height:1;color:' +
    (has ? '#8B3A62' : '#c9ccd1') + ';">' +
    '<i class="ti ti-file-text" style="font-size:14px" aria-hidden="true"></i></button>';
}

/* ---------- 11. Renderers ---------- */
function renderRisks(risks, actionMap) {
  var tbody = document.getElementById('risks-body');
  if (!risks.length) {
    tbody.innerHTML = '<tr><td colspan="13"><div class="no-data">No risks found for this project.</div></td></tr>';
    document.getElementById('badge-risks').textContent = '0';
    return;
  }
  tbody.innerHTML = risks.map(function (r) {
    var actions = actionMap[r.rawId] || [];
    var hm = riskRatingHeatMap(r.riskRating);
    return '<tr class="data-row" data-status="' + esc(r.status) + '">' +
      '<td><button class="expand-btn" data-target="' + r.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td style="white-space:nowrap">' + idLink(r.rawId, r.sysId) + actionBadge(actions.length) + '</td>' +
      editCell('State',            r.rawId, r.statusRaw,         pill(r.status)) +
      editCell('Title',            r.rawId, r.name,              esc(r.name)) +
      '<td style="text-align:center">' + descIcon(r) + '</td>' +
      editCell('C_Probability',     r.rawId, r.probabilityRaw,    esc(r.probability)) +
      editCell(RISK_IMPACT_FIELD,  r.rawId, r.impactRaw,         esc(r.impact)) +
      '<td><span class="heatmap ' + hm + '">' + esc(r.riskRating) + '</span></td>' +
      editCell('Owner',            r.rawId, r.ownerRaw,          esc(r.owner)) +
      editCell('DueDate',           r.rawId, r.dueDate || '',        '<span class="action-due ' + dueCls(r.dueDate) + '">' + fmtDate(r.dueDate) + '</span>') +
      (RISK_REPORTING_FIELD
        ? editCell(RISK_REPORTING_FIELD, r.rawId, r.reportingLevelRaw, esc(r.reportingLevel))
        : '<td>' + esc(r.reportingLevel || '—') + '</td>') +
      editCell('C_NextReviewDateR', r.rawId, r.nextReviewDate || '', '<span class="action-due ' + dueCls(r.nextReviewDate) + '">' + fmtDate(r.nextReviewDate) + '</span>') +
      '<td class="row-action-cell" data-id="' + esc(r.rawId) + '" data-name="' + esc(r.name) + '">' +
        createIssueButtonHtml() +
      '</td>' +
      '</tr>' +
      '<tr class="actions-row" id="' + r.id + '-actions"><td colspan="13">' + actionsBlock(actions, r.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-risks').textContent = risks.length;
}

function renderIssues(issues, actionMap) {
  var tbody = document.getElementById('issues-body');
  if (!issues.length) {
    tbody.innerHTML = '<tr><td colspan="10"><div class="no-data">No issues found for this project.</div></td></tr>';
    document.getElementById('badge-issues').textContent = '0';
    return;
  }
  tbody.innerHTML = issues.map(function (issue) {
    var actions = actionMap[issue.rawId] || [];
    var hm = riskRatingHeatMap(issue.score);
    return '<tr class="data-row" data-status="' + esc(issue.status) + '">' +
      '<td><button class="expand-btn" data-target="' + issue.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td style="white-space:nowrap">' + idLink(issue.rawId, issue.sysId) + actionBadge(actions.length) + '</td>' +
      '<td style="text-align:center">' + descIcon(issue) + '</td>' +
      editCell('Title',            issue.rawId, issue.name,              esc(issue.name)) +
      editCell('C_IssueImpact',    issue.rawId, issue.impactRaw,         esc(issue.impact)) +
      '<td><span class="heatmap ' + hm + '">' + esc(issue.score) + '</span></td>' +
      editCell('State',            issue.rawId, issue.statusRaw,         pill(issue.status)) +
      editCell('Owner',            issue.rawId, issue.ownerRaw,          esc(issue.owner)) +
      editCell('C_ReportingLevel', issue.rawId, issue.reportingLevelRaw, esc(issue.reportingLevel)) +
      editCell('DueDate',          issue.rawId, issue.dueDate || '', '<span class="action-due ' + dueCls(issue.dueDate) + '">' + fmtDate(issue.dueDate) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + issue.id + '-actions"><td colspan="10">' + actionsBlock(actions, issue.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-issues').textContent = issues.length;
}

function renderRequests(requests, actionMap) {
  var tbody = document.getElementById('requests-body');
  if (!requests.length) {
    tbody.innerHTML = '<tr><td colspan="8"><div class="no-data">No change requests found for this project.</div></td></tr>';
    document.getElementById('badge-requests').textContent = '0';
    return;
  }
  tbody.innerHTML = requests.map(function (req) {
    var actions = actionMap[req.rawId] || [];
    return '<tr class="data-row" data-status="' + esc(req.status) + '">' +
      '<td><button class="expand-btn" data-target="' + req.id + '-actions" aria-label="Show actions"></button></td>' +
      '<td style="white-space:nowrap">' + idLink(req.rawId, req.sysId) + actionBadge(actions.length) + '</td>' +
      '<td style="text-align:center">' + descIcon(req) + '</td>' +
      editCell('Title',       req.rawId, req.name,    esc(req.name)) +
      editCell('State',       req.rawId, req.statusRaw, pill(req.status)) +
      '<td>' + esc(req.requestor) + '</td>' +
      '<td>' + fmtDate(req.submitted) + '</td>' +
      editCell('DueDate',     req.rawId, req.decisionBy || '', '<span class="action-due ' + dueCls(req.decisionBy) + '">' + fmtDate(req.decisionBy) + '</span>') +
      '</tr>' +
      '<tr class="actions-row" id="' + req.id + '-actions"><td colspan="8">' + actionsBlock(actions, req.rawId) + '</td></tr>';
  }).join('');
  document.getElementById('badge-requests').textContent = requests.length;
}

/* Collect picklist options from loaded data, then render everything. */
function renderAll(risks, issues, requests, actionMap) {
  /* Remember which action panels are currently open (by their stable row id,
     "<record>-actions") so a soft refresh — e.g. after saving an action —
     doesn't snap them shut. Captured BEFORE the tbodies are rebuilt. */
  var openRows = {};
  document.querySelectorAll('.actions-row.show').forEach(function (row) { if (row.id) openRows[row.id] = true; });

  DATA = { risks: risks, issues: issues, requests: requests, actionMap: actionMap };
  PICK = {};
  risks.forEach(function (r) {
    addPick(RISK_IMPACT_FIELD, r.impactRaw);
    addPick('C_Probability', r.probabilityRaw);
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
  /* Keep any active column sort applied across (soft) refreshes. */
  applySort('risks');
  applySort('issues');
  renderRisks(risks, actionMap);
  renderIssues(issues, actionMap);
  renderRequests(requests, actionMap);
  /* Re-apply any "Expand all" state (a refresh rebuilds the rows collapsed). */
  Object.keys(EXPANDED_ALL).forEach(function (tab) { if (EXPANDED_ALL[tab]) setAllExpanded(tab, true); });
  /* Re-open any individually-expanded panels that survived into the new render. */
  Object.keys(openRows).forEach(function (id) {
    var row = document.getElementById(id);
    if (!row) return;
    row.classList.add('show');
    var btn = document.querySelector('.expand-btn[data-target="' + id + '"]');
    if (btn) btn.classList.add('open');
  });
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

/* Lightweight confirmation POPOVER (NOT the browser's window.confirm, and NOT a
   full-panel modal — it doesn't dim the page). A small card is anchored next to
   the element the user clicked; an invisible click-catcher dismisses it on an
   outside click. opts = { anchor, title, message, confirmLabel, danger?, onConfirm }.
   Enter confirms, Esc / outside-click / Cancel dismisses. */
function positionPopover(pop, anchor) {
  var vw = window.innerWidth, vh = window.innerHeight;
  var pr = pop.getBoundingClientRect();
  var w = pr.width || 280, h = pr.height || 110, left, top;
  if (anchor && anchor.getBoundingClientRect) {
    var r = anchor.getBoundingClientRect();
    top = r.bottom + 6;
    left = r.right - w;                              /* right-align under the control */
    if (top + h > vh - 8) top = r.top - h - 6;       /* flip above if no room below */
    if (top < 8) top = 8;
  } else {
    left = (vw - w) / 2; top = (vh - h) / 2;
  }
  if (left < 8) left = 8;
  if (left + w > vw - 8) left = vw - w - 8;
  pop.style.left = Math.round(left) + 'px';
  pop.style.top = Math.round(top) + 'px';
}
function showConfirm(opts) {
  opts = opts || {};
  var catcher = document.createElement('div');   /* transparent — no dimming */
  catcher.style.cssText = 'position:fixed;top:0;left:0;right:0;bottom:0;z-index:2000;background:transparent;';
  var pop = document.createElement('div');
  var okBg = opts.danger ? '#A32D2D' : '#8B3A62';
  pop.style.cssText = 'position:fixed;z-index:2001;background:#fff;border:1px solid #dadce0;' +
    'border-radius:8px;box-shadow:0 6px 22px rgba(0,0,0,0.18);padding:12px 14px;max-width:280px;' +
    'font:13px/1.45 system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;color:#202124;';
  pop.innerHTML =
    (opts.title ? '<div style="font-weight:600;margin-bottom:4px;">' + esc(opts.title) + '</div>' : '') +
    '<div style="margin-bottom:10px;color:#3c4043;">' + esc(opts.message || '') + '</div>' +
    '<div style="display:flex;justify-content:flex-end;gap:6px;">' +
      '<button class="aw-c-cancel" style="padding:5px 10px;border:1px solid #dadce0;background:#fff;color:#3c4043;border-radius:4px;cursor:pointer;font:inherit;">Cancel</button>' +
      '<button class="aw-c-ok" style="padding:5px 10px;border:none;background:' + okBg + ';color:#fff;border-radius:4px;cursor:pointer;font:inherit;">' + esc(opts.confirmLabel || 'Confirm') + '</button>' +
    '</div>';
  function close() {
    if (pop.parentNode) pop.parentNode.removeChild(pop);
    if (catcher.parentNode) catcher.parentNode.removeChild(catcher);
    document.removeEventListener('keydown', onKey, true);
  }
  function go() { close(); if (opts.onConfirm) opts.onConfirm(); }
  function onKey(e) { if (e.key === 'Escape') { e.preventDefault(); close(); } else if (e.key === 'Enter') { e.preventDefault(); go(); } }
  catcher.addEventListener('mousedown', close);
  document.addEventListener('keydown', onKey, true);
  document.body.appendChild(catcher);
  document.body.appendChild(pop);
  pop.querySelector('.aw-c-cancel').addEventListener('click', close);
  pop.querySelector('.aw-c-ok').addEventListener('click', go);
  positionPopover(pop, opts.anchor);
  var okBtn = pop.querySelector('.aw-c-ok'); if (okBtn) okBtn.focus();
}

/* Description editor — a plain textarea in a centered, non-dimming dialog (a
   transparent click-catcher dismisses it, so the panel isn't greyed out).
   Esc / outside-click / Cancel discards; Save commits. opts = { title, value, onSave }. */
function showDescriptionDialog(opts) {
  opts = opts || {};
  var catcher = document.createElement('div');
  catcher.style.cssText = 'position:fixed;top:0;left:0;right:0;bottom:0;z-index:2000;background:transparent;';
  var card = document.createElement('div');
  card.style.cssText = 'position:fixed;z-index:2001;top:50%;left:50%;transform:translate(-50%,-50%);' +
    'background:#fff;border:1px solid #dadce0;border-radius:8px;box-shadow:0 12px 40px rgba(0,0,0,0.30);' +
    'padding:16px 18px;width:92vw;max-width:460px;box-sizing:border-box;' +
    'font:13px/1.5 system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;color:#202124;';
  card.innerHTML =
    '<div style="font-weight:600;font-size:14px;margin-bottom:8px;">' + esc(opts.title || 'Description') + '</div>' +
    '<textarea class="aw-desc-input" rows="8" style="width:100%;min-height:150px;resize:vertical;' +
      'box-sizing:border-box;padding:8px;border:1px solid #dadce0;border-radius:4px;font:inherit;color:inherit;"></textarea>' +
    '<div style="display:flex;justify-content:flex-end;gap:6px;margin-top:12px;">' +
      '<button class="aw-d-cancel" style="padding:6px 12px;border:1px solid #dadce0;background:#fff;color:#3c4043;border-radius:4px;cursor:pointer;font:inherit;">Cancel</button>' +
      '<button class="aw-d-ok" style="padding:6px 12px;border:none;background:#8B3A62;color:#fff;border-radius:4px;cursor:pointer;font:inherit;">Save</button>' +
    '</div>';
  var ta = card.querySelector('.aw-desc-input');
  ta.value = opts.value || '';   /* set via .value (not innerHTML) to keep text/newlines verbatim */
  function close() {
    if (card.parentNode) card.parentNode.removeChild(card);
    if (catcher.parentNode) catcher.parentNode.removeChild(catcher);
    document.removeEventListener('keydown', onKey, true);
  }
  function save() { var v = ta.value; close(); if (opts.onSave) opts.onSave(v); }
  function onKey(e) { if (e.key === 'Escape') { e.preventDefault(); close(); } }   /* Enter = newline in the textarea */
  catcher.addEventListener('mousedown', close);
  document.addEventListener('keydown', onKey, true);
  document.body.appendChild(catcher);
  document.body.appendChild(card);
  card.querySelector('.aw-d-cancel').addEventListener('click', close);
  card.querySelector('.aw-d-ok').addEventListener('click', save);
  ta.focus();
}
/* Locate a loaded record (any tab) by its full rawId. */
function findRecordByRawId(rawId) {
  var lists = [DATA.risks, DATA.issues, DATA.requests];
  for (var i = 0; i < lists.length; i++) {
    var l = lists[i] || [];
    for (var j = 0; j < l.length; j++) { if (l[j].rawId === rawId) return l[j]; }
  }
  return null;
}
/* Open the Description editor for a record; on save, POST Description + refresh. */
function openDescription(rawId) {
  var rec = findRecordByRawId(rawId);
  if (!rec) return;
  showDescriptionDialog({
    title: 'Description — ' + (rec.name || ''),
    value: rec.description || '',
    onSave: function (text) {
      if (String(text) === String(rec.description || '')) return;   /* unchanged */
      var c;
      try { c = getContext(); } catch (e) { alert('Cannot save description: ' + e.message); return; }
      updateObject(c.base, c.sid, rec.rawId, { Description: text })
        .then(function () { rec.description = text; return loadAndRender(); })
        .catch(function (err) { alert('Failed to save description: ' + err.message); });
    }
  });
}

/* Row-level "Create Issue" action on a Risk — runs the custom action (which sets
   the Risk to Realised and creates a linked Issue), then soft-refreshes so the
   now-Realised Risk drops out of the list and the new Issue appears.

   Clicking the bug button opens a styled confirm popup (showConfirm); on confirm
   it runs against the Risk on the enclosing .row-action-cell. */
function createIssueButtonHtml() {
  return '<button class="row-action create-issue"' +
    ' title="Create Issue — sets this Risk to Realised and creates a linked Issue"' +
    ' aria-label="Create Issue from this risk"><i class="ti ti-bug" aria-hidden="true"></i></button>';
}
function confirmCreateIssue(btn) {
  if (!btn) return;
  var cell = btn.closest('.row-action-cell');
  if (!cell) return;
  var name = cell.getAttribute('data-name') || 'this risk';
  showConfirm({
    anchor: btn,
    title: 'Create Issue from Risk',
    message: 'Escalate "' + name + '" to an Issue? The risk becomes Realised, a linked Issue is created, and its actions are Completed.',
    confirmLabel: 'Create Issue',
    onConfirm: function () { runCreateIssue(cell); }
  });
}
/* Run the custom action against the Risk on this cell. */
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
    .then(function () { return loadAndRender(); })
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
  /* Mark actions done using whichever done-state the tenant actually has
     (real values only — no fabricated path). */
  var doneRaw = pickRawByLabel('ActionItemState', ['Closed', 'Completed', 'Complete', 'Done']);
  if (!doneRaw) return Promise.resolve();   /* no resolvable done-state in this env */
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
      if (!caseId) { return loadAndRender(); }   /* no id returned — nothing to link */
      return linkToProject(c.base, c.sid, caseId, c.projId)
        .then(function () { return loadAndRender(); });
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
      { type: 'static', html: 'Draft' },                   /* Status — new risks start as Draft */
      { type: 'text',   field: 'Title', placeholder: 'Risk name' },
      { type: 'static', html: '' },                        /* Description (icon appears once saved) */
      { type: 'pick',   field: 'C_Probability' },            /* Probability */
      { type: 'pick',   field: RISK_IMPACT_FIELD },         /* Impact */
      { type: 'static', html: '—' },                       /* Score (auto-calculated) */
      { type: 'user',   field: 'Owner' },
      { type: 'date',   field: 'DueDate', actions: false },
      (RISK_REPORTING_FIELD ? { type: 'pick', field: RISK_REPORTING_FIELD } : { type: 'static', html: '—' }),
      { type: 'date',   field: 'C_NextReviewDateR', actions: true }
    ]
  },
  Issue: {
    tbody: 'issues-body', label: 'issue',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'static', html: '' },                        /* Description (icon appears once saved) */
      { type: 'text',   field: 'Title', placeholder: 'Issue name' },
      { type: 'pick',   field: 'C_IssueImpact' },
      { type: 'static', html: '—' },                       /* Score (auto-calculated) */
      { type: 'pick',   field: 'State' },
      { type: 'user',   field: 'Owner' },
      { type: 'pick',   field: 'C_ReportingLevel' },
      { type: 'date',   field: 'DueDate', actions: true }
    ]
  },
  EnhancementRequest: {
    tbody: 'requests-body', label: 'change request',
    cells: [
      { type: 'static', html: '—' },                       /* ID (system-assigned) */
      { type: 'static', html: '' },                        /* Description (icon appears once saved) */
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
    /* Default the Owner to the signed-in user (already committed as a pick). */
    if (inp.getAttribute('data-field') === 'Owner' && CURRENT_USER.id) {
      inp.value = CURRENT_USER.name || '';
      inp.setAttribute('data-user-id', CURRENT_USER.id);
    }
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
  /* New Risks start with State = Draft. Use rawForLabel (not pickRawByLabel) so
     that on an empty panel — where no loaded record reveals a "Draft" path — we
     still derive /CaseState/Draft from the State prefix. Otherwise no State is
     sent and AdaptiveWork defaults the new risk to "Submitted". */
  if (entityType === 'Risk' && !fields.State) {
    var draftState = rawForLabel('State', 'Draft', '/CaseState/');
    if (draftState) fields.State = draftState;
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
    /* A Risk's Status offers only Draft / Open / Closed; other State cells (Issues,
       Requests) and all other picklists use the full environment options.
       Include a blank "—" option when the cell is currently empty, so the first
       real option isn't auto-selected — otherwise picking that first option fires
       no change event and the update never commits. */
    var blank = !value;
    var optsHtml = (field === 'State' && cell.closest && cell.closest('#risks-body'))
      ? riskStateOptionsHtml(value, blank)
      : pickOptionsHtml(field, value, blank);
    cell.innerHTML = '<select class="edit-input">' + optsHtml + '</select>';
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
      /* Impact/Probability drive the computed Score — soft-refresh to pick it up. */
      if (field === RISK_IMPACT_FIELD || field === 'C_Probability' || field === 'C_IssueImpact') { loadAndRender(); return; }
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

/* ---------- 16. Filtering ----------
   Search text AND the status dropdown combine (a row must match both). Re-applied
   after a sort re-render so an active filter isn't lost. */
function reapplyFilters(tab) {
  var cfg = TAB_CFG[tab];
  if (!cfg) return;
  var searchEl = document.getElementById('aw-search');
  var filterEl = document.getElementById('aw-filter');
  var q = (searchEl ? searchEl.value : '').toLowerCase();
  var st = filterEl ? filterEl.value : '';
  document.querySelectorAll('#' + cfg.tbody + ' .data-row').forEach(function (row) {
    var okText = !q || row.innerText.toLowerCase().indexOf(q) > -1;
    var okStatus = !st || (row.getAttribute('data-status') || '') === st;
    row.style.display = (okText && okStatus) ? '' : 'none';
  });
}

/* ---------- 16b. Column sorting (click a header) ----------
   Which header columns are sortable per tab, keyed by their <th> index:
   f = record field to sort on, t = 's' string | 'n' number | 'd' date.
   Blank/"—" values always sort to the bottom (regardless of direction). */
var SORT_COLS = {
  risks:  { 2: { f: 'status', t: 's' }, 3: { f: 'name', t: 's' }, 7: { f: 'riskRating', t: 'score' }, 8: { f: 'owner', t: 's' }, 9: { f: 'dueDate', t: 'd' }, 11: { f: 'nextReviewDate', t: 'd' } },
  issues: { 3: { f: 'name', t: 's' }, 5: { f: 'score',      t: 'score' }, 6: { f: 'status', t: 's' }, 7: { f: 'owner', t: 's' }, 9: { f: 'dueDate', t: 'd' } }
};
var SORT = { risks: { idx: null, dir: 1 }, issues: { idx: null, dir: 1 } };

/* Comparable numeric value for a Score / Risk Rating cell: use the number if the
   value is numeric, otherwise rank by heat-map severity (Low < Medium < High …). */
function scoreVal(v) {
  var s = String(v == null ? '' : v);
  var num = parseFloat(s.replace(/[^0-9.\-]/g, ''));
  if (!isNaN(num) && /\d/.test(s)) return num;
  var rank = { 'heatmap-very-low': 1, 'heatmap-low': 2, 'heatmap-medium': 3, 'heatmap-high': 4, 'heatmap-very-high': 5, 'heatmap-critical': 6 };
  return rank[riskRatingHeatMap(v)] || 0;
}
function sortIsEmpty(v, t) {
  if (v === null || v === undefined) return true;
  var s = String(v).trim();
  if (s === '' || s === '—') return true;
  if (t === 'n') return isNaN(parseFloat(s.replace(/[^0-9.\-]/g, '')));
  if (t === 'd') return isNaN(new Date(v).getTime());
  return false;
}
/* Sort DATA[tab] in place according to the tab's current SORT state. */
function applySort(tab) {
  var st = SORT[tab];
  if (!st || st.idx === null) return;
  var col = SORT_COLS[tab] && SORT_COLS[tab][st.idx];
  var list = DATA[tab];
  if (!col || !list) return;
  list.sort(function (a, b) {
    var av = a[col.f], bv = b[col.f];
    var ae = sortIsEmpty(av, col.t), be = sortIsEmpty(bv, col.t);
    if (ae && be) return 0;
    if (ae) return 1;            /* blanks last, either direction */
    if (be) return -1;
    var r;
    if (col.t === 'score') {
      r = scoreVal(av) - scoreVal(bv);
    } else if (col.t === 'n') {
      r = parseFloat(String(av).replace(/[^0-9.\-]/g, '')) - parseFloat(String(bv).replace(/[^0-9.\-]/g, ''));
    } else if (col.t === 'd') {
      r = new Date(av).getTime() - new Date(bv).getTime();
    } else {
      var as = String(av).toLowerCase(), bs = String(bv).toLowerCase();
      r = as < bs ? -1 : as > bs ? 1 : 0;
    }
    return r * st.dir;
  });
}
/* Re-sort + repaint a single tab's table (used on header click). */
function renderTab(tab) {
  applySort(tab);
  if (tab === 'risks') renderRisks(DATA.risks, DATA.actionMap);
  else if (tab === 'issues') renderIssues(DATA.issues, DATA.actionMap);
  updateSortIndicators(tab);
  reapplyFilters(tab);   /* keep any active search/status filter after a sort */
}
function toggleSort(tab, idx) {
  var st = SORT[tab];
  if (st.idx === idx) st.dir = -st.dir; else { st.idx = idx; st.dir = 1; }
  renderTab(tab);
}
/* Sortable columns show a faint ↕ hint; the active one shows an accent ▲/▼.
   (base label is cached on the th so the marker never accumulates). */
function updateSortIndicators(tab) {
  var tbody = document.getElementById(tab + '-body');
  var table = tbody && tbody.closest ? tbody.closest('table') : null;
  if (!table) return;
  var st = SORT[tab], cols = SORT_COLS[tab] || {};
  var ths = table.querySelectorAll('thead th');
  for (var i = 0; i < ths.length; i++) {
    var th = ths[i];
    var base = th.getAttribute('data-sort-base');
    if (base === null) { base = th.textContent.replace(/[\s↕▲▼]+$/, ''); th.setAttribute('data-sort-base', base); }
    if (!cols[i]) continue;
    var active = (st.idx === i);
    var marker = active ? (st.dir > 0 ? '▲' : '▼') : '↕';
    var mstyle = active ? 'color:#8B3A62;opacity:1;' : 'color:#9aa0a6;opacity:0.75;';
    th.innerHTML = esc(base) + ' <span class="sort-ind" style="font-size:9px;font-weight:400;' + mstyle + '">' + marker + '</span>';
  }
}
/* Make the sortable headers of a tab clickable (called once at init). */
function wireSort(tab) {
  var tbody = document.getElementById(tab + '-body');
  var table = tbody && tbody.closest ? tbody.closest('table') : null;
  if (!table) return;
  var cols = SORT_COLS[tab] || {};
  var ths = table.querySelectorAll('thead th');
  for (var i = 0; i < ths.length; i++) {
    (function (th, idx) {
      if (!cols[idx]) return;
      th.style.cursor = 'pointer';
      th.title = 'Sort by ' + th.textContent.trim();
      th.addEventListener('click', function () { toggleSort(tab, idx); });
    })(ths[i], i);
  }
  updateSortIndicators(tab);
}

/* ---------- 17. UI wiring ---------- */
/* Per-tab config for the single shared toolbar (search / status / New). */
var TAB_CFG = {
  risks:    { tbody: 'risks-body',    placeholder: 'Search risks…',    newLabel: 'New risk',    create: createNewRisk,    statuses: RISK_STATES },
  issues:   { tbody: 'issues-body',   placeholder: 'Search issues…',   newLabel: 'New issue',   create: createNewIssue,   statuses: ['Open', 'In progress', 'Resolved'] },
  requests: { tbody: 'requests-body', placeholder: 'Search change requests…', newLabel: 'New change request', create: createNewRequest, statuses: ['New', 'Pending', 'Approved', 'Rejected'] }
};
var ACTIVE_TAB = 'risks';

/* Whether every action panel on a tab is currently expanded (per tab), so the
   "Expand all" toolbar button can toggle + relabel, and the state can be
   re-applied after a soft refresh (which rebuilds the rows collapsed). */
var EXPANDED_ALL = { risks: false, issues: false, requests: false };

/* Expand or collapse every action panel on a tab at once. When expanding, only
   open rows that actually have action items (skip the "No actions recorded."
   ones); collapse always clears every row. */
function setAllExpanded(tab, expand) {
  var cfg = TAB_CFG[tab];
  if (!cfg) return;
  var tbody = document.getElementById(cfg.tbody);
  if (!tbody) return;
  tbody.querySelectorAll('.actions-row').forEach(function (row) {
    var want = expand && !!row.querySelector('.action-item');
    row.classList.toggle('show', want);
    var btn = tbody.querySelector('.expand-btn[data-target="' + row.id + '"]');
    if (btn) btn.classList.toggle('open', want);
  });
}
/* Toolbar "Expand all" — toggles all action panels on the active tab. */
function toggleExpandAll() {
  var tab = ACTIVE_TAB;
  EXPANDED_ALL[tab] = !EXPANDED_ALL[tab];
  setAllExpanded(tab, EXPANDED_ALL[tab]);
  updateExpandAllBtn();
}
/* Sync the button's label + icon to the active tab's expand state. */
function updateExpandAllBtn() {
  var btn = document.getElementById('btn-expand-all');
  if (!btn) return;
  var on = EXPANDED_ALL[ACTIVE_TAB];
  var lbl = btn.querySelector('.expand-all-label');
  if (lbl) lbl.textContent = on ? 'Collapse all' : 'Expand all';
  var icon = btn.querySelector('i');
  if (icon) icon.className = on ? 'ti ti-chevrons-up' : 'ti ti-chevrons-down';
  btn.setAttribute('aria-label', on ? 'Collapse all action items' : 'Expand all action items');
}

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

  updateExpandAllBtn();   /* reflect the newly-active tab's expand state */
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
      var aStatus = e.target.closest('.act-status');
      if (aStatus) { openActionStatus(aStatus.closest('.action-item')); return; }

      /* Row-level Create Issue action (Risks tab) — styled confirm popup */
      var ci = e.target.closest('.create-issue');
      if (ci) { confirmCreateIssue(ci); return; }

      /* Description icon — open the textarea editor */
      var db = e.target.closest('.desc-btn');
      if (db) { openDescription(db.getAttribute('data-id')); return; }

      var editable = e.target.closest('.editable');
      if (editable && !editable.classList.contains('editing')) { startEdit(editable); return; }
    });
  });

  var search = document.getElementById('aw-search');
  if (search) search.addEventListener('input', function () { reapplyFilters(ACTIVE_TAB); });

  var filter = document.getElementById('aw-filter');
  if (filter) filter.addEventListener('change', function () { reapplyFilters(ACTIVE_TAB); });

  wireSort('risks');
  wireSort('issues');

  var nb = document.getElementById('btn-new');
  if (nb) nb.addEventListener('click', function () { TAB_CFG[ACTIVE_TAB].create(); });

  var eab = document.getElementById('btn-expand-all');
  if (eab) eab.addEventListener('click', toggleExpandAll);

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
    rows.push([r.sysId, r.name, r.probability, r.impact, r.riskRating, r.status, r.owner, r.reportingLevel, exportDateCell(r.dueDate), exportDateCell(r.nextReviewDate)]);
    kinds.push('record');
    (DATA.actionMap[r.rawId] || []).forEach(function (a) { rows.push(exportActionRow(a, 10)); kinds.push('action'); });
  });
  return { rows: rows, kinds: kinds };
}
function exportRowsIssues() {
  var rows = [], kinds = [];
  DATA.issues.forEach(function (i) {
    rows.push([i.sysId, i.name, i.impact, i.score, i.status, i.owner, i.reportingLevel, exportDateCell(i.dueDate)]);
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
      ['ID', 'Risk name', 'Probability', 'Impact', 'Score', 'Status', 'Owner', 'Reporting Level', 'Due Date', 'Next Review Date'],
      exportRowsRisks(), 4) +
    buildSheetXml('Issues',
      ['ID', 'Issue name', 'Impact', 'Score', 'Status', 'Owner', 'Reporting Level', 'Due Date'],
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
      { hdr: 'Probability',     api: 'C_Probability',     kind: 'pick' },
      { hdr: 'Impact',          api: 'C_ImpactR',        kind: 'pick' },
      { hdr: 'Status',          api: 'State',            kind: 'pick' },
      { hdr: 'Owner',           api: 'Owner',            kind: 'user' },
      { hdr: 'Reporting Level', api: 'C_ReportingLevel', kind: 'pick' },
      { hdr: 'Due Date',        api: 'DueDate',            kind: 'date' },
      { hdr: 'Next Review Date', api: 'C_NextReviewDateR', kind: 'date' }
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
      { hdr: 'Due Date',        api: 'DueDate',          kind: 'date' }
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

/* Current raw value of an API field on a loaded record (for change detection). */
function recCurrent(entityType, rec, api) {
  if (api === 'Title') return rec.name === '(unnamed)' ? '' : rec.name;
  if (entityType === 'Risk') {
    if (api === 'C_Probability')     return rec.probabilityRaw;
    if (api === RISK_IMPACT_FIELD)  return rec.impactRaw;
    if (api === 'State')            return rec.statusRaw;
    if (api === 'Owner')            return rec.ownerRaw;
    if (RISK_REPORTING_FIELD && api === RISK_REPORTING_FIELD) return rec.reportingLevelRaw;
    if (api === 'DueDate')          return rec.dueDate || '';
    if (api === RISK_NEXTREVIEW_FIELD) return rec.nextReviewDate || '';
  } else if (entityType === 'Issue') {
    if (api === 'C_IssueImpact')    return rec.impactRaw;
    if (api === 'State')            return rec.statusRaw;
    if (api === 'Owner')            return rec.ownerRaw;
    if (api === 'C_ReportingLevel') return rec.reportingLevelRaw;
    if (api === 'DueDate')          return rec.dueDate || '';
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
    loadAndRender();
  });
}

/* ---------- 18. Load + render ----------
   Queries all three entities (+ their actions) and repaints the tables in place.
   Used for the initial load AND as a "soft refresh" after any change — replacing
   the old location.reload(), so there's no full-page reload: the active tab and
   scroll position are preserved and there's no white flash. Returns the promise. */
function loadAndRender() {
  var c;
  try {
    c = getContext();
  } catch (e) {
    showError('risks-body', 13, e.message);
    showError('issues-body', 10, e.message);
    showError('requests-body', 8, e.message);
    return Promise.reject(e);
  }

  var qRisks =
    "SELECT SYSID, Title, C_Probability, " + RISK_IMPACT_FIELD + ", C_RiskRating, State, Owner.Name, " +
    (RISK_REPORTING_FIELD ? RISK_REPORTING_FIELD + ", " : "") +
    "DueDate, C_NextReviewDateR, Description FROM Risk WHERE PlannedFor = '" + c.projId + "'";
  var qIssues =
    "SELECT SYSID, Title, C_IssueImpact, C_IssueScore, State, Owner.Name, C_ReportingLevel, DueDate, Description" +
    " FROM Issue WHERE PlannedFor = '" + c.projId + "'";
  var qRequests =
    "SELECT SYSID, Title, RequestType, State, CreatedBy.Name, CreatedOn, DueDate, Description " +
    "FROM EnhancementRequest WHERE PlannedFor = '" + c.projId + "'";

  return Promise.all([
    czql(c.base, c.sid, qRisks),
    czql(c.base, c.sid, qIssues),
    czql(c.base, c.sid, qRequests)
  ])
  .then(function (results) {
    /* Risks tab shows only the displayed states (hide Realised, etc.). */
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
      "SELECT Name, EntityOwner.Name, DueDate, ActionItemState, C_SummaryUpdateAction, Container.id " +
      "FROM ActionItem WHERE Container IN (" + inList + ")";

    return czql(c.base, c.sid, qActions).then(function (actionEntities) {
      var actionMap = buildActionMap(actionEntities.map(toAction));
      renderAll(risks, issues, requests, actionMap);
    });
  })
  .catch(function (err) {
    console.error('AdaptiveWork panel fetch error:', err);
    var msg = 'Failed to load data: ' + err.message;
    showError('risks-body', 13, msg);
    showError('issues-body', 10, msg);
    showError('requests-body', 8, msg);
  });
}

/* ---------- 19. Bootstrap ---------- */
setTimeout(function () {
  initUI();

  /* Validate the injected context up front; each loader re-reads it as needed. */
  try {
    getContext();
  } catch (e) {
    showError('risks-body', 13, e.message);
    showError('issues-body', 10, e.message);
    showError('requests-body', 8, e.message);
    return;
  }

  showLoading('risks-body', 13);
  showLoading('issues-body', 10);
  showLoading('requests-body', 8);

  /* Load real picklist option paths (non-blocking). Query the custom picklist
     type entities for their values, and also try describeEntities metadata. */
  loadPicklistValues().catch(function (err) {
    console.warn('Picklist value query failed:', err && err.message);
  });
  loadPicklistMeta().catch(function (err) {
    console.warn('Picklist metadata unavailable, falling back to data-derived options:', err && err.message);
  });
  loadCurrentUser().catch(function (err) {
    console.warn('Current user unavailable:', err && err.message);
  });

  loadAndRender();
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
| Impact / Probability dropdowns only listed values already in use | Canonical, fully-ordered option lists for `C_Impact` (1 - Minor … 6 - Severe) and `C_Probability` (1 - Unlikely … 6 - Very Likely) always show; raw `/Type/Value` path is reused from data or rebuilt from the detected prefix. "Likelihood" column header renamed to "Probability" |
| Owner was read-only text | Owner (`AssignedTo`) is now an editable **type-ahead** of system users — searches server-side (`User WHERE Name LIKE '%…%'`, debounced, 2+ chars) instead of bulk-loading everyone. Works on existing rows and the new-item draft row; sends the `/User/…` id |
```
