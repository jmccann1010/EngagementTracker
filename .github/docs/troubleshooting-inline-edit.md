# Troubleshooting: Table Editor Inline Edit Feature

## Issue: Inline editing not working (cell becomes editable but doesn't save on blur/tab-out)

### Debugging Steps

I've added comprehensive console logging to help diagnose the issue. Follow these steps:

#### 1. **Open Browser Developer Tools**
- Press `F12` in your browser
- Navigate to the **Console** tab
- Clear any existing logs

#### 2. **Navigate to TableEditor Page**
- Go to the TableEditor page with a valid master ID
- Check console for: `"Inline edit initialized. Editable cells found: X"`
- If X = 0, the cells don't have the `editable-cell` class (HTML rendering issue)
- If X > 0, cells are properly marked as editable

#### 3. **Click on an Editable Cell**
Console should show:
```
Cell clicked: {row-id: X, column-name: "YYY", ...}
makeEditable called on cell: [td element]
Cell data - dataType: "varchar", isIdentity: false
Original value: "current value"
Input element created: Yes
```

**If you DON'T see these logs:**
- Event delegation is not working
- Check if DataTables is initialized properly
- Check if jQuery is loaded only once

**If cell is an identity column:**
```
Identity column - editing blocked
```

#### 4. **Edit the Value**
- Type a new value in the input field
- The cell should have a yellow background (`.editing-cell` CSS)

#### 5. **Tab Out or Click Outside the Cell**
Console should show:
```
Blur event triggered on: [input element]
Saving cell after blur...
saveCell called, editingCell: [td element]
Input found: 1
New value: "new value" Original value: "old value"
Saving - rowId: X, columnName: "YYY", masterId: Z, newValue: "new value"
```

**If blur event doesn't trigger:**
- Event delegation selector is wrong
- Input wasn't properly focused
- Another event is preventing blur

#### 6. **AJAX Request**
Console should show:
```
AJAX success: {success: true, newValue: "new value"}
```

**If you see network error:**
```
AJAX error: "error" [xhr response text]
```
- Check Network tab for the POST request to `/TableEditor/UpdateCell`
- Verify status code (200 = success, 400 = bad request, 500 = server error)
- Check request payload and response

#### 7. **Success or Error Feedback**
- **Success:** Cell shows green background briefly (2 seconds)
- **Error:** Cell shows red background with error message (3 seconds)

---

## Common Issues & Solutions

### Issue 1: Blur event never fires
**Symptom:** Cell becomes editable but nothing happens when tabbing out or clicking away

**Solution:** 
- Check if jQuery is loaded twice (causes event handlers to be unbound)
- Verify the event delegation selector matches: `.editing-cell input`
- Ensure the input element is properly created and focused

### Issue 2: AJAX request returns 400 Bad Request
**Symptom:** Console shows "AJAX error: 400"

**Check Network tab for details:**
- Missing anti-forgery token → `$('input[name="__RequestVerificationToken"]').val()` returns undefined
- Invalid parameters → rowId, masterId, or columnName is null/0
- Model validation failed → newValue format doesn't match expected type

**Solution:**
- Ensure `Html.BeginForm()` is wrapping the form properly
- Verify `#hiddenMasterId` exists and has a value
- Check that cell `data-row-id` and `data-column-name` attributes are set

### Issue 3: AJAX request returns 500 Internal Server Error
**Symptom:** Console shows "AJAX error: 500"

**Causes:**
- Backend repository error (SQL exception, connection issue, etc.)
- Type conversion failure in `ConvertToColumnType`
- Database constraint violation

**Solution:**
- Check server logs for exception details
- Verify the column exists in the database
- Ensure data type conversion is correct
- Check for database constraints (foreign keys, uniqueness, etc.)

### Issue 4: Cell becomes editable but input doesn't appear
**Symptom:** Cell background turns yellow but no input field shows

**Check console for:**
- `Input element created: No`

**Causes:**
- `createInputForDataType` returned empty string
- HTML escaping broke the input HTML
- CSS is hiding the input

**Solution:**
- Verify `data-data-type` attribute has a valid value
- Check if `escapeHtml()` is working correctly
- Inspect cell HTML in Elements tab

