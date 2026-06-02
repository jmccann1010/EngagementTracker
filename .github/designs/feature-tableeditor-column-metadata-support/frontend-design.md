# Frontend Design: TableEditor Column Metadata Support

## Overview

Enable the TableEditor Razor view to render jQuery DataTable columns from `TableEditorColumnsFromDatabase` when present, falling back to `TableEditorColumnsFromProperties` when metadata is absent.

---

## Architecture Decisions

### 1. Dual-Source Column Rendering

**Decision:** The view will check `Model.TableEditorData.TableEditorColumnsFromDatabase` first, then fall back to `TableEditorColumnsFromProperties`.

**Rationale:**
- Maintains backward compatibility with existing master tables that have no metadata
- Provides administrators with a curated column experience when metadata exists
- Avoids breaking existing TableEditor functionality

**Implementation:**
```razor
@if (Model is { TableEditorData.TableEditorColumnsFromDatabase: not null } && Model.TableEditorData.TableEditorColumnsFromDatabase.Any())
{
	@* Render from metadata columns *@
}
else if (Model is { TableEditorData.TableEditorColumnsFromProperties: not null } && Model.TableEditorData.TableEditorColumnsFromProperties.Any())
{
	@* Render from introspected columns (existing) *@
}
```

---

### 2. Table Rendering Logic

**Current State:** View renders columns from `TableEditorColumnsFromProperties` only.

**Required Changes:**
1. Add conditional check for `TableEditorColumnsFromDatabase` before the existing properties check
2. When metadata exists, iterate `TableEditorColumnsFromDatabase` to build `<th>` headers and `<td>` cells
3. Use the same JavaScript DataTables initialization for both column sources
4. Preserve existing sort/search/paging behavior

**Key Logic:**
```razor
@if (Model is { TableEditorData.TableEditorColumnsFromDatabase: not null } && Model.TableEditorData.TableEditorColumnsFromDatabase.Any())
{
	<table id="propertiesTable" class="display">
		<thead>
			<tr>
				@foreach (var col in Model.TableEditorData.TableEditorColumnsFromDatabase)
				{
					<th scope="col">@col.ColumnName</th>
				}
				<th scope="col"></th>  @* Edit button column *@
			</tr>
		</thead>
		<tbody>
			@if (Model.TableEditorData.TableEditorRows != null && Model.TableEditorData.TableEditorRows.Any())
			{
				@foreach (var row in Model.TableEditorData.TableEditorRows)
				{
					<tr>
						@for (var i = 0; i < row.RowData.Count; i++)
						{
							<td>@row.RowData[i].ToString()</td>
						}
						<td>
							<button class="btn btn-primary" onclick="openModal('@row.RowId')" title="click to edit">
								<i class="fa-regular fa-pen-to-square"></i>
							</button>
						</td>
					</tr>
				}
			}
		</tbody>
	</table>
}
else if (Model is { TableEditorData.TableEditorColumnsFromProperties: not null } && Model.TableEditorData.TableEditorColumnsFromProperties.Any())
{
	@* Existing properties-driven table (unchanged) *@
}
```

---

### 3. Modal Edit Form Logic

**Challenge:** The modal edit form currently iterates `TableEditorColumnsFromProperties` to build input fields.

**Solution:** Add conditional logic to use metadata columns when present:
```razor
@{
	var columns = Model.TableEditorData.TableEditorColumnsFromDatabase?.Any() == true
		? Model.TableEditorData.TableEditorColumnsFromDatabase
		: Model.TableEditorData.TableEditorColumnsFromProperties;
}

@foreach (var col in columns)
{
	@* Render input field based on DataType, IsIdentity, etc. *@
}
```

**Impact:** Minimal — same form rendering logic, just flexible column source.

---

### 4. JavaScript Compatibility

**Current Behavior:** The view uses jQuery DataTables with default config:
```javascript
$('#propertiesTable').DataTable({
	paging: true,
	searching: true,
	ordering: true,
	info: true,
	lengthChange: true,
	pageLength: 100
});
```

**Required Changes:** None. The same initialization applies regardless of column source.

**Validation:** DataTables automatically adapts to the rendered `<thead>` structure.

---

### 5. AJAX Endpoint Compatibility

**Current Endpoints:**
- `POST /TableEditor/GetRowInfo` — returns row data by `rowId` and `masterId`
- `POST /TableEditor/UpdateRowInfo` — accepts `rowId`, `masterId`, and `changedValues` array
- `POST /TableEditor/DeleteRowInfo` — accepts `rowId`, `masterId`, and `changedValues` array

**Required Changes:** None. Backend repository already handles both column sources.

