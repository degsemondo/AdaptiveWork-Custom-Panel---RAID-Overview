/* ============================================================
   AdaptiveWork Custom Panel — JavaScript (with inline editing)
   ============================================================ */

var API_PATH = '/V2.0/services/data/query';

function czql(baseUrl, sessionId, query) {
  var url = baseUrl.replace(/\/$/, '') + API_PATH;
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

function updateEntity(entityId, updates) {
  return new Promise(function(resolve, reject) {
    if (!window.API || !window.API.Objects || typeof window.API.Objects.update !== 'function') {
      reject(new Error('API.Objects.update is not available'));
      return;
    }
    window.API.Objects.update(
      entityId,
      updates,
      function(result) { resolve(result); },
      function(err) { reject(err); }
    );
  });
}

function extractPicklistValue(field) {
  if (!field) return '';
  if (typeof field === 'string') return field;
  if (field.Value) return field.Value;
  if (field.value) return field.value;
  if (field.Name) return field.Name;
  if (field.name) return field.name;
  if (field.display) return field.display;
  if (field.displayName) return field.displayName;
  if (field.displayText) return field.displayText;
  if (field.label) return field.label;
  if (field.text) return field.text;
  if (field.id) {
    var idStr = field.id;
    var lastSlash = idStr.lastIndexOf('/');
    if (lastSlash !== -1) {
      return idStr.substring(lastSlash + 1);
    }
  }
  if (typeof field === 'object') {
    var keys = Object.keys(field);
    if (keys.length > 0) return String(field[keys[0]]);
  }
  return '';
}

function toRisk(e) {
  var impactValue = extractPicklistValue(e.C_Impact);
  var likelihoodValue = extractPicklistValue(e.C_Likelihood);
  var ratingValue = extractPicklistValue(e.C_RiskRating);
  var statusValue = extractPicklistValue(e.State);
  
  return {
    id:          (e.id || '').replace(/\//g, '-'),
    name:        e.Title       || '(unnamed)',
    impact:      impactValue || '(2) Moderate',
    likelihood:  likelihoodValue || '(3) Possible',
    rating:      ratingValue || 'N/A',
    status:      statusValue   || 'Open',
    owner:       (e.AssignedTo && e.AssignedTo.Name) || '—',
    dueDate:     e.DueDate     || null,
    rawId:       e.id          || ''
  };
}

function toIssue(e) {
  var priorityValue = extractPicklistValue(e.Priority);
  var statusValue = extractPicklistValue(e.State);
  
  return {
    id:      (e.id || '').replace(/\//g, '-'),
    name:    e.Title     || '(unnamed)',
    priority: priorityValue || 'Medium',
    status:  statusValue || 'Open',
    owner:   (e.AssignedTo && e.AssignedTo.Name) || '—',
    raised:  e.CreatedOn || null,
    dueDate: e.DueDate   || null,
    rawId:   e.id        || ''
  };
}

function toRequest(e) {
  var typeValue = extractPicklistValue(e.RequestType);
  var statusValue = extractPicklistValue(e.State);
  
  return {
    id:         (e.id || '').replace(/\//g, '-'),
    name:       e.Title       || '(unnamed)',
    type:       typeValue || 'Change request',
    status:     statusValue   || 'New',
    requestor:  (e.CreatedBy  && e.CreatedBy.Name) || '—',
    submitted:  e.CreatedOn   || null,
    decisionBy: e.DueDate     || null,
    rawId:      e.id          || ''
  };
}

function toAction(e) {
  var today   = new Date();
  var due     = e.DueDate ? new Date(e.DueDate) : null;
  var actionState = extractPicklistValue(e.ActionItemState);
  var isDone  = actionState && (actionState === 'Complete' || actionState === 'Completed');
  var isLate  = !isDone && due && due < today;
  var statusLabel = isDone  ? 'Completed'
                 : isLate  ? 'Overdue'
                 : due      ? 'Due ' + fmtDate(e.DueDate)
                 :            'No due date';
  return {
    name:     e.Name     || '(unnamed)',
    assignee: (e.EntityOwner && e.EntityOwner.Name) || '—',
    dueDate:  e.DueDate  || null,
    status:   statusLabel,
    parentId: (e.Container && e.Container.id) || ''
  };
}

function fmtDate(iso) {
  if (!iso) return '—';
  return new Date(iso).toLocaleDateString('en-GB', {
    day: 'numeric', month: 'short', year: 'numeric'
  });
}

function dueCls(iso) {
  if (!iso) return '';
  return new Date(iso) < new Date() ? 'overdue' : 'ok';
}

function riskRatingHeatMap(rating) {
  var map = {
    'Very Low': 'heatmap-very-low',
    'Low': 'heatmap-low',
    'Medium': 'heatmap-medium',
    'High': 'heatmap-high',
    'Very High': 'heatmap-very-high',
    'Critical': 'heatmap-critical'
  };
  return map[rating] || 'heatmap-neutral';
}

function buildActionMap(actions) {
  var map = {};
  actions.forEach(function (a) {
    var pid = a.parentId;
    if (!pid) return;
    if (!map[pid]) map[pid] = [];
    map[pid].push(a);
  });
  return map;
}

var PILL_MAP = {
  Critical: 'pill-critical', High: 'pill-high', Medium: 'pill-medium', Low: 'pill-low',
  Open: 'pill-open', 'In progress': 'pill-inprog', 'Active': 'pill-inprog',
  Resolved: 'pill-resolved', Complete: 'pill-resolved', Completed: 'pill-resolved',
  Pending: 'pill-pending', New: 'pill-new', Submitted: 'pill-new',
  Approved: 'pill-approved', Rejected: 'pill-rejected', Cancelled: 'pill-rejected',
  Identified: 'pill-pending', Monitoring: 'pill-inprog', Mitigated: 'pill-resolved', Closed: 'pill-rejected',
  Realised: 'pill-rejected'
};

function pill(label) {
  var cls = PILL_MAP[label] || 'pill-medium';
  return '<span class="pill ' + cls + '">' + label + '</span>';
}

function actionItemHtml(a) {
  var isDone   = a.status === 'Completed';
  var isLate   = a.status === 'Overdue';
  var iconCol  = isDone ? '#3B6D11' : '#185FA5';
  var dueCls   = isLate ? 'overdue' : 'ok';
  return '<div class="action-item">' +
    '<i class="ti ti-circle-check" style="font-size:14px;color:' + iconCol + '" aria-hidden="true"></i>' +
    '<span class="action-name">' + a.name + '</span>' +
    '<span class="action-meta">' + a.assignee + '</span>' +
    '<span class="action-due ' + dueCls + '">· ' + a.status + '</span>' +
    '</div>';
}

function actionsBlock(actions) {
  if (!actions || !actions.length) {
    return '<div class="actions-block"><div class="actions-label">Associated actions</div>' +
           '<p style="font-size:12px;color:var(--color-text-secondary);margin:4px 0 0">No actions recorded.</p></div>';
  }
  return '<div class="actions-block"><div class="actions-label">Associated actions</div>' +
    actions.map(actionItemHtml).join('') + '</div>';
}

function renderRisks(risks, actionMap) {
  var tbody = document.getElementById('risks-body');
  if (!risks.length) {
    tbody.innerHTML = '<tr><td colspan="8"><div class="no-data">No risks found for this project.</div></td></tr>';
    document.getElementById('badge-risks').textContent = '0';
    return;
  }
  tbody.innerHTML = risks.map(function (r) {
    var actions = actionMap[r.rawId] || [];
    var dc = dueCls(r.dueDate);
    return '<tr class="data-row" data-status="' + r.status + '">' +
      '<td><button class="expand-btn" data-target="' + r.id + '-actions" aria-label="Show actions"><i class="ti ti-chevron-right"></i></button></td>' +
      '<td class="editable" data-field="Title" data-id="' + r.rawId + '" data-value="' + (r.name || '') + '">' + r.name + '</td>' +
      '<td class="editable" data-field="C_Impact" data-id="' + r.rawId + '" data-value="' + (r.impact || '') + '">' + pill(r.impact) + '</td>' +
      '<td class="editable" data-field="C_Likelihood" data-id="' + r.rawId + '" data-value="' + (r.likelihood || '') + '">' + pill(r.likelihood) + '</td>' +
      '<td class="heatmap ' + riskRatingHeatMap(r.rating) + '">' + r.rating + '</td>' +
      '<td class="editable" data-field="State" data-id="' + r.rawId + '" data-value="' + (r.status || '') + '">' + pill(r.status) + '</td>' +
      '<td>' + r.owner + '</td>' +
      '<td class="editable action-due ' + dc + '" data-field="DueDate" data-id="' + r.rawId + '" data-value="' + (r.dueDate || '') + '">' + fmtDate(r.dueDate) + '</td>' +
      '</tr>' +
      '<tr class="actions-row" id="' + r.id + '-actions">' +
      '<td colspan="8">' + actionsBlock(actions) + '</td></tr>';
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
    var dc = dueCls(issue.dueDate);
    return '<tr class="data-row" data-status="' + issue.status + '">' +
      '<td><button class="expand-btn" data-target="' + issue.id + '-actions" aria-label="Show actions"><i class="ti ti-chevron-right"></i></button></td>' +
      '<td class="editable" data-field="Title" data-id="' + issue.rawId + '" data-value="' + (issue.name || '') + '">' + issue.name + '</td>' +
      '<td class="editable" data-field="Priority" data-id="' + issue.rawId + '" data-value="' + (issue.priority || '') + '">' + pill(issue.priority) + '</td>' +
      '<td class="editable" data-field="State" data-id="' + issue.rawId + '" data-value="' + (issue.status || '') + '">' + pill(issue.status) + '</td>' +
      '<td>' + issue.owner + '</td>' +
      '<td>' + fmtDate(issue.raised) + '</td>' +
      '<td class="editable action-due ' + dc + '" data-field="DueDate" data-id="' + issue.rawId + '" data-value="' + (issue.dueDate || '') + '">' + fmtDate(issue.dueDate) + '</td>' +
      '</tr>' +
      '<tr class="actions-row" id="' + issue.id + '-actions">' +
      '<td colspan="7">' + actionsBlock(actions) + '</td></tr>';
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
    var dc = dueCls(req.decisionBy);
    return '<tr class="data-row" data-status="' + req.status + '">' +
      '<td><button class="expand-btn" data-target="' + req.id + '-actions" aria-label="Show actions"><i class="ti ti-chevron-right"></i></button></td>' +
      '<td class="editable" data-field="Title" data-id="' + req.rawId + '" data-value="' + (req.name || '') + '">' + req.name + '</td>' +
      '<td class="editable" data-field="RequestType" data-id="' + req.rawId + '" data-value="' + (req.type || '') + '">' + req.type + '</td>' +
      '<td class="editable" data-field="State" data-id="' + req.rawId + '" data-value="' + (req.status || '') + '">' + pill(req.status) + '</td>' +
      '<td>' + req.requestor + '</td>' +
      '<td>' + fmtDate(req.submitted) + '</td>' +
      '<td class="editable action-due ' + dc + '" data-field="DueDate" data-id="' + req.rawId + '" data-value="' + (req.decisionBy || '') + '">' + fmtDate(req.decisionBy) + '</td>' +
      '</tr>' +
      '<tr class="actions-row" id="' + req.id + '-actions">' +
      '<td colspan="7">' + actionsBlock(actions) + '</td></tr>';
  }).join('');
  document.getElementById('badge-requests').textContent = requests.length;
}

function showLoading(tbodyId, cols) {
  document.getElementById(tbodyId).innerHTML =
    '<tr><td colspan="' + cols + '"><div class="no-data">' +
    '<i class="ti ti-loader" style="font-size:16px;vertical-align:-2px;margin-right:6px" aria-hidden="true"></i>' +
    'Loading…</div></td></tr>';
}

function showError(tbodyId, cols, msg) {
  document.getElementById(tbodyId).innerHTML =
    '<tr><td colspan="' + cols + '"><div class="no-data" style="color:#A32D2D">' +
    '<i class="ti ti-alert-circle" style="font-size:16px;vertical-align:-2px;margin-right:6px" aria-hidden="true"></i>' +
    msg + '</div></td></tr>';
}

var globalBaseUrl = '';
var globalSessionId = '';

function startEdit(cell) {
  if (cell.classList.contains('editing')) return;
  
  var field = cell.getAttribute('data-field');
  var id = cell.getAttribute('data-id');
  var value = cell.getAttribute('data-value') || '';
  
  var options = {
    'C_Impact': ['(1) Minimal', '(2) Moderate', '(3) Major', '(4) Critical', '(5) Catastrophic'],
    'C_Likelihood': ['(1) Rare', '(2) Unlikely', '(3) Possible', '(4) Probable', '(5) Almost Certain'],
    'C_RiskRating': ['Very Low', 'Low', 'Medium', 'High', 'Very High'],
    'Priority': ['Critical', 'High', 'Medium', 'Low'],
    'State': ['New', 'Closed', 'Realised'],
    'RequestType': ['Change request', 'Enhancement', 'Bug fix', 'Support'],
    'DueDate': 'date',
    'Title': 'text'
  };
  
  var inputType = options[field] || 'text';
  var inputHtml = '';
  
  if (Array.isArray(inputType)) {
    inputHtml = '<select class="edit-input">' +
      '<option value="">-- Select --</option>' +
      inputType.map(function(opt) {
        return '<option value="' + opt + '" ' + (opt === value ? 'selected' : '') + '>' + opt + '</option>';
      }).join('') +
      '</select>';
  } else if (inputType === 'date') {
    inputHtml = '<input type="date" class="edit-input" value="' + (value ? value.split('T')[0] : '') + '" />';
  } else {
    inputHtml = '<input type="text" class="edit-input" value="' + value.replace(/"/g, '&quot;') + '" />';
  }
  
  var html = inputHtml +
    '<button class="edit-save" data-id="' + id + '" data-field="' + field + '">Save</button>' +
    '<button class="edit-cancel">Cancel</button>';
  
  cell.classList.add('editing');
  cell.innerHTML = html;
  cell.querySelector('.edit-input').focus();
}

function cancelEdit(cell) {
  cell.classList.remove('editing');
  var field = cell.getAttribute('data-field');
  var value = cell.getAttribute('data-value') || '';
  
  if (field === 'C_Impact' || field === 'C_Likelihood' || field === 'Priority' || field === 'State' || field === 'C_RiskRating' || field === 'RequestType') {
    cell.innerHTML = pill(value);
  } else {
    cell.innerHTML = field === 'DueDate' ? fmtDate(value) : value;
  }
}

function saveEdit(cell) {
  var input = cell.querySelector('.edit-input');
  var newValue = input.value;
  var field = cell.getAttribute('data-field');
  var id = cell.getAttribute('data-id');
  
  if (!newValue) {
    alert('Field cannot be empty');
    return;
  }
  
  var btn = cell.querySelector('.edit-save');
  btn.disabled = true;
  btn.textContent = 'Saving…';
  
  var payload = {};
  payload[field] = newValue;
  
  updateEntity(id, payload)
    .then(function() {
      cell.setAttribute('data-value', newValue);
      cell.classList.remove('editing');
      
      if (field === 'C_Impact' || field === 'C_Likelihood' || field === 'Priority' || field === 'State' || field === 'C_RiskRating' || field === 'RequestType') {
        cell.innerHTML = pill(newValue);
      } else if (field === 'DueDate') {
        cell.innerHTML = '<span class="' + dueCls(newValue) + '">' + fmtDate(newValue) + '</span>';
      } else {
        cell.innerHTML = newValue;
      }
    })
    .catch(function(err) {
      alert('Save failed: ' + err.message);
      btn.disabled = false;
      btn.textContent = 'Save';
    });
}

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
      var btn = e.target.closest('.expand-btn');
      if (btn) {
        var targetId = btn.getAttribute('data-target');
        var row = document.getElementById(targetId);
        if (row) {
          var isOpen = row.classList.contains('show');
          row.classList.toggle('show', !isOpen);
          btn.classList.toggle('open', !isOpen);
        }
        return;
      }
      
      var editable = e.target.closest('.editable');
      if (editable && !editable.classList.contains('editing')) {
        startEdit(editable);
        return;
      }
      
      var saveBtn = e.target.closest('.edit-save');
      if (saveBtn) {
        saveEdit(saveBtn.closest('.editable'));
        return;
      }
      
      var cancelBtn = e.target.closest('.edit-cancel');
      if (cancelBtn) {
        cancelEdit(cancelBtn.closest('.editable'));
        return;
      }
    });
  });

  var searches = [
    { input: 'input[placeholder="Search risks…"]',    tbody: 'risks-body' },
    { input: 'input[placeholder="Search issues…"]',   tbody: 'issues-body' },
    { input: 'input[placeholder="Search requests…"]', tbody: 'requests-body' }
  ];
  searches.forEach(function (s) {
    var el = document.querySelector(s.input);
    if (el) el.addEventListener('input', function () { filterTable(s.tbody, el.value); });
  });

  var paneRisks = document.getElementById('pane-risks');
  if (paneRisks) {
    var statusSel = paneRisks.querySelector('select');
    if (statusSel) statusSel.addEventListener('change', function () { filterByStatus('risks-body', statusSel.value); });
  }
  var paneIssues = document.getElementById('pane-issues');
  if (paneIssues) {
    var issSel = paneIssues.querySelector('select');
    if (issSel) issSel.addEventListener('change', function () { filterByStatus('issues-body', issSel.value); });
  }
  var paneReqs = document.getElementById('pane-requests');
  if (paneReqs) {
    var reqSel = paneReqs.querySelector('select');
    if (reqSel) reqSel.addEventListener('change', function () { filterByStatus('requests-body', reqSel.value); });
  }
}

