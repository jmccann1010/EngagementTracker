# Backend Design: TableEditor Inline Edit

## Overview

Add backend support for inline cell editing in the Table Editor. This includes a new controller action `UpdateCell` and a repository method `UpdateCellValueAsync` that handle single-cell updates with validation and data type conversion.

---

## Architecture Decisions

### 1. Single-Cell Update Endpoint

**Decision:** Create a dedicated `UpdateCell` endpoint that accepts row ID, column name, and new value.

**Rationale:**
- Simpler than reusing `UpdateRowInfo` (which expects all column values)
- More efficient — updates only one column instead of entire row
- Clear API contract for inline edits
- Easier to validate and test

**Alternative Considered:**
Reuse `UpdateRowInfo` with partial data — Rejected: would require complex logic to determine which fields changed and handle nulls/defaults for unchanged fields.

---

### 2. Parameterized SQL with Type Conversion

**Decision:** Build parameterized UPDATE statements with proper data type conversion based on column metadata.

**Rationale:**
- **Security:** Parameterized queries prevent SQL injection
- **Correctness:** Type conversion ensures data matches column type (e.g., string "123" → int 123)
- **Error Handling:** Clear error messages when conversion fails
- **Consistency:** Same pattern as existing repository methods

**Implementation:**
```csharp
var sqlCommand = $"UPDATE [{master.DatabaseTable}] SET [{columnName}] = @newValue WHERE [{identityColumn}] = @rowId";

using var command = new SqlCommand(sqlCommand, conn);
command.Parameters.AddWithValue("@newValue", ConvertToColumnType(newValue, dataType));
command.Parameters.AddWithValue("@rowId", rowId);
```

---

### 3. Validation Strategy

**Decision:** Validate inputs at both controller and repository layers.

**Rationale:**
- **Controller:** Validate request parameters (masterId, rowId, columnName presence)
- **Repository:** Validate column exists, is not identity, data type conversion succeeds
- **Defense in Depth:** Multiple validation layers catch different error categories
- **Clear Error Messages:** Specific feedback helps frontend display meaningful errors

**Validation Layers:**
1. **Controller:** Parameter binding, null checks, basic range checks
2. **Repository:** Column existence, identity check, type conversion
3. **Database:** Constraints, foreign keys, unique indexes

---

### 4. Error Handling and Logging

**Decision:** Log all errors, return user-friendly messages to frontend.

**Rationale:**
- **Security:** Don't expose internal details (e.g., SQL error messages) to frontend
- **Debugging:** Full error details logged for developers
- **UX:** User-friendly messages help users understand and fix issues

**Error Response Pattern:**
```json
{
	"success": false,
	"message": "Invalid data type for column 'Age'. Expected number."
}
```

---

## Component Boundaries

### Controller Layer (Implementation Required)
- **File:** `CustomToolbox.Mvc/Controllers/TableEditorController.cs`
- **New Method:** `UpdateCell`
  - Decorated with `[HttpPost]`, `[ValidateAntiForgeryToken]`
  - Accepts: `masterId`, `rowId`, `columnName`, `newValue`
  - Validates inputs
  - Delegates to repository
  - Returns JSON response

### Repository Layer (Implementation Required)
- **File:** `TableEditor.Repository/TableEditorRepository.cs`
- **New Method:** `UpdateCellValueAsync`
  - Signature: `Task<(bool success, string message, string newValue)> UpdateCellValueAsync(int masterId, int rowId, string columnName, string newValue)`
  - Loads master metadata
  - Validates column (exists, not identity)
  - Converts data type
  - Executes parameterized UPDATE
  - Returns success/error tuple

### Data Access Layer (No Changes)
- Existing DAL entities and context remain unchanged

---

## Data Flow

```
Frontend AJAX POST /TableEditor/UpdateCell
	├─ masterId: 123
	├─ rowId: 5
	├─ columnName: "FirstName"
	└─ newValue: "John"
	↓
Controller: UpdateCell(int masterId, int rowId, string columnName, string newValue)
	├─ Validate: masterId > 0
	├─ Validate: rowId > 0
	├─ Validate: columnName not null/empty
	└─ Call repository
	↓
Repository: UpdateCellValueAsync(masterId, rowId, columnName, newValue)
	├─ Load master: GetMasterByIdAsync(masterId)
	├─ Load columns: GetTableEditorTableDataAsync(masterId)
	├─ Find column: columns.FirstOrDefault(c => c.ColumnName == columnName)
	├─ Validate: column != null
	├─ Validate: column.IsIdentity == false
	├─ Find identity column: columns.First(c => c.IsIdentity == true)
	├─ Convert data type: ConvertToColumnType(newValue, column.DataType)
	├─ Build SQL: UPDATE [table] SET [column] = @value WHERE [id] = @rowId
	├─ Execute SqlCommand with parameters
	└─ Return (success, message, newValue)
	↓
Controller: Build JSON response
	└─ Return { success, message, newValue }
	↓
Frontend: Handle response
	├─ success → show success indicator, update cell
	└─ !success → show error indicator, revert cell
```

