# Make.com Mastery

> A battle-tested Claude skill for building and debugging Make.com scenarios.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude-Code-blueviolet)](https://claude.ai/code)

## Why This Exists

Make.com's MCP integration is powerful but has hidden gotchas that cost hours to debug:

- **Context overflow**: `scenarios_get` returns 200KB+ blueprints that kill your conversation
- **Silent failures**: Scenarios report "success" while modules fail behind Ignore handlers
- **IML quirks**: 1-based arrays, the `emptyarray` requirement, nested access gotchas

This skill documents patterns learned through painful trial and error so you don't have to.

## Installation

### Claude Code (CLI)

```bash
# Add the marketplace
/plugin marketplace add striketheviol/make-com-mastery

# Install the skill
/plugin install make-com-mastery@make-com-mastery
```

### Claude Desktop / Claude.ai

1. Download this repo as a ZIP
2. Go to Claude → Settings → Skills
3. Upload the ZIP or drag the `SKILL.md` file

### Manual

Copy the contents of `SKILL.md` to `~/.claude/skills/make-com-mastery/SKILL.md`

## What's Included

```
make-com-mastery/
├── SKILL.md                    # Main skill file
├── marketplace.json            # For plugin marketplace
├── README.md                   # This file
└── references/
    ├── api-reference.md        # Direct API commands & Python patterns
    ├── iml-reference.md        # IML functions, arrays, indexing
    └── module-patterns.md      # Working configs for HubSpot, Brevo, Discord, OpenAI
```

## Quick Examples

**Debug a "successful" scenario that doesn't work:**
```
"My Make.com scenario shows success but the HubSpot contact isn't updating"
```
→ Claude will identify Ignore handlers masking the real error

**Build a webhook automation:**
```
"Create a webhook that adds form submissions to HubSpot and Brevo list 6"
```
→ Claude will use the correct patterns (upsertAContact, emptyarray for listIds)

**Modify a blueprint safely:**
```
"I need to add a new field mapping to scenario 12345"
```
→ Claude will fetch via API to file, modify, and update without context overflow

## Key Insights

### The Context Overflow Problem

The MCP tool `scenarios_get` returns the full blueprint JSON, which can be 200KB+ for complex scenarios. This floods your context window and degrades Claude's performance.

**Solution:** Use the Direct API and save to file:
```bash
curl -s "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID/blueprint" \
  -H "Authorization: Token API_KEY" \
  -o blueprint.json
```

### The Silent Failure Problem

Make.com scenarios can show `status: 1` (SUCCESS) while modules silently fail. The culprit is usually an Ignore error handler:

```json
"onerror": [{"handler": "builtin:Ignore"}]
```

**Solution:** Temporarily remove Ignore handlers, re-run, see the actual error.

### The Array Problem

Brevo, HubSpot, and other modules require numeric arrays. But `split()` returns strings:

```javascript
// WRONG - returns ["6", "18"] (strings)
split("6,18"; ",")

// RIGHT - returns [6, 18] (numbers)
add(add(emptyarray; 6); 18)
```

## Contributing

Found a pattern that should be documented? PRs welcome!

1. Fork the repo
2. Add your pattern to the appropriate reference file
3. Test it in a real scenario
4. Submit a PR with before/after evidence

## License

MIT — Use freely, attribution appreciated.

---

Built with frustration and eventual triumph by [@StrikeTheViol](https://twitter.com/StrikeTheViol)
