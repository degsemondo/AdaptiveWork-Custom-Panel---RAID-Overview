# CSS Field - v1.1 Working Version

```css
/* Professional Planview-style Custom Panel - Compact Design */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  background: #f5f5f5;
  color: #333;
  font-size: 12px;
  line-height: 1.4;
}

.panel-container {
  background: white;
  border-radius: 4px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
  overflow: hidden;
}

.panel-header {
  background: linear-gradient(135deg, #8B3A62 0%, #6f2f52 100%);
  color: white;
  padding: 8px 12px;
  border-bottom: 2px solid #6f2f52;
}

.panel-header h1 {
  font-size: 15px;
  font-weight: 600;
  margin: 0;
}

.tabs {
  display: flex;
  border-bottom: 1px solid #e0e0e0;
  background: #fafafa;
  padding: 0;
}

.tab {
  padding: 6px 12px;
  cursor: pointer;
  border: none;
  background: transparent;
  color: #666;
  font-size: 12px;
  font-weight: 500;
  border-bottom: 2px solid transparent;
  transition: all 0.2s ease;
  position: relative;
  top: 1px;
  margin-bottom: -1px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.tab:hover {
  color: #8B3A62;
  background: #f0f0f0;
}

.tab.active {
  color: #8B3A62;
  border-bottom-color: #8B3A62;
  background: white;
}

.tab-badge {
  display: inline-block;
  background: #e8e8e8;
  color: #666;
  padding: 2px 6px;
  border-radius: 10px;
  font-size: 11px;
  font-weight: 600;
  min-width: 20px;
  text-align: center;
}

.tab.active .tab-badge {
  background: #8B3A62;
  color: white;
}

.tab-content {
  display: none;
  padding: 6px;
}

.tab-content.active {
  display: block;
}

.toolbar {
  display: flex;
  gap: 6px;
  margin-bottom: 6px;
  align-items: center;
  flex-wrap: wrap;
}

.search-input {
  flex: 1;
  min-width: 150px;
  padding: 5px 8px;
  border: 1px solid #ddd;
  border-radius: 3px;
  font-size: 12px;
  transition: border-color 0.2s ease;
}

.search-input:focus {
  outline: none;
  border-color: #8B3A62;
  box-shadow: 0 0 0 2px rgba(139, 58, 98, 0.1);
}

.filter-select {
  padding: 5px 8px;
  border: 1px solid #ddd;
  border-radius: 3px;
  font-size: 12px;
  background: white;
  cursor: pointer;
  transition: border-color 0.2s ease;
}

.filter-select:focus {
  outline: none;
  border-color: #8B3A62;
  box-shadow: 0 0 0 2px rgba(139, 58, 98, 0.1);
}

.btn {
  padding: 5px 10px;
  background: #8B3A62;
  color: white;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  font-size: 12px;
  font-weight: 500;
  transition: background-color 0.2s ease;
  white-space: nowrap;
}

.btn:hover {
  background: #6f2f52;
}

.table-wrapper {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
}

th {
  background: #f5f5f5;
  padding: 4px 6px;
  text-align: left;
  font-weight: 600;
  color: #333;
  border-bottom: 1px solid #ddd;
  border-top: 1px solid #ddd;
  height: 24px;
}

td {
  padding: 4px 6px;
  border-bottom: 1px solid #eee;
  vertical-align: middle;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f9f9f9;
}

.expand-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #8B3A62;
  padding: 2px 4px;
  font-size: 12px;
  transition: transform 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
}

.expand-btn::before {
  content: "▶";
}

.expand-btn.expanded::before {
  content: "▼";
}

.risk-name {
  font-weight: 500;
  color: #333;
}

.badge {
  display: inline-block;
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 11px;
  font-weight: 500;
}

.badge-new {
  background: #e3f2fd;
  color: #1976d2;
}

.badge-closed {
  background: #e8f5e9;
  color: #388e3c;
}

.badge-realised {
  background: #fff3e0;
  color: #f57c00;
}

.editable-cell {
  cursor: text;
  position: relative;
  padding: 2px 4px;
}

.editable-cell:hover {
  background: #f0f0f0;
}

.edit-input,
.edit-select {
  width: 100%;
  padding: 3px 4px;
  border: 1px solid #8B3A62;
  border-radius: 2px;
  font-size: 12px;
  font-family: inherit;
}

.edit-input:focus,
.edit-select:focus {
  outline: none;
  box-shadow: 0 0 0 2px rgba(139, 58, 98, 0.2);
}

.edit-controls {
  display: flex;
  gap: 3px;
  margin-top: 2px;
}

.edit-controls button {
  padding: 3px 6px;
  font-size: 11px;
  border: none;
  border-radius: 2px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.save-btn {
  background: #4caf50;
  color: white;
}

.save-btn:hover {
  background: #45a049;
}

.cancel-btn {
  background: #f44336;
  color: white;
}

.cancel-btn:hover {
  background: #da190b;
}

/* Heat map styles for Risk Rating */
.heat-very-low {
  background: #4caf50;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.heat-low {
  background: #8bc34a;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.heat-medium {
  background: #ff9800;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.heat-high {
  background: #ff7043;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.heat-very-high {
  background: #f4511e;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.heat-critical {
  background: #d32f2f;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  font-weight: 500;
  text-align: center;
  display: inline-block;
}

.action-item-row {
  display: none;
}

.action-item-row.expanded {
  display: table-row;
}

.action-item-content {
  padding: 4px 6px;
  background: #fafafa;
}

.action-item-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 11px;
}

.action-item-table th {
  background: #f0f0f0;
  padding: 3px 4px;
  font-size: 11px;
  height: auto;
  border: 1px solid #e0e0e0;
}

.action-item-table td {
  padding: 3px 4px;
  border: 1px solid #e0e0e0;
}

.no-data {
  text-align: center;
  padding: 12px;
  color: #999;
  font-style: italic;
}
```

## Design Features
- Burgundy color scheme (#8B3A62)
- Compact spacing (12px font, minimal padding)
- Heat map colors for Risk Rating display
- Status badges with color coding
- Smooth transitions and hover effects
- Professional Planview-style interface
