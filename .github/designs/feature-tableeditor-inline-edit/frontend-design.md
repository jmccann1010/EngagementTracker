# Frontend Design: TableEditor Inline Edit

## Overview

Transform the Table Editor data grid from modal-based editing to inline cell editing. Users will click into any editable cell to edit its value, and changes will be auto-saved via AJAX when the cell loses focus.

---

## Architecture Decisions

### 1. Click-to-Edit Pattern

**Decision:** Convert table cells to editable inputs on click, with auto-save on blur.

**Rationale:**
- More intuitive UX than modal dialogs for single-cell edits
- Reduces clicks: one click to edit vs. two clicks (edit button + modal)
- Familiar pattern from spreadsheet applications (Excel, Google Sheets)
- Maintains context — user sees full row while editing

**Implementation Strategy:**
- Add `data-*` attributes to each `<td>` to store metadata (rowId, columnName, dataType, isIdentity, originalValue)
- Attach click handler to editable cells
- Replace cell content with appropriate input element based on data type
- Save on blur event with AJAX POST

---

### 2. Data Type-Specific Inputs

**Decision:** Render different input types based on column data type.

**Rationale:**
- Improves validation — browser provides built-in validation for `type="number"`, `type="datetime-local"`, etc.
- Better UX — date pickers for dates, number steppers for numbers
- Reduces errors — users can't enter text in numeric fields

**Input Type Mapping:**
```javascript
switch (dataType) {
	case 'System.Int32':
	case 'System.Decimal':
	case 'System.Double':
		return '<input type="number" step="any" />';
	case 'System.DateTime':
		return '<input type="datetime-local" />';
	case 'System.Boolean':
		return '<input type="checkbox" />';
	case 'System.Guid':
	case 'System.String':
	default:
		return '<input type="text" />';
}
```

---

### 3. Auto-Save on Blur

**Decision:** Save changes automatically when the input loses focus (blur event).

**Rationale:**
- Eliminates need for "Save" button — streamlines UX
- Consistent with other inline-edit implementations (CrossoverFields)
- Prevents data loss — changes are persisted immediately

**Edge Cases:**
- If value hasn't changed, skip AJAX call (optimization)
- If user presses Escape, cancel edit without saving
- If AJAX fails, revert to original value and show error

---

### 4. Visual States

**Decision:** Use CSS classes to indicate cell state (editable, editing, saving, success, error).

**Rationale:**
- Clear feedback helps users understand system state
- CSS classes make visual states maintainable and testable
- Consistent with existing CustomToolbox patterns

**CSS Classes:**
```css
.editable-cell {
	cursor: pointer;
	background-color: #f8f9fa; /* light gray hover */
}

.editing-cell {
	border: 2px solid #007bff; /* blue border */
	background-color: #fff;
}

.saving-cell {
	background-color: #fff3cd; /* yellow */
}

.success-cell {
	background-color: #d4edda; /* green */
}

.error-cell {
	background-color: #f8d7da; /* red */
}
```

---

## Component Boundaries

### View Layer (Implementation Required)
- **File:** `CustomToolbox.Mvc/Views/TableEditor/TableEditor.cshtml`
- **Changes:**
  - Add `data-*` attributes to `<td>` elements (rowId, columnName, dataType, isIdentity, originalValue)
  - Add CSS classes for visual states
  - Add JavaScript module for inline edit behavior
  - Apply to both metadata and properties rendering paths

### Controller Layer (Implementation Required)
- **File:** `CustomToolbox.Mvc/Controllers/TableEditorController.cs`
- **Changes:**
  - Add `UpdateCell` action method
  - Accept: `masterId`, `rowId`, `columnName`, `newValue`
  - Validate inputs and delegate to repository
  - Return JSON: `{ success: bool, message: string, newValue: string }`

### No Changes Required
- Existing modal edit functionality remains for multi-field edits
- "Add Row" button continues to open modal
- DataTables initialization unchanged

---

## Data Flow

