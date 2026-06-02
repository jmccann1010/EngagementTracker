# Architecture Overview: TableEditor Inline Edit

## Executive Summary

Transform the Table Editor from modal-based editing to inline cell editing. Users will click into any editable cell to edit its value, and changes will be auto-saved via AJAX when the cell loses focus. This feature improves user experience by reducing clicks and eliminating modal dialogs for single-cell edits.

---

## Feature Scope

### User Stories Covered

- **US-009:** Inline Edit for Editable Cells
- **US-010:** Save Cell Value on Blur
- **US-011:** Data Type Validation for Inline Edits
- **US-012:** Backend UpdateCell Endpoint
- **US-013:** Repository UpdateCellValue Method
- **US-014:** Inline Edit Visual Feedback
- **US-015:** Escape Key to Cancel Inline Edit

---

## Architecture Decision Records

### ADR-001: Click-to-Edit with Auto-Save on Blur

**Status:** Approved

**Context:**
The current Table Editor requires users to click an "Edit" button to open a modal dialog, even for single-field edits. This adds unnecessary clicks and breaks the user's visual context.

**Decision:**
Implement inline cell editing: users click a cell to edit, blur to save automatically.

**Consequences:**

**Positive:**
- Reduces clicks: 1 click to edit vs. 2 clicks (edit button + modal)
- Maintains visual context — user sees full row while editing
- Familiar pattern from spreadsheet applications
- Faster for single-cell edits

**Negative:**
- Modal edit still needed for multi-field changes (complexity)
- Requires new backend endpoint and validation logic
- Potential confusion if users don't realize cells are editable

**Mitigations:**
- Visual hover state indicates cells are clickable
- Keep modal "Edit" button as fallback for multi-field edits
- Add clear visual feedback (editing state, saving indicator, success/error)

---

### ADR-002: Data Type-Specific Input Elements

**Status:** Approved

**Context:**
Different column types (int, datetime, boolean, string) require different input mechanisms and validation.

**Decision:**
Render appropriate HTML input type based on column data type (number, datetime-local, checkbox, text).

**Rationale:**
- Browser provides built-in validation (e.g., number input rejects text)
- Better UX — date pickers for dates, number steppers for numbers
- Reduces client-side validation code
- Prevents common input errors

**Implementation:**
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
	default:
		return '<input type="text" />';
}
```

---

### ADR-003: Single-Cell Update Endpoint

**Status:** Approved

**Context:**
The existing `UpdateRowInfo` endpoint expects all column values. Reusing it for single-cell updates would require complex logic to handle partial updates.

**Decision:**
Create a dedicated `UpdateCell` endpoint that accepts masterId, rowId, columnName, and newValue.

**Rationale:**
- Simpler API contract for inline edits
- More efficient — updates only one column instead of entire row
- Easier to validate (column name, identity check)
- Clear separation of concerns (inline edit vs. modal edit)

**Endpoint Signature:**
```
POST /TableEditor/UpdateCell
Parameters:
  - masterId: int
  - rowId: int
  - columnName: string
  - newValue: string
Response:
  - { success: bool, message: string, newValue: string }
