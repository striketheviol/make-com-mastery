# IML Expression Reference

IML (Integromat Markup Language) is Make.com's expression language. These are the patterns that trip people up most often.

## Building Numeric Arrays

Many modules (Brevo, HubSpot, etc.) require numeric arrays, not string arrays. The `split()` function returns strings, which will fail validation.

| Function | Input | Output | Notes |
|----------|-------|--------|-------|
| `emptyarray` | - | `[]` | Initialize empty array |
| `add(array; item)` | `[]; 6` | `[6]` | Append single item |
| `split(string; delim)` | `"6,18"; ","` | `["6", "18"]` | ⚠️ Returns STRINGS |

### Correct: Chain emptyarray + add() for numeric arrays

```javascript
add(emptyarray; 6)                      // [6]
add(add(emptyarray; 6); 18)             // [6, 18]
add(add(add(emptyarray; 6); 18); 20)    // [6, 18, 20]
```

### Wrong patterns

```javascript
split("6,18"; ",")    // ["6", "18"] — fails numeric validation
add(6; 18)            // Error: '6' is not a valid array
[6, 18]               // Literal arrays don't work in IML
```

### Conditional array building

```javascript
{{if(condition1; add(add(emptyarray; 6); 18);
  if(condition2; add(add(emptyarray; 6); 20);
    add(emptyarray; 6)))}}
```

## Array Indexing (1-Based!)

Make.com uses **1-based** array indexing, not 0-based like JavaScript/Python.

| Syntax | Result |
|--------|--------|
| `{{6.choices[0]}}` | ❌ UNDEFINED |
| `{{6.choices[1]}}` | ✅ First element |
| `{{6.choices[2]}}` | ✅ Second element |
| `{{first(6.choices)}}` | ✅ First element (helper function) |
| `{{last(6.choices)}}` | ✅ Last element |

## Nested Property Access

**WRONG — Returns entire JSON blob:**
```
{{first(6.choices).message.content}}
```

**RIGHT — Drills into nested properties:**
```
{{6.choices[1].message.content}}
```

This is the #1 gotcha with OpenAI/Perplexity responses. You'll see the entire JSON object dumped into a text field instead of the content.

## Scenario Inputs

When your scenario has an interface (inputs), reference them with:

```
{{var.input.fieldname}}
```

**NOT:**
```
{{1.fieldname}}           // Won't work
{{input.fieldname}}       // Won't work
{{parameters.fieldname}}  // Won't work
```

## IML Function Quick Reference

| Pattern | Use |
|---------|-----|
| `{{6.choices[1].message.content}}` | OpenAI/Perplexity response text |
| `{{3.vid}}` | HubSpot contact ID from upsert |
| `{{var.input.company}}` | Scenario input parameter |
| `{{first(5.fieldsById.question_X)}}` | First value from Tally multi-select |
| `{{last(split(email; "@"))}}` | Domain from email address |
| `{{ifempty(field; "")}}` | Default for empty values |
| `{{if(condition; true_val; false_val)}}` | Conditional |
| `{{contains(text; "search")}}` | Check if text contains substring (returns boolean) |
| `{{lower(text)}}` | Lowercase |
| `{{upper(text)}}` | Uppercase |
| `{{trim(text)}}` | Remove whitespace |
| `{{length(array)}}` | Array/string length |
| `{{now}}` | Current timestamp |
| `{{formatDate(now; "YYYY-MM-DD")}}` | Format date |
| `{{add(add(emptyarray; 6); 18)}}` | Build numeric array [6, 18] |
| `{{parseNumber(string; ".")}}` | Convert string to number |
| `{{toString(number)}}` | Convert number to string |

## Common Validation Errors

| Error Message | Cause | Fix |
|---------------|-------|-----|
| `listIds/0 should be type number` | `split()` returns strings | Use `emptyarray + add()` |
| `'6' is not a valid array` | Missing `emptyarray` | `add(emptyarray; 6)` |
| `Cannot read property of undefined` | 0-based index | Use 1-based: `[1]` not `[0]` |
| `BlueprintValidationError` | IML in parameters block | Move to mapper block |
| `Invalid expression` | Wrong input syntax | Use `{{var.input.x}}` |

## Debugging IML

When an expression doesn't work:

1. **Simplify**: Test with a static value first
2. **Check indexing**: Remember 1-based arrays
3. **Check block**: IML only works in `mapper`, not `parameters`
4. **Check escaping**: Quotes inside strings need escaping
5. **Use Set Variable**: Add a `builtin:SetVariable` module to inspect intermediate values