---

## Implementation Details

### Controller Action

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> UpdateCell(int masterId, int rowId, string columnName, string newValue)
{
	try
	{
		// Validate inputs
		if (masterId <= 0)
		{
			return BadRequest(new { success = false, message = "Invalid master ID" });
		}

		if (rowId <= 0)
		{
			return BadRequest(new { success = false, message = "Invalid row ID" });
		}

		if (string.IsNullOrWhiteSpace(columnName))
		{
			return BadRequest(new { success = false, message = "Column name is required" });
		}

		// Delegate to repository
		var (success, message, updatedValue) = await _tableEditorRepository.UpdateCellValueAsync(masterId, rowId, columnName, newValue);

		if (success)
		{
			return Ok(new { success = true, newValue = updatedValue });
		}
		else
		{
			return BadRequest(new { success = false, message = message });
		}
	}
	catch (Exception ex)
	{
		// Log error
		ErrorLoggingStaticClient.LogGenericException(_environment, "Error in UpdateCell", "0",
			SystemTypeIds.CustomToolbox, ErrorTypeIds.GenericError, ex, "", "", 0);

		return StatusCode(500, new { success = false, message = "An unexpected error occurred. Please try again." });
	}
}
```

---

### Repository Method

```csharp
public async Task<(bool success, string message, string newValue)> UpdateCellValueAsync(int masterId, int rowId, string columnName, string newValue)
{
	try
	{
		// Load master metadata
		var master = await _masterRepository.GetMasterByIdAsync(masterId);
		if (master == null)
		{
			return (false, "Master table not found", newValue);
		}

		// Load table columns (metadata or properties)
		var tableEditorTable = await GetTableEditorTableDataAsync(masterId);

		// Determine column source
		var columns = tableEditorTable.TableEditorColumnsFromDatabase?.Any() == true
			? tableEditorTable.TableEditorColumnsFromDatabase.Select(c => new { c.ColumnName, c.DataType, c.IsIdentity }).ToList()
			: tableEditorTable.TableEditorColumnsFromProperties?.Select(c => new { c.ColumnName, c.DataType, c.IsIdentity }).ToList();

		if (columns == null || !columns.Any())
		{
			return (false, "No columns found for this table", newValue);
		}

		// Find target column
		var targetColumn = columns.FirstOrDefault(c => c.ColumnName?.Equals(columnName, StringComparison.OrdinalIgnoreCase) == true);
		if (targetColumn == null)
		{
			return (false, $"Column '{columnName}' not found", newValue);
		}

		// Check if identity column
		if (targetColumn.IsIdentity == true)
		{
			return (false, "Cannot edit identity column", newValue);
		}

		// Find identity column for WHERE clause
		var identityColumn = columns.FirstOrDefault(c => c.IsIdentity == true);
		if (identityColumn == null)
		{
			return (false, "No identity column found for UPDATE WHERE clause", newValue);
		}

		// Convert value to appropriate type
		object convertedValue;
		try
		{
			convertedValue = ConvertToColumnType(newValue, targetColumn.DataType);
		}
		catch (Exception ex)
		{
			return (false, $"Invalid data type for column '{columnName}'. {ex.Message}", newValue);
		}

		// Build parameterized UPDATE statement
		var sqlCommand = $"UPDATE [{master.DatabaseTable}] SET [{targetColumn.ColumnName}] = @newValue WHERE [{identityColumn.ColumnName}] = @rowId";

		// Execute SQL
		await using var conn = new SqlConnection("Server=" + master.DatabaseInstance +
												 ";persist security info=True;" +
												 "user id=SDApps;password=dZifqYO25oJK1aVE0tS8;" +
												 "Integrated Security=false;" +
												 "Encrypt=False;" +
												 "TrustServerCertificate=true;" +
												 "Database=" + master.DatabaseName);
		conn.Open();

		using var command = new SqlCommand(sqlCommand, conn);
		command.Parameters.AddWithValue("@newValue", convertedValue);
		command.Parameters.AddWithValue("@rowId", rowId);

		int rowsAffected = await command.ExecuteNonQueryAsync();

		conn.Close();

		if (rowsAffected == 0)
		{
			return (false, "Row not found or no changes made", newValue);
		}

		return (true, string.Empty, newValue);
	}
	catch (SqlException ex)
	{
		ErrorLoggingStaticClient.LogGenericException(_environment, "Error in UpdateCellValueAsync", "0",
			SystemTypeIds.CustomToolbox, ErrorTypeIds.GenericError, ex, "", "", 0);

		return (false, "Database error. Please try again.", newValue);
	}
	catch (Exception ex)
	{
		ErrorLoggingStaticClient.LogGenericException(_environment, "Error in UpdateCellValueAsync", "0",
			SystemTypeIds.CustomToolbox, ErrorTypeIds.GenericError, ex, "", "", 0);

		return (false, "An unexpected error occurred. Please try again.", newValue);
	}
}