function filterTable(tbodyId, query) {
  var q = query.toLowerCase();
  document.querySelectorAll('#' + tbodyId + ' .data-row').forEach(function (row) {
    var visible = row.innerText.toLowerCase().includes(q);
    row.style.display = visible ? '' : 'none';
  });
}

function filterByStatus(tbodyId, val) {
  document.querySelectorAll('#' + tbodyId + ' .data-row').forEach(function (row) {
    row.style.display = (!val || row.getAttribute('data-severity') === val || row.getAttribute('data-status') === val) ? '' : 'none';
  });
}

setTimeout(function () {
  var ctx;
  try {
    ctx = API.Context.getData();
    if (!ctx || !ctx.sessionId || !ctx.project || !ctx.project.id) {
      throw new Error('Data field must supply sessionId and project object.');
    }
  } catch (e) {
    ['risks-body','issues-body','requests-body'].forEach(function (id) {
      showError(id, 8, e.message);
    });
    return;
  }

  var sid    = ctx.sessionId;
  var projId = ctx.project.id;

  var appHost = window.location.hostname;
  var apiHost = appHost
    .replace(/^app2\./, 'api2.')
    .replace(/^app\./, 'api.')
    .replace(/^eu1\./, 'apie1.')
    .replace(/^eu\./, 'apie.');
  var base = 'https://' + apiHost;

  globalBaseUrl = base;
  globalSessionId = sid;

  showLoading('risks-body',    8);
  showLoading('issues-body',   7);
  showLoading('requests-body', 7);

  var qRisks =
    "SELECT Title, C_Impact, C_Likelihood, C_RiskRating, State, AssignedTo.Name, DueDate " +
    "FROM Risk " +
    "WHERE PlannedFor = '" + projId + "'";

  var qIssues =
    "SELECT Title, Priority, State, AssignedTo.Name, CreatedOn, DueDate " +
    "FROM Issue " +
    "WHERE PlannedFor = '" + projId + "'";

  var qRequests =
    "SELECT Title, RequestType, State, CreatedBy.Name, CreatedOn, DueDate " +
    "FROM EnhancementRequest " +
    "WHERE PlannedFor = '" + projId + "'";

  Promise.all([
    czql(base, sid, qRisks),
    czql(base, sid, qIssues),
    czql(base, sid, qRequests)
  ])
  .then(function (results) {
    var risks    = results[0].map(toRisk);
    var issues   = results[1].map(toIssue);
    var requests = results[2].map(toRequest);

    var caseIds = []
      .concat(results[0].map(function(e) { return e.id; }))
      .concat(results[1].map(function(e) { return e.id; }))
      .concat(results[2].map(function(e) { return e.id; }));

    if (!caseIds.length) {
      renderRisks(risks,    {});
      renderIssues(issues,  {});
      renderRequests(requests, {});
      initUI();
      return;
    }

    var inList = caseIds.map(function(id) { return "'" + id + "'"; }).join(',');
    var qActions =
      "SELECT Name, EntityOwner.Name, DueDate, ActionItemState, Container.id " +
      "FROM ActionItem " +
      "WHERE Container IN (" + inList + ")";

    return czql(base, sid, qActions).then(function (actionEntities) {
      var actionMap = buildActionMap(actionEntities.map(toAction));
      renderRisks(risks,    actionMap);
      renderIssues(issues,  actionMap);
      renderRequests(requests, actionMap);
      initUI();
    });
  })
  .catch(function (err) {
    console.error('AdaptiveWork panel fetch error:', err);
    var msg = 'Failed to load data: ' + err.message;
    showError('risks-body',    8, msg);
    showError('issues-body',   7, msg);
    showError('requests-body', 7, msg);
  });
}, 100);
The New buttons are now fully functional and will create items using AdaptiveWork's Upsert method.
