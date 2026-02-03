# Direct API Reference

## Authentication

**Base URL:** 
- EU: `https://eu1.make.com/api/v2`
- US: `https://us1.make.com/api/v2`

**Auth Header:** `Authorization: Token YOUR_API_KEY`

Get your API key from Make.com → Profile → API.

## Essential Commands

```bash
# List all scenarios
curl -s "https://eu1.make.com/api/v2/scenarios?teamId=YOUR_TEAM_ID" \
  -H "Authorization: Token YOUR_API_KEY"

# Get scenario status (without full blueprint)
curl -s "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID" \
  -H "Authorization: Token YOUR_API_KEY"

# Get full blueprint — SAVE TO FILE, don't put in context
curl -s "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID/blueprint" \
  -H "Authorization: Token YOUR_API_KEY" \
  -o blueprint.json

# Update blueprint
curl -X PATCH "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID" \
  -H "Authorization: Token YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"blueprint": "JSON_STRING", "confirmed": true}'

# Activate scenario
curl -X POST "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID/start" \
  -H "Authorization: Token YOUR_API_KEY"

# Deactivate scenario
curl -X POST "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID/stop" \
  -H "Authorization: Token YOUR_API_KEY"

# List connections
curl -s "https://eu1.make.com/api/v2/connections?teamId=YOUR_TEAM_ID" \
  -H "Authorization: Token YOUR_API_KEY"

# List executions for a scenario
curl -s "https://eu1.make.com/api/v2/scenarios/SCENARIO_ID/executions?limit=10" \
  -H "Authorization: Token YOUR_API_KEY"
```

## Python Pattern for Blueprint Modifications

```python
import requests
import json

API_KEY = "YOUR_API_KEY"
BASE_URL = "https://eu1.make.com/api/v2"  # or us1
HEADERS = {"Authorization": f"Token {API_KEY}"}

def fetch_blueprint(scenario_id):
    """Fetch and save blueprint to file (keeps it out of context)"""
    r = requests.get(f"{BASE_URL}/scenarios/{scenario_id}/blueprint", headers=HEADERS)
    blueprint = r.json().get("response", {}).get("blueprint", {})
    
    # Save original for rollback
    with open(f"blueprint_{scenario_id}_original.json", "w") as f:
        json.dump(blueprint, f, indent=2)
    
    return blueprint

def update_blueprint(scenario_id, blueprint):
    """Update scenario with modified blueprint"""
    payload = {"blueprint": json.dumps(blueprint), "confirmed": True}
    r = requests.patch(f"{BASE_URL}/scenarios/{scenario_id}", headers=HEADERS, json=payload)
    return r.json()

def activate_scenario(scenario_id):
    """Start a scenario"""
    r = requests.post(f"{BASE_URL}/scenarios/{scenario_id}/start", headers=HEADERS)
    return r.json()

def deactivate_scenario(scenario_id):
    """Stop a scenario"""
    r = requests.post(f"{BASE_URL}/scenarios/{scenario_id}/stop", headers=HEADERS)
    return r.json()

# Example: Modify a specific module's mapper
scenario_id = 12345
blueprint = fetch_blueprint(scenario_id)

# Find and modify the module you need
for module in blueprint.get("flow", []):
    if module.get("module") == "sendinblue:CreateContact":
        module["mapper"]["listIds"] = "{{add(add(emptyarray; 6); 18)}}"

# Update the scenario
deactivate_scenario(scenario_id)
result = update_blueprint(scenario_id, blueprint)
activate_scenario(scenario_id)
print(result)
```

## Why Direct API Over MCP for Modifications

1. **MCP blueprints overflow context** — API writes to disk
2. **Bulk operations need loops** — Python + API handles this cleanly
3. **Rollback is easier** — Save original blueprint to file first
4. **Full error responses** — API gives raw HTTP status codes

## Finding Your IDs

| What You Need | Where to Find It |
|---------------|------------------|
| Team ID | URL when viewing scenarios: `make.com/TEAM_ID/scenarios` |
| Scenario ID | URL when editing: `make.com/.../scenarios/SCENARIO_ID` |
| Connection ID | API: `/connections?teamId=X` or MCP: `connections_list` |
| API Key | Profile → API → Create Token |
