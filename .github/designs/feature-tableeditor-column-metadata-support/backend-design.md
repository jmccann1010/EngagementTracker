# Backend Design: TableEditor Column Metadata Support

## Overview

Enable the TableEditor repository to load column definitions from the `TableEditorColumn` table when available, falling back to schema introspection when no metadata exists.

---

## Architecture Decisions

### 1. Dual-Source Column Strategy

**Decision:** The repository will check for metadata columns first, then fall back to properties if none exist.

**Rationale:**
- Maintains backward compatibility with existing master tables that have no `TableEditorColumn` entries
- Allows administrators to curate column visibility and configuration per master table
- Avoids breaking existing TableEditor functionality

**Implementation:**
```csharp
public async Task<TableEditorTable> GetTableEditorTableDataAsync(int masterId)
{
	var tblEditorColumns = await _masterRepository.GetActiveTableColumnsAsync(masterId);

	if (tblEditorColumns.Any())
	{
		return await GetTableFromDataAsync(masterId);  // Use metadata
	}
	return await GetTableFromPropertiesAsync(masterId);  // Introspect schema
}
```

---

### 2. GetTableFromDataAsync Implementation

**Current State:** Stub method that returns empty `TableEditorTable`.

**Required Changes:**
1. Load columns from `TableEditorColumn` via `IMasterRepository.GetActiveTableColumnsAsync(masterId)`
2. Populate `TableEditorTable.TableEditorColumnsFromDatabase` with metadata
3. Build dynamic SELECT statement using only metadata columns (in order)
4. Query master table and populate `TableEditorRows` using metadata data types
5. Handle all supported data types: `System.Int32`, `System.String`, `System.DateTime`, `System.Boolean`, `System.Decimal`, `System.Double`, `System.Guid`

**Key Logic:**
```csharp
private async Task<TableEditorTable> GetTableFromDataAsync(int masterId)
{
	// 1. Load metadata columns
	var tblEditorColumns = await _masterRepository.GetActiveTableColumnsAsync(masterId);
	tableEditorTable.TableEditorColumnsFromDatabase = tblEditorColumns;

	// 2. Build SELECT using metadata column names
	var selectStatement = "SELECT " + 
		string.Join(", ", tblEditorColumns.Select(c => c.ColumnName)) +
		" FROM " + master.DatabaseTable;

	// 3. Execute and populate rows using metadata DataType for correct reader calls
	// (Same pattern as GetTableFromPropertiesAsync but driven by metadata)
}
```

---

### 3. CRUD Operation Compatibility

**Challenge:** `CreateInsertStatement` and `CreateUpdateStatement` currently use `TableEditorColumnsFromProperties`.

**Solution:** Refactor CRUD methods to check both column sources:
```csharp
private string CreateInsertStatement(Master master, TableEditorTable tableEditorTable, List<string> changedValues)
{
	var columns = tableEditorTable.TableEditorColumnsFromDatabase?.Any() == true
		? tableEditorTable.TableEditorColumnsFromDatabase
		: tableEditorTable.TableEditorColumnsFromProperties;

	// Build INSERT using unified column source
}
```

**Impact:** Minimal — same SQL generation logic, just flexible column source.

---

### 4. Column Order Preservation

**Requirement:** Columns must render in the order defined in `TableEditorColumn` (by `Id` or insertion order).

**Implementation:**
- `GetActiveTableColumnsAsync` returns `List<TableEditorColumn>` ordered by `Id` or natural DB order
- Repository preserves this order when building SELECT and populating rows

---

### 5. Identity Column Handling

**Current Logic:** Identifies identity columns via `IsIdentity` property.

**Metadata Support:** `TableEditorColumn.IsIdentity` is already present and will be respected.

**No Changes Required:** Existing UPDATE/DELETE logic that finds identity columns will work with metadata.

---

## Data Flow

```
Controller: TableEditorAsync(masterId)
	↓
Repository: GetTableEditorTableDataAsync(masterId)
	↓
	├─ GetActiveTableColumnsAsync(masterId) → Returns List<TableEditorColumn>
	↓
	├─ IF columns.Any() → GetTableFromDataAsync(masterId)
	│   ├─ Populate TableEditorColumnsFromDatabase
	│   ├─ Build SELECT from metadata
	│   └─ Query master table
	│
	└─ ELSE → GetTableFromPropertiesAsync(masterId)  [existing]
		├─ Populate TableEditorColumnsFromProperties
		├─ Introspect schema
		└─ Query master table
```

---

## Component Boundaries

### Repository Layer (Implementation Required)
- **File:** `TableEditor.Repository/TableEditorRepository.cs`
- **Methods:**
  - `GetTableFromDataAsync` — full implementation
  - `CreateInsertStatement` — refactor to use both column sources
  - `CreateUpdateStatement` — refactor to use both column sources

### Interface Layer (No Changes)
- `ITableEditorRepository` — existing interface sufficient
- `IMasterRepository` — `GetActiveTableColumnsAsync` already defined

### DAL Layer (No Changes)
- `TableEditorColumn` entity already exists with all required properties

---

## Security Considerations

1. **SQL Injection:** Column names from `TableEditorColumn` must be validated or parameterized (current implementation uses string concatenation — acceptable for admin-controlled metadata but should be flagged for review).
2. **Authorization:** No new permissions required — existing TableEditor access controls apply.

---

## Testing Strategy

### Unit Tests (16 tests recommended)

**File:** `CustomToolbox.Test/Repository/TableEditorRepository_GetTableFromDataAsync_Tests.cs`

1. **Happy Path**
   - Metadata exists → returns table with metadata columns
   - No metadata → falls back to properties
   - Columns render in metadata order

2. **Data Type Coverage**
   - Int32, String, DateTime, Boolean, Decimal, Double, Guid all handled correctly

3. **Edge Cases**
   - Empty master table (no rows) → returns empty TableEditorRows
   - Master not found → returns empty TableEditorTable
   - Metadata columns reference non-existent table columns → SQL exception logged

4. **CRUD Operations**
   - INSERT uses metadata columns when present
   - UPDATE uses metadata columns when present
   - DELETE works with metadata-driven rows

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Metadata columns reference non-existent table columns | Wrap SQL execution in try/catch; log errors; return empty table |
| Column order mismatch between metadata and changedValues array | Ensure frontend sends values in same order as backend column list |
| Performance degradation with large column sets | No significant impact expected — same query pattern as properties |

---

## Open Questions

None. Design is complete and ready for implementation.
