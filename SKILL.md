---
name: make-com-mastery
description: |
  Build and debug Make.com scenarios with Claude Code or Claude Desktop. Covers blueprint structure, MCP tool safety (context overflow prevention), IML expressions, and working module patterns for HubSpot, Brevo, Discord, and OpenAI. Use when creating automations, debugging "successful" scenarios that produce wrong output, or integrating services through Make.com.
---

# Make.com Mastery

A battle-tested skill for building and debugging Make.com scenarios with AI assistance.

## Why This Skill Exists

Make.com's MCP integration is powerful but has hidden gotchas:
- `scenarios_get` can return **200KB+ blueprints** that overflow context windows
- Scenarios report "success" while modules silently fail behind error handlers
- IML syntax has unintuitive rules (1-based arrays, `emptyarray` requirement)

This skill documents patterns learned the hard way so you don't have to.

## Quick Start

**"Debug my scenario that says success but doesn't work"**
→ Remove Ignore error handlers, check actual module errors

**"Create a webhook that adds contacts to HubSpot and Brevo"**
→ Use upsertAContact (not updateContact), build listIds with `emptyarray + add()`

**"Update my scenario blueprint"**
→ Fetch via API to file, modify with Python, PATCH back

## Direct API vs MCP Decision Matrix

Default to the **Direct API** for blueprint modifications. The MCP is great for discovery but risky for complex updates.

| Task | Approach | Why |
|------|----------|-----|
| **Modify blueprints** | Direct API | MCP can overflow context; API writes to disk |
| **Bulk operations** | Direct API | Data stays out of context window |
| **Debug failed scenarios** | Direct API | Full control over payloads |
| **Fetch blueprints** | Direct API | Save to file, parse with Python |
| **Inspect a scenario** | MCP `scenarios_get` | OK if used **early in conversation** only |
| List scenarios/connections | MCP | Quick, low-risk |
| Get module schemas | MCP `app-module_get` | Use `format=instructions` |
| Check execution status | MCP `executions_list` | Simple query |
| Discover module names | MCP `app-modules_list` | Fastest way |
| Validate module config | MCP `validate_module_configuration` | Pre-flight check |

## Context Overflow Warning

### `scenarios_get` — Use Early or Use API

`scenarios_get` returns the **full blueprint** which can be 200KB+. It works via MCP, but **only early in a conversation** when context window headroom is large.

**Rule of thumb:** 
- First few messages? MCP `scenarios_get` is fine.
- Long conversation with many tool calls? Fetch via Direct API and save to disk instead.

### MCP Tool Risk Levels

| Tool | Risk | Notes |
|------|------|-------|
| `scenarios_list`, `connections_list`, `executions_list` | ✅ LOW | Always safe |
| `app-modules_list`, `scenarios_interface` | ✅ LOW | Always safe |
| `app-module_get` (format=instructions) | ✅ LOW | Always safe |
| `validate_module_configuration` | ✅ LOW | Pre-flight check |
| `scenarios_get` | ⚠️ CONDITIONAL | Safe early; use API later |
| `executions_get_detail` | ⚠️ HIGH | Prefer `executions_list` |
| `app-module_get` (format=json) | ⚠️ MEDIUM | Use format=instructions |

## Blueprint Modification Workflow

```
1. Fetch blueprint     → Direct API: curl ... -o blueprint.json (SAVE TO FILE)
                         OR early in conversation: MCP scenarios_get
2. Parse with Python   → Modify in memory/disk (NOT in context)
3. Validate            → MCP validate_module_configuration
4. Deactivate          → MCP scenarios_deactivate (or API /stop)
5. Update              → MCP scenarios_update (or API PATCH)
6. Activate            → MCP scenarios_activate (or API /start)
7. Test                → Trigger webhook / run scenario
8. Check executions    → MCP executions_list
```

## Blueprint Structure Essentials

```json
{
  "flow": [
    {
      "id": 1,
      "mapper": {},
      "module": "app:ModuleName",
      "version": 1,
      "metadata": {"designer": {"x": 0, "y": 0}},
      "parameters": {"__IMTCONN__": YOUR_CONNECTION_ID}
    }
  ],
  "name": "Scenario Name",
  "metadata": {
    "instant": false, "version": 1,
    "designer": {"orphans": []},
    "scenario": {"dlq": false, "dataloss": false, "maxErrors": 3, "autoCommit": true, "sequential": false}
  }
}
```