```
User clicks editable cell
	↓
JavaScript: Replace cell content with input
	├─ Set input value to originalValue
	├─ Focus input
	└─ Add blur event handler
	↓
User edits value and clicks away (blur)
	↓
JavaScript: Check if value changed
	├─ If unchanged → revert to display mode, no AJAX
	└─ If changed → show saving indicator, make AJAX POST
	↓
AJAX POST /TableEditor/UpdateCell
	├─ masterId: @Model.TableEditorData.MasterId
	├─ rowId: from data-row-id
	├─ columnName: from data-column-name
	└─ newValue: input.value
	↓
Controller: Validate inputs
	↓
Repository: UpdateCellValueAsync(masterId, rowId, columnName, newValue)
	├─ Load master metadata
	├─ Build parameterized UPDATE SQL
	├─ Execute SqlCommand
	└─ Return error message or empty string
	↓
Controller: Return JSON { success, message, newValue }
	↓
JavaScript: Handle response
	├─ Success → show success indicator, update cell display value
	└─ Failure → show error indicator, revert to originalValue
```

---

## Visual Design

### Table Cell Rendering (Metadata Path)

**Before (Current):**
```razor
<td>@row.RowData[i].ToString()</td>
```

**After (With Inline Edit Support):**
```razor
<td class="@(Model.TableEditorData.TableEditorColumnsFromDatabase[i].IsIdentity == true ? "" : "editable-cell")"
	data-row-id="@row.RowId"
	data-column-index="@i"
	data-column-name="@Model.TableEditorData.TableEditorColumnsFromDatabase[i].ColumnName"
	data-data-type="@Model.TableEditorData.TableEditorColumnsFromDatabase[i].DataType"
	data-is-identity="@Model.TableEditorData.TableEditorColumnsFromDatabase[i].IsIdentity"
	data-original-value="@row.RowData[i].ToString()">
	@row.RowData[i].ToString()
</td>
```

**Same pattern applies to properties path with `TableEditorColumnsFromProperties`.**

---

### JavaScript Module Structure

```javascript
var TableEditorInlineEdit = (function() {
	var masterId = null;
	var currentEditingCell = null;

	function init(masterIdParam) {
		masterId = masterIdParam;
		attachClickHandlers();
	}

	function attachClickHandlers() {
		$('#propertiesTable tbody').on('click', 'td.editable-cell', function() {
			makeEditable($(this));
		});
	}

	function makeEditable($cell) {
		if (currentEditingCell) {
			saveCell(currentEditingCell); // Save any existing edit
		}

		var originalValue = $cell.data('original-value');
		var dataType = $cell.data('data-type');
		var isIdentity = $cell.data('is-identity');

		if (isIdentity === true) {
			return; // Identity columns not editable
		}

		var $input = createInputForDataType(dataType, originalValue);
		$cell.addClass('editing-cell').html($input);
		$input.focus();

		$input.on('blur', function() {
			saveCell($cell);
		});

		$input.on('keydown', function(e) {
			if (e.key === 'Escape') {
				cancelEdit($cell, originalValue);
			}
		});

		currentEditingCell = $cell;
	}

	function createInputForDataType(dataType, value) {
		var inputType = 'text';
		var additionalAttrs = '';

		switch (dataType) {
			case 'System.Int32':
			case 'System.Decimal':
			case 'System.Double':
				inputType = 'number';
				additionalAttrs = 'step="any"';
				break;
			case 'System.DateTime':
				inputType = 'datetime-local';
				value = formatDateTimeForInput(value);
				break;
			case 'System.Boolean':
				inputType = 'checkbox';
				additionalAttrs = value === 'True' ? 'checked' : '';
				break;
		}

		return $('<input>')
			.attr('type', inputType)
			.attr(additionalAttrs)
			.val(value)
			.addClass('form-control form-control-sm');
	}

	function saveCell($cell) {
		var $input = $cell.find('input');
		if (!$input.length) return;

		var newValue = $input.is(':checkbox') ? $input.prop('checked').toString() : $input.val();
		var originalValue = $cell.data('original-value');

		if (newValue === originalValue) {
			cancelEdit($cell, originalValue); // No change, just revert
			return;
		}

		$cell.removeClass('editing-cell').addClass('saving-cell')
			.html('<span class="spinner-border spinner-border-sm"></span> Saving...');

		$.ajax({
			url: '/TableEditor/UpdateCell',
			type: 'POST',
			data: {
				masterId: masterId,
				rowId: $cell.data('row-id'),
				columnName: $cell.data('column-name'),
				newValue: newValue,
				__RequestVerificationToken: $('input[name="__RequestVerificationToken"]').val()
			},
			success: function(response) {
				if (response.success) {
					$cell.removeClass('saving-cell').addClass('success-cell')
						.html(response.newValue)
						.data('original-value', response.newValue);

					setTimeout(function() {
						$cell.removeClass('success-cell');
					}, 1000);
				} else {
					showError($cell, response.message, originalValue);
				}
			},
			error: function() {
				showError($cell, 'Network error. Please try again.', originalValue);
			}
		});

		currentEditingCell = null;
	}

	function cancelEdit($cell, originalValue) {
		$cell.removeClass('editing-cell').html(originalValue);
		currentEditingCell = null;
	}

	function showError($cell, message, originalValue) {
		$cell.removeClass('saving-cell').addClass('error-cell')
			.html(originalValue + ' <i class="fa fa-exclamation-triangle" title="' + message + '"></i>');
	}

	function formatDateTimeForInput(dateString) {
		// Convert server datetime string to 'YYYY-MM-DDTHH:mm' format
		var date = new Date(dateString);
		if (isNaN(date.getTime())) return '';

		var year = date.getFullYear();
		var month = String(date.getMonth() + 1).padStart(2, '0');
		var day = String(date.getDate()).padStart(2, '0');
		var hours = String(date.getHours()).padStart(2, '0');
		var minutes = String(date.getMinutes()).padStart(2, '0');

		return `${year}-${month}-${day}T${hours}:${minutes}`;
	}

	return {
		init: init
	};
})();

// Initialize on document ready
$(document).ready(function() {
	var masterId = @Model.TableEditorData.MasterId;
	TableEditorInlineEdit.init(masterId);
});
```

