# Module Patterns

Working configurations for common Make.com modules. These patterns are tested and production-ready.

## Module Naming Convention

Module names are **case-sensitive** and inconsistent across apps. Always verify with `app-modules_list`.

| App | Example Modules | Naming Style |
|-----|-----------------|--------------|
| HubSpot | `SearchContacts`, `searchDeals` | Mixed! |
| Discord | `createMessage` | camelCase |
| Perplexity | `createAChatCompletion` | camelCase with 'A' |
| OpenAI | `createAChatCompletion` | camelCase with 'A' |
| Brevo | `CreateContact` | PascalCase |
| Slack | `createAMessage` | camelCase with 'A' |

---

## Brevo (Sendinblue)

### App Versioning

| App Name | Version | Connection Type | Notes |
|----------|---------|-----------------|-------|
| `sendinblue` | 1 | `sendinblue` | Legacy — avoid |
| `sendinblue` | 2 | `sendinblue2` | **Current — use this** |

### CreateContact (Working Pattern)

```json
{
  "id": 4,
  "module": "sendinblue:CreateContact",
  "version": 2,
  "parameters": {"__IMTCONN__": YOUR_BREVO_CONNECTION_ID},
  "mapper": {
    "email": "{{1.email}}",
    "listIds": "{{add(add(emptyarray; 6); 18)}}",
    "attributes": {
      "FIRSTNAME": "{{ifempty(1.first_name; \"\")}}",
      "LASTNAME": "{{ifempty(1.last_name; \"\")}}"
    },
    "updateEnabled": true,
    "emailBlacklisted": false,
    "smsBlacklisted": false
  }
}
```

**Key Parameters:**
- `updateEnabled: true` → Upsert behavior (create or update if exists)
- `listIds` → **Must be numeric array** using `emptyarray + add()` pattern
- `attributes` → Use UPPERCASE attribute names as defined in Brevo

### Multiple Lists

```javascript
// Add to lists 6 and 18
"listIds": "{{add(add(emptyarray; 6); 18)}}"

// Add to lists 6, 18, and 20
"listIds": "{{add(add(add(emptyarray; 6); 18); 20)}}"
```

---

## HubSpot

### ALWAYS Use `upsertAContact` Over `updateContact`

The `updateContact` module often fails with `[405]` errors even when the contact exists. Use `upsertAContact` instead — it creates if missing, updates if exists.

### upsertAContact (Recommended)

```json
{
  "id": 8,
  "module": "hubspotcrm:upsertAContact",
  "version": 2,
  "parameters": {"__IMTCONN__": YOUR_HUBSPOT_CONNECTION_ID},
  "mapper": {
    "email": "{{1.email}}",
    "_method": "compact",
    "parseCustomFields": true,
    "swapPrimaryEmail": false,
    "properties": [
      {"key": "firstname", "value": "{{1.first_name}}"},
      {"key": "lastname", "value": "{{1.last_name}}"},
      {"key": "company", "value": "{{1.company}}"},
      {"key": "custom_property", "value": "{{2.output}}"}
    ]
  }
}
```

**Notes:**
- Use `properties` array for custom properties
- Standard fields can also go in properties array
- Returns `vid` (contact ID) on success: `{{8.vid}}`

### SearchContacts

```json
{
  "id": 2,
  "module": "hubspotcrm:SearchContacts",
  "version": 2,
  "parameters": {"__IMTCONN__": YOUR_HUBSPOT_CONNECTION_ID},
  "mapper": {
    "limit": "10",
    "outputProperties": ["firstname", "lastname", "company", "email"],
    "propertyFilters": [
      {"value": "hot", "property": "lead_status", "filterOperator": "EQ"}
    ]
  }
}
```

### Filter Operators

| Operator | Meaning |
|----------|---------|
| `EQ` | Equals |
| `NEQ` | Not equals |
| `CONTAINS` | Contains substring |
| `GT`, `GTE` | Greater than (or equal) |
| `LT`, `LTE` | Less than (or equal) |
| `HAS_PROPERTY` | Property is set |
| `NOT_HAS_PROPERTY` | Property is not set |

---

## OpenAI / Perplexity

### createAChatCompletion (Works for both)

```json
{
  "id": 3,
  "module": "openai-gpt-4:createAChatCompletion",
  "version": 1,
  "parameters": {"__IMTCONN__": YOUR_OPENAI_CONNECTION_ID},
  "mapper": {
    "model": "gpt-4o-mini",
    "max_tokens": 500,
    "temperature": 0.3,
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "{{var.input.prompt}}"}
    ]
  }
}
```

For Perplexity, change:
- Module: `perplexity-ai:createAChatCompletion`
- Model: `"sonar"` or `"sonar-pro"`

### Accessing the Response

**CRITICAL: 1-based indexing!**

```javascript
// Get the response text
{{3.choices[1].message.content}}

// NOT this (returns undefined):
{{3.choices[0].message.content}}

// NOT this (returns JSON blob):
{{first(3.choices).message.content}}
```

---

## Discord

### createMessage

```json
{
  "id": 5,
  "module": "discord:createMessage",
  "version": 1,
  "parameters": {"__IMTCONN__": YOUR_DISCORD_CONNECTION_ID},
  "mapper": {
    "channelId": "YOUR_CHANNEL_ID",
    "content": "New lead: {{1.email}}"
  }
}
```

### With Embed

```json
{
  "mapper": {
    "channelId": "YOUR_CHANNEL_ID",
    "content": "",
    "embeds": [
      {
        "title": "New Lead",
        "description": "{{1.company}} - {{1.email}}",
        "color": 3066993,
        "fields": [
          {"name": "Name", "value": "{{1.first_name}} {{1.last_name}}", "inline": true},
          {"name": "Source", "value": "{{1.source}}", "inline": true}
        ]
      }
    ]
  }
}
```

---

## Webhooks

### Receiving (builtin:BasicWebhook)

```json
{
  "id": 1,
  "module": "builtin:BasicWebhook",
  "version": 1,
  "parameters": {"hook": YOUR_WEBHOOK_ID},
  "mapper": {}
}
```

### Sending (http:ActionSendData)

```json
{
  "id": 10,
  "module": "http:ActionSendData",
  "version": 3,
  "parameters": {},
  "mapper": {
    "url": "https://your-endpoint.com/webhook",
    "method": "POST",
    "headers": [
      {"name": "Content-Type", "value": "application/json"}
    ],
    "body": "{\"email\": \"{{1.email}}\", \"status\": \"processed\"}"
  }
}
```

---

## Scheduling Options

```json
// Webhook-triggered (instant)
"scheduling": {"type": "immediately"}

// On-demand / manual
"scheduling": {"type": "indefinitely", "interval": 900}

// Daily at 9am
"scheduling": {"type": "daily", "time": "09:00:00", "interval": 1}

// Weekly on Sunday at 6pm
"scheduling": {"type": "weekly", "days": [7], "time": "18:00:00", "interval": 1}
```

Days: 1=Monday, 2=Tuesday, ..., 7=Sunday

---

## Error Handling

### Ignore Handler (Hides Errors!)

```json
{
  "id": 5,
  "module": "discord:createMessage",
  "onerror": [{"handler": "builtin:Ignore"}],
  ...
}
```

⚠️ **Warning:** Ignore handlers make debugging impossible. The scenario reports success while modules silently fail. Remove during debugging.

### Resume Handler

```json
{
  "onerror": [{"handler": "builtin:Resume"}]
}
```

Continues to next module even if this one fails.

### Break Handler

```json
{
  "onerror": [{"handler": "builtin:Break"}]
}
```

Stops execution but marks as incomplete (can retry).
