# Validation Fix: Partial Numeric String Prevention

## Problem Identified

The previous validation implementation had a vulnerability where `parseInt()` would accept partially numeric strings:

```javascript
parseInt("123abc", 10)  // Returns 123 (incorrect!)
parseInt("100.5extra", 10)  // Returns 100 (incorrect!)
parseInt("50 units", 10)  // Returns 50 (incorrect!)
```

This meant environment variables like `WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="123abc"` would be accepted as `123`, which is incorrect behavior.

## Solution Implemented

### 1. String Format Validation Before Parsing

Added regex validation to ensure the entire string is numeric before calling `parseInt()`:

```typescript
if (typeof value === 'string') {
  const trimmed = value.trim();
  
  // Check if the string contains only digits (and optional leading + or -)
  if (!/^[+-]?\d+$/.test(trimmed)) {
    throw new Error(`${source}: "${value}" is not a valid integer (contains non-numeric characters)`);
  }
  
  parsed = parseInt(trimmed, 10);
}
```

**Regex Pattern: `/^[+-]?\d+$/`**
- `^` - Start of string
- `[+-]?` - Optional leading + or - sign
- `\d+` - One or more digits (0-9)
- `$` - End of string

This ensures **the entire string** is validated before parsing.

### 2. Validation Order

```
1. Check string format (regex) ✓ NEW
2. Parse string to number
3. Check if NaN
4. Check if integer (no decimals)
5. Check if positive (≥ 1)
6. Check if safe integer
7. Check if reasonable (≤ 1,000,000)
```

## Test Coverage Added

### Partially Numeric String Tests (4 new tests)

```typescript
✅ "123abc"      → Rejected (contains non-numeric characters)
✅ "100.5extra"  → Rejected (contains non-numeric characters)
✅ "50 units"    → Rejected (contains non-numeric characters)
✅ "0x64"        → Rejected (hex format not allowed)
```

### Whitespace Handling Tests (1 new test)

```typescript
✅ "  100  "     → Accepted as 100
✅ "\t500\n"     → Accepted as 500
```

### Explicit Sign Tests (1 new test)

```typescript
✅ "+100"        → Accepted as 100
✅ "+500"        → Accepted as 500
```

### Comprehensive Duplicate Handling Test (1 new test)

```typescript
✅ Single request with duplicates across all filter types:
   - Input: 5 events (3 unique), 5 orders (3 unique), 
            4 resolvers (3 unique), 5 chains (3 unique)
   - Expected: subscriptionCount = 12 (not 19)
   - Verifies: Each filter array length matches unique count
```

## Examples

### ❌ Now Correctly Rejected

```bash
# Previously accepted as 123, now rejected
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="123abc"
# Error: "123abc" is not a valid integer (contains non-numeric characters)

# Previously accepted as 100, now rejected
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="100.5extra"
# Error: "100.5extra" is not a valid integer (contains non-numeric characters)

# Previously accepted as 50, now rejected
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="50 units"
# Error: "50 units" is not a valid integer (contains non-numeric characters)

# Previously accepted as 0, now rejected
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="0x64"
# Error: "0x64" is not a valid integer (contains non-numeric characters)
```

### ✅ Still Correctly Accepted

```bash
# Clean integer
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="1000"
# → 1000

# With whitespace (trimmed)
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="  500  "
# → 500

# With explicit positive sign
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="+100"
# → 100

# Negative (but then rejected in positive check)
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="-10"
# → Correctly rejected as "must be a positive integer"
```

## Security Implications

### Before Fix
An attacker or misconfigured system could set:
```bash
WEBSOCKET_MAX_SUBSCRIPTIONS_PER_CONNECTION="999999999999abc"
```
And it would be parsed as `999999999999`, potentially causing issues.

### After Fix
The same value is immediately rejected with a clear error message:
```
Error: "999999999999abc" is not a valid integer (contains non-numeric characters)
```

## Changes Summary

### Modified Files
1. **relayer/src/websocket-server.ts**
   - Updated `validateSubscriptionLimit()` function
   - Added string format validation before parsing

2. **relayer/test/websocket-server.test.ts**
   - Added 4 tests for partially numeric strings
   - Added 1 test for whitespace handling
   - Added 1 test for explicit sign handling
   - Added 1 comprehensive test for duplicates in single request
   - Updated 1 existing test for new error message

### Test Count Changes
- **Before**: 36 test cases
- **Added**: 7 new test cases
- **After**: 43 test cases

## Validation Rules (Updated)

```
┌───────────────────────────────────────────────────────────────────┐
│ Rule                        │ Example Pass    │ Example Fail      │
├─────────────────────────────┼─────────────────┼───────────────────┤
│ ✅ String format (NEW)      │ "100", "+500"   │ "123abc", "0x64"  │
│ ✅ Required                 │ 100             │ undefined         │
│ ✅ Valid Number             │ 100, "100"      │ NaN               │
│ ✅ Whole Integer            │ 100             │ 10.5              │
│ ✅ Positive (≥ 1)           │ 1, 100          │ 0, -5             │
│ ✅ Safe Integer             │ 999999          │ 2^53 + 1          │
│ ✅ Reasonable (≤ 1M)        │ 1000            │ 10,000,000        │
└───────────────────────────────────────────────────────────────────┘
```

## Backward Compatibility

✅ **All valid configurations continue to work**
- Pure numeric strings: "100", "1000", "999999"
- Strings with whitespace: "  100  " (trimmed)
- Strings with explicit sign: "+100"

❌ **Invalid configurations now properly rejected**
- Partially numeric: "123abc" (was incorrectly accepted)
- With text: "50 units" (was incorrectly accepted)
- Hex format: "0x64" (was incorrectly accepted)

## Conclusion

This fix ensures that only genuinely numeric string values are accepted, preventing potential configuration errors and security issues from partially numeric strings being silently truncated by `parseInt()`.