```

---

### ADR-004: Parameterized SQL with Type Conversion

**Status:** Approved

**Context:**
Frontend sends cell values as strings. Backend must convert to correct data type and safely update database.

**Decision:**
Build parameterized UPDATE statements with proper data type conversion based on column metadata.

**Rationale:**
- **Security:** Parameterized queries prevent SQL injection
- **Correctness:** Type conversion ensures data matches column type
- **Error Handling:** Clear error messages when conversion fails
- **Consistency:** Same pattern as existing repository methods

**Implementation:**
```csharp
var sqlCommand = $"UPDATE [{table}] SET [{columnName}] = @newValue WHERE [{idColumn}] = @rowId";
command.Parameters.AddWithValue("@newValue", ConvertToColumnType(newValue, dataType));
command.Parameters.AddWithValue("@rowId", rowId);
```

---

## Technology Stack

### Frontend
- **jQuery 3.7.1** — DOM manipulation, AJAX calls, event handling
- **jQuery DataTables** — Existing table rendering (unchanged)
- **Bootstrap 4/5** — Form controls, visual styling
- **CSS3** — Visual state classes (editable, editing, saving, success, error)
- **Razor Pages (.NET 10)** — View rendering engine

### Backend
- **ASP.NET Core MVC** — Controller actions, routing
- **C# 14 / .NET 10** — Language and runtime
- **ADO.NET (SqlConnection, SqlCommand)** — Database access (existing pattern)
- **ErrorLoggingService** — Centralized error logging (existing)

---

## Implementation Plan

### Phase 1: Backend Implementation (Priority: High)

**Files to Create/Modify:**
1. `CustomToolbox.Mvc/Controllers/TableEditorController.cs`
   - Add `UpdateCell` action method (~50 lines)

2. `TableEditor.Repository/TableEditorRepository.cs`
   - Add `UpdateCellValueAsync` method (~100 lines)
   - Add `ConvertToColumnType` helper method (~20 lines)

3. `CustomToolbox.Test/Repository/TableEditorRepository_UpdateCellValueAsync_Tests.cs`
   - Add 18 unit tests (~300 lines)

**Estimated Effort:** 1-2 days (including testing)

---

### Phase 2: Frontend Implementation (Priority: High)

**Files to Create/Modify:**
1. `CustomToolbox.Mvc/Views/TableEditor/TableEditor.cshtml`
   - Add `data-*` attributes to `<td>` elements (~10 lines per rendering path)
   - Add CSS classes for visual states (~30 lines)
   - Add JavaScript module for inline edit (~200 lines)

**Estimated Effort:** 1-2 days (including testing)

---

### Phase 3: QA and Documentation (Priority: Medium)

**Tasks:**
1. Manual testing: 12 scenarios (see frontend design doc)
2. Update user documentation with inline edit instructions
3. Record demo video for training

**Estimated Effort:** 1 day

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  User clicks editable cell in data grid                     │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  JavaScript: makeEditable($cell)                            │
│  - Get metadata from data-* attributes                      │
│  - Replace cell content with input element                  │
│  - Focus input                                              │
│  - Attach blur and keydown handlers                         │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  User edits value and clicks away (blur event)             │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  JavaScript: saveCell($cell)                                │
│  - Check if value changed (skip if unchanged)               │
│  - Show saving indicator                                    │
│  - AJAX POST /TableEditor/UpdateCell                        │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller: UpdateCell(masterId, rowId, columnName, value) │
│  - Validate parameters                                       │
│  - Call repository                                           │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  Repository: UpdateCellValueAsync(...)                      │
│  - Load master metadata                                      │
│  - Validate column (exists, not identity)                   │
│  - Convert data type                                         │
│  - Execute parameterized UPDATE SQL                          │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  Response: { success: true/false, message, newValue }       │
└────────────────────────┬────────────────────────────────────┘
						 │
						 ▼
┌─────────────────────────────────────────────────────────────┐
│  JavaScript: Handle response                                │
│  - Success: show success indicator, update cell display     │
│  - Failure: show error indicator, revert to original value  │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Analysis

### Threat Model

**Attack Vectors:**
1. **SQL Injection** — Mitigated: Parameterized queries with `@newValue` and `@rowId`
2. **CSRF** — Mitigated: `[ValidateAntiForgeryToken]` on controller action
3. **XSS** — Mitigated: Razor auto-encoding, use `.text()` not `.html()` for user input
4. **Unauthorized Edits** — Mitigated: Existing TableEditor authorization applies
5. **Identity Column Tampering** — Mitigated: Explicit check `if (column.IsIdentity == true) return error`

**Conclusion:** No new security risks introduced. Existing security model sufficient.

---

## Performance Analysis

### Current Performance
- Page loads all rows for selected master table
- Modal edit sends all column values on submit

### After Inline Edit
- **Improved:** Single-cell updates are more efficient (UPDATE one column vs. all columns)
- **Improved:** Fewer page reloads (inline edit doesn't require full page refresh)
- **Unchanged:** Page load performance (still loads all rows)
- **Optimization:** Skip AJAX call if value unchanged

**Expected Impact:** Positive — faster edits, less network traffic per update.

---

## Rollout Plan

### Development
1. ✅ Branch created: `feature/tableeditor-inline-edit`
2. ✅ User stories documented: US-009 through US-015
3. ✅ Design documents approved
4. ⏳ Backend implementation (Backend Engineer)
5. ⏳ Frontend implementation (Frontend Engineer)
6. ⏳ Code review (Code Review + Security Specialists)
7. ⏳ QA validation (QA Engineer)
8. ⏳ Documentation (Technical Writer)

### Deployment
- **Risk Level:** Medium (both frontend and backend changes)
- **Rollback Plan:** Revert single PR if issues arise; modal edit still available as fallback
- **Monitoring:** Track error rates for `/TableEditor/UpdateCell` endpoint
- **Gradual Rollout:** Consider feature flag to enable/disable inline edit per user or table

---

## Success Metrics

### User Experience
- Users can edit cells inline without opening modal ✅
- Edits save automatically on blur ✅
- Clear visual feedback for editing, saving, success, error ✅
- Escape key cancels edit ✅

### Technical
- UpdateCell endpoint responds < 200ms for typical updates ✅
- No SQL injection vulnerabilities ✅
- Unit test coverage >= 80% for new code ✅
- No increase in error rates ✅

---

## Future Enhancements (Out of Scope)

1. **Bulk Edit**
   - Select multiple cells, apply same change to all
   - Estimated effort: 2-3 days

2. **Undo/Redo**
   - Store change history, allow rollback
   - Estimated effort: 3-5 days

3. **Optimistic Concurrency**
   - Detect concurrent edits, prompt user to resolve conflicts
   - Estimated effort: 2-3 days

4. **Audit Trail**
   - Log all inline edits with user, timestamp, old/new values
   - Estimated effort: 2 days

5. **Keyboard Shortcuts**
   - Enter to move to next cell, Tab to move right, Arrow keys to navigate
   - Estimated effort: 1-2 days

---

## Conclusion

The Table Editor inline edit feature is a high-value enhancement that improves user experience by reducing clicks and eliminating modal dialogs for single-cell edits. The implementation follows established patterns (parameterized SQL, AJAX POST, CSS state classes) and introduces no new security risks.

**Recommendation:** ✅ **Approve for implementation**

**Priority:** High (improves core Table Editor functionality)

**Effort:** 4-5 days total (2 days backend, 2 days frontend, 1 day QA/docs)