**Key rules:**
- **parameters**: Static config (connection IDs). No IML expressions allowed.
- **mapper**: Dynamic values. IML expressions go here.
- **Scenario inputs**: Use `{{var.input.fieldname}}` — never `{{1.fieldname}}`.
- **Interface spec**: Only `name` and `type` properties allowed.

## Debugging "Successful" Scenarios

### The Golden Rule: Ignore Handlers Mask Errors

A scenario can show `status: 1` (SUCCESS) while modules silently fail. Error handlers like `builtin:Ignore` hide actual problems.

### Debugging Workflow

```
1. Check execution status     → executions_list(scenarioId)
2. If status=1 but wrong output → Error handlers are hiding failures
3. Remove Ignore handlers      → Update blueprint with onerror:[] removed
4. Trigger test data           → Run the scenario
5. Check execution again       → Now status=3 with actual error
6. Read the error message      → [405], [400], validation errors, etc.
7. Fix root cause              → See Common Error Patterns
8. Re-add Ignore handlers      → Restore error handling once fixed
```

### Common Error Patterns

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `[405]` on HubSpot update | `updateContact` is finicky | Use `upsertAContact` instead |
| `[400] Property values not valid` | Enum value mismatch | Check `contains()` transforms |
| JSON blob in text field | Wrong array access syntax | Use `[1]` indexing, not `first().nested` |
| `listIds/0 should be type number` | `split()` for numeric array | Use `add(emptyarray; N)` pattern |
| `'6' is not a valid array` | `add()` without `emptyarray` | Start with `add(emptyarray; 6)` |
| `BlueprintValidationError` | Wrong input variable syntax | Use `{{var.input.x}}` |

## Common Pitfalls (Quick Reference)

1. **Array indexing is 1-based**: `[1]` for first element, not `[0]`
2. **Nested access**: Use `{{6.choices[1].message.content}}` not `first().nested`
3. **Ignore handlers hide errors**: Remove them to debug
4. **updateContact is unreliable**: Use `upsertAContact`
5. **Module names are case-sensitive**: Verify with `app-modules_list`
6. **No IML in parameters**: Never use `{{...}}` in parameters block
7. **Scenario input syntax**: `{{var.input.fieldname}}`
8. **Interface spec**: Only `name` and `type` properties
9. **Numeric arrays need emptyarray**: `add(emptyarray; N)` pattern
10. **split() returns strings**: Use `emptyarray + add()` for numbers

## Reference Files

Load these as needed based on the task:

- **[references/api-reference.md](references/api-reference.md)**: Direct API commands, auth, Python patterns
- **[references/iml-reference.md](references/iml-reference.md)**: IML functions, array building, indexing
- **[references/module-patterns.md](references/module-patterns.md)**: Working configs for Brevo, HubSpot, OpenAI, Discord

## Examples

### Example 1: Webhook → HubSpot + Brevo

```
User: "Create a webhook that captures form data, adds to HubSpot, and subscribes to Brevo list 6"

Claude: I'll create this scenario. Key patterns:
- Webhook trigger (builtin:BasicWebhook)
- HubSpot upsertAContact (not updateContact)
- Brevo CreateContact with listIds: {{add(emptyarray; 6)}}
```

### Example 2: Debug Silent Failure

```
User: "My scenario shows success but the HubSpot contact isn't updating"

Claude: This is usually an Ignore handler masking errors. Let me:
1. Fetch the blueprint via API
2. Find modules with onerror: [{"handler": "builtin:Ignore"}]
3. Temporarily remove them
4. Re-run to see the actual error
```

### Example 3: AI Research Bot

```
User: "Build a scenario that takes a company name and enriches the contact with research"

Claude: I'll use:
- Scenario inputs (interface) for company name
- OpenAI/Perplexity module for research
- Access response with {{3.choices[1].message.content}} (1-based!)
- HubSpot upsertAContact to save findings
```

## License

MIT — Use freely, attribution appreciated.
