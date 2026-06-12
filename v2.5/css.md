# CSS Field - v2.0 (Ultra-Compact)

```css
/* ============================================================
   AdaptiveWork Custom Panel — Professional Planview Design
   ULTRA-COMPACT VERSION
   ============================================================ */

.aw-panel {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen', 'Ubuntu', 'Cantarell', sans-serif;
  font-size: 12px;
  color: #2c3e50;
  padding: 0;
  background: #fff;
  border-radius: 4px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
}

.aw-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  border-bottom: 1px solid #e8eaed;
  background: #fafbfc;
  border-radius: 4px 4px 0 0;
}

.aw-header-title {
  font-size: 14px;
  font-weight: 600;
  margin-right: auto;
  display: flex;
  align-items: center;
  gap: 8px;
  color: #1a1a1a;
}

.aw-header-title i {
  color: #8B3A62;
}

.aw-tabs {
  display: flex;
  gap: 0px;
}

.aw-tab {
  padding: 6px 12px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  border: none;
  background: transparent;
  color: #5f6368;
  border-bottom: 2px solid transparent;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: -2px;
}

.aw-tab:hover {
  color: #202124;
  background: rgba(0, 0, 0, 0.02);
}

.aw-tab.active {
  color: #8B3A62;
  border-bottom-color: #8B3A62;
}

.aw-badge {
  font-size: 10px;
  font-weight: 600;
  padding: 1px 6px;
  border-radius: 10px;
  min-width: 18px;
  text-align: center;
}

.badge-risk    { background: #f3e5e8; color: #8B3A62; }
.badge-issue   { background: #fce4ec; color: #c2185b; }
.badge-request { background: #e3f2fd; color: #1565c0; }

.aw-body { padding: 0; }
.aw-tab-pane         { display: none; }
.aw-tab-pane.active  { display: block; }

/* Shared toolbar now lives inline in the header row, pushed to the right. */
.aw-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
}

.aw-toolbar input,
.aw-toolbar select {
  font-size: 12px;
  padding: 5px 8px;
  border-radius: 3px;
  border: 1px solid #dadce0;
  background: #fff;
  color: #202124;
  transition: all 0.2s ease;
}

.aw-toolbar input:focus,
.aw-toolbar select:focus {
  outline: none;
  border-color: #8B3A62;
  box-shadow: 0 0 0 3px rgba(139, 58, 98, 0.1);
}

.aw-btn {
  font-size: 12px;
  font-weight: 500;
  padding: 5px 12px;
  border-radius: 3px;
  border: 1px solid #dadce0;
  background: #fff;
  color: #202124;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.aw-btn:hover {
  background: #f8f9fa;
  border-color: #bdc1c6;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12);
}

.aw-btn-primary {
  background: #8B3A62;
  color: #fff;
  border-color: #8B3A62;
}

.aw-btn-primary:hover {
  background: #6d2d4c;
  border-color: #6d2d4c;
  box-shadow: 0 2px 4px rgba(139, 58, 98, 0.2);
}

.aw-table-wrap {
  overflow-x: auto;
  padding: 8px 12px;
}

table.aw-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
  line-height: 1.1;
}

table.aw-table th {
  text-align: left;
  padding: 4px 8px;
  font-weight: 600;
  font-size: 11px;
  color: #5f6368;
  border-bottom: 2px solid #e8eaed;
  background: #fafbfc;
  white-space: nowrap;
  letter-spacing: 0.3px;
  height: 24px;
}

table.aw-table td {
  padding: 4px 8px;
  border-bottom: 1px solid #f0f0f0;
  vertical-align: middle;
  color: #202124;
  height: 24px;
}

table.aw-table tbody tr:hover {
  background: #f8f9fa;
}

.pill {
  display: inline-flex;
  align-items: center;
  padding: 2px 6px;
  border-radius: 10px;
  font-size: 11px;
  font-weight: 500;
  white-space: nowrap;
}

.pill-critical { background: #fce4ec; color: #c2185b; }
.pill-high     { background: #fff3e0; color: #e65100; }
.pill-medium   { background: #fce4ec; color: #ad1457; }
.pill-low      { background: #e8f5e9; color: #2e7d32; }
.pill-open     { background: #fce4ec; color: #c2185b; }
.pill-inprog   { background: #e3f2fd; color: #1565c0; }
.pill-resolved { background: #e8f5e9; color: #2e7d32; }
.pill-pending  { background: #fff3e0; color: #e65100; }
.pill-new      { background: #f3e5f5; color: #6a1b9a; }
.pill-approved { background: #e8f5e9; color: #2e7d32; }
.pill-rejected { background: #fce4ec; color: #c2185b; }

.expand-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #8B3A62;
  font-size: 16px;
  padding: 4px 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  width: 28px;
  height: 24px;
  font-weight: 600;
}

.expand-btn::before {
  content: '▶';
  font-size: 12px;
}

.expand-btn:hover {
  color: #6d2d4c;
  background: rgba(139, 58, 98, 0.08);
  border-radius: 3px;
}

.expand-btn.open {
  transform: rotate(90deg);
}

.actions-row       { display: none; }
.actions-row.show  { display: table-row; }

.actions-block {
  padding: 6px 10px;
  background: #fafbfc;
}

.actions-label {
  font-size: 10px;
  font-weight: 600;
  color: #5f6368;
  margin-bottom: 4px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.action-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 4px 8px;
  border-radius: 3px;
  background: #fff;
  border: 1px solid #e8eaed;
  margin-bottom: 3px;
  font-size: 12px;
}

.action-item:last-child { margin-bottom: 0; }

.action-name { font-size: 12px; font-weight: 500; color: #202124; flex: 1; }
.action-meta { font-size: 11px; color: #5f6368; }

.action-due          { font-size: 11px; color: #5f6368; }
.action-due.overdue  { color: #d32f2f; font-weight: 500; }
.action-due.ok       { color: #388e3c; font-weight: 500; }

/* ── Action CRUD ── */
.actions-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}
.actions-head .actions-label { margin-bottom: 0; }
.act-add {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 11px;
  font-weight: 500;
  padding: 2px 8px;
  border: 1px solid #dadce0;
  border-radius: 3px;
  background: #fff;
  color: #8B3A62;
  cursor: pointer;
}
.act-add:hover { background: #f8f9fa; border-color: #bdc1c6; }
.action-empty { font-size: 12px; color: #5f6368; margin: 4px 0 0; }

.action-tools {
  margin-left: auto;
  display: inline-flex;
  gap: 4px;
}
.act-edit, .act-del {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 22px;
  border: 1px solid #dadce0;
  background: #fff;
  cursor: pointer;
  border-radius: 3px;
  font-size: 14px;
  line-height: 1;
}
.act-edit { color: #8B3A62; }
.act-del  { color: #c2185b; }
.act-edit:hover { background: rgba(139, 58, 98, 0.10); border-color: #8B3A62; }
.act-del:hover  { background: #fce4ec; border-color: #c2185b; }

.action-edit {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  width: 100%;
}
.action-edit .act-f-name { flex: 1; min-width: 140px; }
.action-edit .ta-cell { position: relative; }
.act-save, .act-cancel {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: 2px;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
}
.act-save { background: #8B3A62; color: #fff; }
.act-save:hover:not(:disabled) { background: #6d2d4c; }
.act-save:disabled { opacity: 0.5; cursor: not-allowed; }
.act-cancel { background: #f0f0f0; color: #202124; }
.act-cancel:hover { background: #e8e8e8; }

.no-data {
  padding: 24px 20px;
  text-align: center;
  color: #80868b;
  font-size: 12px;
}

.editable {
  cursor: pointer;
  position: relative;
  border-radius: 2px;
  transition: all 0.2s ease;
}

.editable:hover:not(.editing) {
  background: rgba(139, 58, 98, 0.06);
}

.editable.editing {
  padding: 2px 4px;
  background: #fafbfc;
  border-radius: 3px;
}

.edit-input {
  font-size: 12px;
  padding: 3px 5px;
  border: 1px solid #8B3A62;
  border-radius: 3px;
  margin-right: 4px;
  min-width: 80px;
  font-family: inherit;
  color: #202124;
}

.edit-input:focus {
  outline: none;
  border-color: #8B3A62;
  box-shadow: 0 0 0 2px rgba(139, 58, 98, 0.1);
}

.edit-save, .edit-cancel {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: 2px;
  border: none;
  cursor: pointer;
  margin-right: 3px;
  transition: all 0.2s ease;
}

.edit-save {
  background: #8B3A62;
  color: white;
}

.edit-save:hover:not(:disabled) {
  background: #6d2d4c;
  box-shadow: 0 2px 4px rgba(139, 58, 98, 0.2);
}

.edit-save:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.edit-cancel {
  background: #f0f0f0;
  color: #202124;
}

.edit-cancel:hover {
  background: #e8e8e8;
}

/* ── Inline "new item" draft row ── */
.new-row > td {
  background: #fafbfc;
  vertical-align: top;
  padding-top: 8px;
  padding-bottom: 8px;
}
.new-row .new-input {
  width: 100%;
  box-sizing: border-box;
  margin-right: 0;
}
.new-row-actions {
  margin-top: 6px;
  display: flex;
  gap: 4px;
}
.new-save, .new-cancel {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: 2px;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
}
.new-save {
  background: #8B3A62;
  color: #fff;
}
.new-save:hover:not(:disabled) {
  background: #6d2d4c;
  box-shadow: 0 2px 4px rgba(139, 58, 98, 0.2);
}
.new-save:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.new-cancel {
  background: #f0f0f0;
  color: #202124;
}
.new-cancel:hover {
  background: #e8e8e8;
}

/* ── User type-ahead ── */
.ta-cell { position: relative; }
.ta-input { min-width: 120px; }
.ta-menu {
  position: absolute;
  z-index: 50;
  left: 4px;
  top: 100%;
  min-width: 160px;
  max-height: 180px;
  overflow-y: auto;
  background: #fff;
  border: 1px solid #dadce0;
  border-radius: 3px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}
.ta-item {
  padding: 5px 8px;
  font-size: 12px;
  cursor: pointer;
  white-space: nowrap;
}
.ta-item:hover {
  background: rgba(139, 58, 98, 0.08);
}
.ta-empty {
  padding: 5px 8px;
  font-size: 12px;
  color: #80868b;
}

.heatmap {
  padding: 4px 8px;
  text-align: center;
  color: #fff;
  font-weight: 600;
  border-radius: 3px;
  font-size: 11px;
  line-height: 1.1;
}

.heatmap-very-low { background: #4caf50; }
.heatmap-low { background: #8bc34a; }
.heatmap-medium { background: #ff9800; }
.heatmap-high { background: #ff7043; }
.heatmap-very-high { background: #f4511e; }
.heatmap-critical { background: #d32f2f; }
.heatmap-neutral { background: #9e9e9e; }

/* ── ID deep link ── */
.id-link {
  color: #8B3A62;
  font-weight: 600;
  text-decoration: none;
}
.id-link:hover {
  text-decoration: underline;
}
```