**Validation:** The `changedValues` array is built by iterating form inputs in the same order as columns, so metadata-driven forms will send values in metadata order.

---

## Component Boundaries

### View Layer (Implementation Required)
- **File:** `CustomToolbox.Mvc/Views/TableEditor/TableEditor.cshtml`
- **Sections:**
  - Table header rendering — add metadata column check
  - Table row rendering — add metadata column check
  - Modal form rendering — add metadata column check

### Controller Layer (No Changes)
- **File:** `CustomToolbox.Mvc/Controllers/TableEditorController.cs`
- **Methods:** All existing controller actions remain unchanged; they already pass the full `TableEditorTable` model.

### Model Layer (No Changes)
- `TableEditorDto` already exposes `TableEditorData.TableEditorColumnsFromDatabase`
- `TableEditorTable` already has both `TableEditorColumnsFromDatabase` and `TableEditorColumnsFromProperties`

---

## Data Flow

```
User: Selects master table
	↓
Controller: TableEditorAsync(masterId)
	↓
Repository: GetTableEditorTableDataAsync(masterId)
	↓
	├─ IF metadata exists → TableEditorColumnsFromDatabase populated
	└─ ELSE → TableEditorColumnsFromProperties populated
	↓
View: TableEditor.cshtml receives TableEditorDto
	↓
	├─ IF TableEditorColumnsFromDatabase.Any() → Render metadata columns
	└─ ELSE → Render properties columns [existing]
	↓
JavaScript: DataTables initialization (same for both sources)
	↓
User: Edits row → AJAX POST → Repository handles both column sources
```

---

## Visual Design

### Table Header
- **Metadata Mode:** Display `ColumnName` from `TableEditorColumn` in metadata order
- **Properties Mode:** Display `ColumnName` from schema introspection (existing)

### Table Rows
- **Metadata Mode:** Display `RowData[i]` for each metadata column in order
- **Properties Mode:** Display `RowData[i]` for each introspected column in order

### Modal Form
- **Metadata Mode:** Build input fields from `TableEditorColumnsFromDatabase`
- **Properties Mode:** Build input fields from `TableEditorColumnsFromProperties` (existing)

### Input Field Types
- **Identity columns:** `readonly` text input
- **DateTime columns:** `type="datetime-local"` input
- **Username columns:** Pre-filled with `@Model.CurrentUserName`
- **All other columns:** Standard `type="text"` input

---

## Security Considerations

1. **XSS Protection:** Razor automatically encodes `@col.ColumnName` and `@row.RowData[i]` — no additional escaping required.
2. **CSRF Protection:** Existing `@Html.BeginForm()` includes CSRF token — no changes needed.
3. **Authorization:** No new permissions required — existing TableEditor access controls apply.

---

## Testing Strategy

### Manual Testing (12 scenarios recommended)

**File:** Manual test plan documented in QA validation summary

1. **Metadata Mode**
   - Master table with metadata → columns render from `TableEditorColumnsFromDatabase`
   - Column headers match metadata `ColumnName` values
   - Column order matches metadata insertion order
   - Edit modal renders inputs for metadata columns only
   - Add/Update/Delete operations work with metadata columns

2. **Properties Mode**
   - Master table without metadata → columns render from `TableEditorColumnsFromProperties`
   - Behavior identical to existing TableEditor (regression check)

3. **Data Type Handling**
   - Int32 columns render as text inputs
   - DateTime columns render as datetime-local inputs
   - String columns render as text inputs
   - Identity columns render as readonly inputs

4. **Edge Cases**
   - Empty master table (no rows) → table renders with headers but no data rows
   - Master not found → error message displayed

---

## Accessibility Considerations

1. **Semantic HTML:** Use `<th scope="col">` for column headers (already present)
2. **Form Labels:** Each modal input has an associated `<label>` (already present)
3. **Button Titles:** Edit buttons have `title="click to edit"` (already present)
4. **Keyboard Navigation:** DataTables and Bootstrap modals are keyboard-accessible by default

---

## Performance Considerations

1. **Rendering:** No significant performance impact — same loop structure for both column sources
2. **DataTables Initialization:** Same JavaScript initialization for both modes
3. **AJAX Overhead:** No additional AJAX calls introduced; existing endpoints reused

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Metadata columns don't align with actual table schema | Backend SQL exception logged; view displays error message |
| JavaScript DataTables breaks with metadata columns | DataTables auto-adapts to rendered `<thead>` structure |
| Modal form inputs don't match backend expectations | `changedValues` array built in same order as backend column list |
| User confusion between metadata and properties modes | No visual indicator needed — behavior is identical to end user |

---

## Open Questions

None. Design is complete and ready for implementation.