---

## Security Considerations

1. **CSRF Protection:** Include `__RequestVerificationToken` in AJAX POST (already present on page).
2. **Input Validation:** Server-side validation in controller and repository (see backend design).
3. **XSS Protection:** Razor automatically encodes output; use `.text()` not `.html()` when displaying user input.
4. **Authorization:** Existing TableEditor access controls apply — no new permissions needed.

---

## Accessibility Considerations

1. **Keyboard Navigation:** Tab between cells, Enter to edit, Escape to cancel.
2. **Screen Readers:** Use `aria-label` on input elements to describe the column being edited.
3. **Focus Management:** Focus input immediately after cell is clicked; restore focus after save/cancel.
4. **Error Announcements:** Use `aria-live` region to announce save success/failure to screen readers.

---

## Testing Strategy

### Manual Testing (12 scenarios recommended)

**File:** Manual test plan documented in QA validation summary

1. **Core Inline Edit**
   - Click editable cell → cell becomes input
   - Input pre-filled with current value
   - Input focused automatically

2. **Save on Blur**
   - Edit value, click away → AJAX POST sent
   - Success → cell shows new value with success indicator
   - Failure → cell reverts to original value with error indicator

3. **Data Type Handling**
   - Int32 column → number input
   - DateTime column → datetime-local input
   - Boolean column → checkbox
   - String column → text input

4. **Escape to Cancel**
   - Edit value, press Escape → cell reverts without saving
   - No AJAX call made

5. **Identity Columns**
   - Identity columns not editable (no click handler)

6. **Edge Cases**
   - No change → no AJAX call
   - Empty value → handled appropriately per column
   - Invalid data → server validation rejects, error shown

---

## Performance Considerations

1. **AJAX Optimization:** Skip AJAX call if value unchanged.
2. **Event Delegation:** Use `.on()` with delegation for click handlers (supports dynamic content).
3. **Debouncing:** Not needed for blur-triggered saves (single event).

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| User clicks away before edit completes | Save on blur ensures edits are captured before leaving cell |
| Multiple cells edited simultaneously | Track `currentEditingCell`; save previous cell before editing new one |
| Network error during save | Show error indicator; user can retry by editing again |
| Data type mismatch | Server validates data type; client renders appropriate input type |

---

## Open Questions

None. Design is complete and ready for implementation.