### Issue 5: Nothing happens when clicking cells
**Symptom:** No console logs, cell doesn't become editable

**Causes:**
- `initializeInlineEdit()` never called
- DataTables not initialized
- jQuery loaded twice (event handlers lost)
- Event delegation not attached

**Solution:**
- Check console for `"Inline edit initialized..."`
- Verify DataTables initialization: `dataTable = $('#propertiesTable').DataTable(...)`
- Confirm jQuery is only loaded once (check Network tab)
- Verify `$('#propertiesTable tbody')` exists

---

## Testing Checklist

### Pre-flight Checks
- [ ] jQuery loaded only once (in `_Layout.cshtml`, NOT in view)
- [ ] DataTables CSS and JS loaded
- [ ] Anti-forgery token present: `$('input[name="__RequestVerificationToken"]')` returns element
- [ ] Master ID exists: `$('#hiddenMasterId').val()` returns valid number

### Functional Tests

#### Test 1: Click editable cell
- [ ] Cell has `editable-cell` class
- [ ] Console: "Cell clicked"
- [ ] Cell background turns yellow
- [ ] Input field appears
- [ ] Input is focused and value selected

#### Test 2: Edit and save on blur
- [ ] Type new value
- [ ] Tab out or click outside
- [ ] Console: "Blur event triggered"
- [ ] Console: "Saving cell after blur..."
- [ ] Cell shows spinner icon
- [ ] Console: "AJAX success"
- [ ] Cell shows green flash
- [ ] New value displayed

#### Test 3: Edit and save on Enter
- [ ] Click cell
- [ ] Type new value
- [ ] Press Enter
- [ ] Same success behavior as blur

#### Test 4: Cancel with Escape
- [ ] Click cell
- [ ] Type new value
- [ ] Press Escape
- [ ] Cell reverts to original value
- [ ] No AJAX request sent

#### Test 5: No change
- [ ] Click cell
- [ ] Don't change value
- [ ] Tab out
- [ ] Cell immediately restored (no AJAX)

#### Test 6: Identity column
- [ ] Click identity column cell
- [ ] Console: "Identity column - editing blocked"
- [ ] Cell does NOT become editable

#### Test 7: Data type inputs
Test each data type:
- [ ] String → text input
- [ ] Int → number input (step=1)
- [ ] Decimal/Money → number input (step=any)
- [ ] Bit/Boolean → checkbox
- [ ] Date → date picker
- [ ] DateTime → datetime-local picker
- [ ] Text/NText → textarea

#### Test 8: Error handling
Simulate errors:
- [ ] Network failure → "Network error" shown, cell restored after 3s
- [ ] Backend validation error → Error message shown, cell restored after 3s
- [ ] Invalid value → Error message from backend shown

---

## Quick Fix Commands

### Clear browser cache and reload
```javascript
// Run in Console
location.reload(true);
```

### Check if jQuery is loaded
```javascript
// Run in Console
console.log('jQuery version:', $.fn.jquery);
```

### Check DataTables instance
```javascript
// Run in Console
console.log('DataTable:', $('#propertiesTable').DataTable());
```

### Check anti-forgery token
```javascript
// Run in Console
console.log('Token:', $('input[name="__RequestVerificationToken"]').val());
```

### Manually test cell click
```javascript
// Run in Console
$('.editable-cell').first().click();
```

### Check event handlers
```javascript
// Run in Console
$._data($('#propertiesTable tbody')[0], 'events');
```

---

## Files Modified with Inline Edit Feature

| File | Changes |
|------|---------|
| `CustomToolbox.Mvc/Views/TableEditor/TableEditor.cshtml` | Added `data-*` attributes, CSS states, JavaScript module |
| `CustomToolbox.Mvc/Controllers/TableEditorController.cs` | Added `UpdateCell` action |
| `TableEditor.Repository/Interfaces/ITableEditorRepository.cs` | Added `UpdateCellValueAsync` method |
| `TableEditor.Repository/TableEditorRepository.cs` | Implemented cell update logic |

---

## Contact

If issues persist after following this guide:
1. Export browser console logs (right-click → Save as...)
2. Export Network tab HAR file (Network tab → right-click → Save all as HAR)
3. Include both files when reporting the issue