private object ConvertToColumnType(string value, string? dataType)
{
	if (string.IsNullOrWhiteSpace(value))
	{
		return DBNull.Value; // Handle nulls
	}

	return dataType switch
	{
		"System.Int32" => int.Parse(value),
		"System.Decimal" => decimal.Parse(value),
		"System.Double" => double.Parse(value),
		"System.DateTime" => DateTime.Parse(value),
		"System.Boolean" => bool.Parse(value),
		"System.Guid" => Guid.Parse(value),
		"System.String" => value,
		_ => value // Default to string for unknown types
	};
}
```

---

## Validation Rules

### Controller Validation
| Parameter | Rule | Error Response |
|-----------|------|----------------|
| `masterId` | Must be > 0 | `400 Bad Request: "Invalid master ID"` |
| `rowId` | Must be > 0 | `400 Bad Request: "Invalid row ID"` |
| `columnName` | Must not be null/empty | `400 Bad Request: "Column name is required"` |

### Repository Validation
| Check | Rule | Error Response |
|-------|------|----------------|
| Master exists | `GetMasterByIdAsync(masterId) != null` | `"Master table not found"` |
| Column exists | Column found in metadata or properties | `"Column '{columnName}' not found"` |
| Not identity | `column.IsIdentity != true` | `"Cannot edit identity column"` |
| Type conversion | Parse succeeds for target data type | `"Invalid data type for column '{columnName}'. {ex.Message}"` |
| Row exists | `rowsAffected > 0` | `"Row not found or no changes made"` |

---

## Security Considerations

1. **SQL Injection Prevention:** Parameterized queries with `@newValue` and `@rowId` placeholders.
2. **CSRF Protection:** `[ValidateAntiForgeryToken]` attribute on controller action.
3. **Input Validation:** Reject invalid masterId, rowId, columnName at controller layer.
4. **Identity Column Protection:** Prevent editing of identity columns.
5. **Authorization:** Existing TableEditor access controls apply (no new permissions).
6. **Error Message Safety:** Don't expose internal details (e.g., SQL error text) to frontend.

---

## Testing Strategy

### Unit Tests (18 tests recommended)

**File:** `CustomToolbox.Test/Repository/TableEditorRepository_UpdateCellValueAsync_Tests.cs`

1. **Happy Path**
   - Update string column → success
   - Update int column → success
   - Update datetime column → success
   - Update boolean column → success

2. **Validation**
   - Invalid masterId → error message
   - Master not found → error message
   - Column not found → error message
   - Identity column → error message
   - Row not found → error message

3. **Data Type Conversion**
   - Valid int string → parsed correctly
   - Invalid int string → error message
   - Valid datetime string → parsed correctly
   - Invalid datetime string → error message
   - Null/empty value → DBNull.Value

4. **Edge Cases**
   - Empty string for nullable string column → success
   - Whitespace-only value → treated as null
   - Very long string → truncated or error based on column size

---

## Performance Considerations

1. **Single-Row Update:** Only updates one row, one column — minimal database load.
2. **Connection Pooling:** Uses same connection pattern as existing methods — benefits from ADO.NET pooling.
3. **No Transaction Overhead:** Single UPDATE statement doesn't require explicit transaction.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Concurrent edits to same cell | Last write wins (no optimistic concurrency); acceptable for single-user scenarios |
| Invalid data type conversion | Try/catch with user-friendly error message |
| SQL constraint violations | Catch SqlException, log error, return generic message |
| Row deleted between load and save | Check `rowsAffected > 0`, return error if row not found |

---

## Future Enhancements (Out of Scope)

1. **Optimistic Concurrency:** Include timestamp/version column in WHERE clause to detect concurrent edits.
2. **Batch Updates:** Allow updating multiple cells in one request.
3. **Undo/Redo:** Store change history for rollback capability.
4. **Audit Trail:** Log all inline edits with user, timestamp, old value, new value.

---

## Open Questions

None. Design is complete and ready for implementation.
