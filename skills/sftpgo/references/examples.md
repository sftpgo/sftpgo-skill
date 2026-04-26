# Examples

Each example shows **two forms**:

- **WebAdmin paste** — the raw string you type into the SFTPGo WebAdmin form field (Subject, Body, Endpoint, template_user, etc.).
- **REST API payload** — the JSON body you `POST` to `/api/v2/events/actions` to create the same action via the API.

When a rule is needed to wire the action to a trigger, the corresponding `POST /api/v2/events/rules` body is shown below the action.

All examples have been validated against SFTPGo v2.7.x.

---

## 1. Upload notification — Email

**Trigger:** filesystem event `upload` / `first-upload`.

### WebAdmin paste

**Subject:**

```gotemplate
[SFTPGo] {{.Name}} uploaded {{pathBase .VirtualPath}}
```

**Body:**

```gotemplate
User:        {{.Name}} <{{.Email}}>
File:        {{.VirtualPath}}
Size:        {{humanizeBytes .FileSize}}
Elapsed:     {{.Elapsed}} ms
Protocol:    {{.Protocol}}
IP:          {{.IP}}
When:        {{.Timestamp.UTC.Format "2006-01-02T15:04:05Z"}}
{{if ne .Status 1}}
Status:      FAILURE ({{stringJoin .Errors ", "}})
{{end}}
```

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "notify-upload",
  "description": "Email notification on file upload",
  "type": 3,
  "options": {
    "email_config": {
      "recipients": ["ops@example.com"],
      "subject": "[SFTPGo] {{.Name}} uploaded {{pathBase .VirtualPath}}",
      "body": "User:        {{.Name}} <{{.Email}}>\nFile:        {{.VirtualPath}}\nSize:        {{humanizeBytes .FileSize}}\nElapsed:     {{.Elapsed}} ms\nProtocol:    {{.Protocol}}\nIP:          {{.IP}}\nWhen:        {{.Timestamp.UTC.Format \"2006-01-02T15:04:05Z\"}}\n{{if ne .Status 1}}\nStatus:      FAILURE ({{stringJoin .Errors \", \"}})\n{{end}}",
      "content_type": 0
    }
  }
}
```

### Rule to wire it up

```json
POST /api/v2/events/rules
{
  "name": "upload-notify",
  "status": 1,
  "trigger": 1,
  "conditions": {
    "fs_events": ["upload", "first-upload"]
  },
  "actions": [
    {"name": "notify-upload", "order": 1}
  ]
}
```

---

## 2. Chat webhooks — Slack, Teams, Discord, Google Chat, Mattermost

Most chat platforms accept an incoming webhook that fires a single HTTP POST with a simple JSON body. All of them use `Content-Type: application/json` and `method: POST`. Pick the **body template** that matches your target; the action wrapper is the same.

### The universal minimum: `{"text": "..."}`

A one-line JSON body is the simplest and most portable form. It works out of the box on:

- **Slack** — classic incoming webhooks.
- **Microsoft Teams** — classic *Incoming Webhook* connectors (the most common Teams integration deployed in the wild).
- **Mattermost** — Slack-compatible webhook endpoints.
- **Google Chat** — incoming webhooks.

It does **not** work on:

- **Discord** — uses `{"content": "..."}` instead of `{"text": "..."}`.
- **Teams Workflow / Power Automate webhooks** — require the Adaptive Card format (see 2d).

The minimal body below is the one to reach for first. If the user asks for something richer (avatars, colored cards, fields), move on to the platform-specific variants afterwards.

```json
{"text": {{printf "User %s uploaded %s (%s) via %s from %s" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol .IP | toJson}}}
```

Pass every dynamic value through `toJson` (or `toJsonUnquoted` inside an already-quoted position). That's the only thing protecting the body from usernames or paths that contain quotes, backslashes, or control characters.

### Per-platform details

#### 2a. Slack — simple text

**Endpoint:** `https://hooks.slack.com/services/T.../B.../xxx`

**Body:**

```json
{"text": {{printf "User %s uploaded %s (%s) via %s from %s" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol .IP | toJson}}}
```

Slack also renders `*bold*`, backticks, and `<@USERID>` mentions inside the text field — add them inside the `printf` format string if you want them.

#### 2b. Slack — Block Kit (richer layout)

**Body:**

```json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": {{printf "*%s* uploaded `%s` (%s) via %s from `%s`" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol .IP | toJson}}
      }
    },
    {
      "type": "context",
      "elements": [
        {"type": "mrkdwn", "text": {{printf "Event UID: %s — %s" .UID (.Timestamp.UTC.Format "2006-01-02T15:04:05Z") | toJson}}}
      ]
    }
  ]
}
```

#### 2c. Microsoft Teams — simple text (classic Incoming Webhook)

The classic Teams *Incoming Webhook* connector — the one most customers deployed before the Workflow migration — accepts the same minimal `{"text": "..."}` body as Slack. This is the first thing to try; it covers the vast majority of real-world Teams integrations.

**Endpoint:** `https://<tenant>.webhook.office.com/webhookb2/...`

**Body:**

```json
{"text": {{printf "User %s uploaded %s (%s) via %s from %s" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol .IP | toJson}}}
```

Teams renders the text with Markdown (`**bold**`, backticks, line breaks with `\n` when rendered) and collapses whitespace.

#### 2d. Microsoft Teams — MessageCard (classic webhook, richer layout)

Still the classic connector endpoint, but with a structured card instead of plain text. Use this when the user wants fields, color bars, or a titled card.

**Body:**

```json
{
  "@type": "MessageCard",
  "@context": "https://schema.org/extensions",
  "summary": {{printf "Upload by %s" .Name | toJson}},
  "themeColor": "0076D7",
  "title": "SFTPGo upload notification",
  "sections": [{
    "facts": [
      {"name": "User",    "value": {{toJson .Name}}},
      {"name": "File",    "value": {{toJson .VirtualPath}}},
      {"name": "Size",    "value": {{humanizeBytes .FileSize | toJson}}},
      {"name": "Protocol","value": {{toJson .Protocol}}},
      {"name": "IP",      "value": {{toJson .IP}}},
      {"name": "When",    "value": {{(.Timestamp.UTC.Format "2006-01-02 15:04:05 UTC") | toJson}}}
    ],
    "markdown": true
  }]
}
```

#### 2e. Microsoft Teams — Adaptive Card (Workflow / Power Automate webhook)

The newer Teams webhooks created through Power Automate Workflows **require** the Adaptive Card schema. A plain `{"text": "..."}` or MessageCard body will be rejected. Identify this flavor by the endpoint URL — it goes through `*.logic.azure.com` or the newer Microsoft Power Automate endpoint format.

**Body:**

```json
{
  "type": "message",
  "attachments": [{
    "contentType": "application/vnd.microsoft.card.adaptive",
    "content": {
      "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
      "type": "AdaptiveCard",
      "version": "1.4",
      "body": [
        {"type": "TextBlock", "size": "Medium", "weight": "Bolder", "text": "SFTPGo upload"},
        {"type": "FactSet", "facts": [
          {"title": "User",     "value": {{toJson .Name}}},
          {"title": "File",     "value": {{toJson .VirtualPath}}},
          {"title": "Size",     "value": {{humanizeBytes .FileSize | toJson}}},
          {"title": "Protocol", "value": {{toJson .Protocol}}},
          {"title": "IP",       "value": {{toJson .IP}}},
          {"title": "When",     "value": {{(.Timestamp.UTC.Format "2006-01-02 15:04:05 UTC") | toJson}}}
        ]}
      ]
    }
  }]
}
```

#### 2f. Discord — simple text

Discord uses `content`, not `text`. This is the one place where the universal `{"text": "..."}` body does not apply.

**Endpoint:** `https://discord.com/api/webhooks/<webhook_id>/<token>`

**Body:**

```json
{"content": {{printf "**%s** uploaded `%s` (%s) via %s" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol | toJson}}}
```

#### 2g. Discord — embed (richer card)

**Body:**

```json
{
  "embeds": [{
    "title": "SFTPGo upload",
    "color": 3447003,
    "fields": [
      {"name": "User",     "value": {{toJson .Name}},                    "inline": true},
      {"name": "Size",     "value": {{humanizeBytes .FileSize | toJson}},"inline": true},
      {"name": "Protocol", "value": {{toJson .Protocol}},                "inline": true},
      {"name": "File",     "value": {{toJson .VirtualPath}},             "inline": false}
    ],
    "timestamp": {{(.Timestamp.UTC.Format "2006-01-02T15:04:05Z") | toJson}}
  }]
}
```

#### 2h. Google Chat — simple text

**Endpoint:** `https://chat.googleapis.com/v1/spaces/.../messages?key=...&token=...`

**Body:**

```json
{"text": {{printf "User %s uploaded %s (%s) via %s from %s" .Name .VirtualPath (humanizeBytes .FileSize) .Protocol .IP | toJson}}}
```

Google Chat also accepts card-v2 messages with `cardsV2: [...]` if a structured layout is required — the same JSON-escaping rules apply.

#### 2i. Mattermost

Mattermost's built-in webhook is Slack-compatible — the Slack snippets above (2a, 2b) work unchanged against a Mattermost endpoint of the form `https://<server>/hooks/<id>`. Mattermost additionally honors `username` and `icon_url` to override the bot identity:

```json
{
  "username": "SFTPGo",
  "icon_url": "https://sftpgo.com/favicon.png",
  "text": {{printf "**%s** uploaded `%s` (%s)" .Name .VirtualPath (humanizeBytes .FileSize) | toJson}}
}
```

### REST API payload (works for every chat variant above)

Only the `body` string changes; the rest of the action payload is identical:

```json
POST /api/v2/events/actions
{
  "name": "chat-upload-notify",
  "description": "Chat notification on upload",
  "type": 1,
  "options": {
    "http_config": {
      "endpoint": "<platform-specific webhook URL>",
      "method": "POST",
      "timeout": 20,
      "headers": [
        {"key": "Content-Type", "value": "application/json"}
      ],
      "body": "<one of the bodies above, with \" escaped for JSON>"
    }
  }
}
```

---

## 3. Generic webhook with full object state (provider events)

**Trigger:** provider event `update` on users.

### WebAdmin paste

**Endpoint:** `https://audit.internal/sftpgo/users`

**Body:**

```json
{
  "event": {{toJson .Event}},
  "object_type": {{toJson .ObjectType}},
  "object_name": {{toJson .ObjectName}},
  "initiator": {{if .Initiator.Admin}}{{toJson .Initiator.Admin.Username}}{{else if .Initiator.User}}{{toJson .Initiator.User.Username}}{{else}}"system"{{end}},
  "timestamp": {{toJson (.Timestamp.UTC.Format "2006-01-02T15:04:05Z")}},
  "object": {{.Object.JSON}}
}
```

**Headers:**

- `Content-Type: application/json`
- `X-SFTPGo-Event-Id: {{.UID}}`

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "audit-provider",
  "description": "Push all user provider events to the audit service",
  "type": 1,
  "options": {
    "http_config": {
      "endpoint": "https://audit.internal/sftpgo/users",
      "method": "POST",
      "timeout": 30,
      "headers": [
        {"key": "Content-Type", "value": "application/json"},
        {"key": "X-SFTPGo-Event-Id", "value": "{{.UID}}"}
      ],
      "body": "{\n  \"event\": {{toJson .Event}},\n  \"object_type\": {{toJson .ObjectType}},\n  \"object_name\": {{toJson .ObjectName}},\n  \"initiator\": {{if .Initiator.Admin}}{{toJson .Initiator.Admin.Username}}{{else if .Initiator.User}}{{toJson .Initiator.User.Username}}{{else}}\"system\"{{end}},\n  \"timestamp\": {{toJson (.Timestamp.UTC.Format \"2006-01-02T15:04:05Z\")}},\n  \"object\": {{.Object.JSON}}\n}"
    }
  }
}
```

### Rule

```json
POST /api/v2/events/rules
{
  "name": "audit-users",
  "status": 1,
  "trigger": 2,
  "conditions": {
    "provider_events": ["add", "update", "delete"],
    "options": {
      "provider_objects": ["user"]
    }
  },
  "actions": [
    {"name": "audit-provider", "order": 1}
  ]
}
```

---

## 4. OIDC JIT user provisioning — group-based role mapping

**Trigger:** IDP login event (`trigger = 7`), scope `user`.

Scenario: users whose IdP role claim equals `app.power-user` must be provisioned into the SFTPGo group `power-users`; everyone else goes into `standard-users`.

**Prerequisite:** add the IdP claim that carries the role (e.g. `sftpgo_role`) to the OIDC binding's `custom_fields` array — that's how it becomes available as `{{index .IDPFields "sftpgo_role"}}` inside the template. The binding's separate `role_field` setting drives the JIT user/admin role internally but is **not** exposed on the EventParams (`.Role` is empty for IDP-login events). If you also want the email available, add `email` to `custom_fields`.

### WebAdmin paste — `template_user`

```json
{
  "username": {{toJson .Name}},
  "email": {{if index .IDPFields "email"}}{{toJson (index .IDPFields "email")}}{{else}}""{{end}},
  "status": 1,
  "permissions": {"/":["*"]},
  "groups": [
    {{if eq (index .IDPFields "sftpgo_role") "app.power-user"}}
    {"type": 1, "name": "power-users"}
    {{else}}
    {"type": 1, "name": "standard-users"}
    {{end}}
  ]
}
```

(`type: 1` in the group mapping means "primary group". The OIDC binding's `custom_fields` must include both `email` and `sftpgo_role` for the placeholders above to resolve.)

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "idp-provision",
  "description": "JIT provision user on first IDP login",
  "type": 13,
  "options": {
    "idp_config": {
      "mode": 1,
      "template_user": "{\n  \"username\": {{toJson .Name}},\n  \"email\": {{if index .IDPFields \"email\"}}{{toJson (index .IDPFields \"email\")}}{{else}}\"\"{{end}},\n  \"status\": 1,\n  \"permissions\": {\"/\":[\"*\"]},\n  \"groups\": [\n    {{if eq (index .IDPFields \"sftpgo_role\") \"app.power-user\"}}\n    {\"type\": 1, \"name\": \"power-users\"}\n    {{else}}\n    {\"type\": 1, \"name\": \"standard-users\"}\n    {{end}}\n  ]\n}"
    }
  }
}
```

### Rule

```json
POST /api/v2/events/rules
{
  "name": "idp-jit",
  "status": 1,
  "trigger": 7,
  "conditions": {
    "idp_login_event": 1
  },
  "actions": [
    {"name": "idp-provision", "order": 1}
  ]
}
```

### Variant — multiple roles, with fallback

If the role claim is a JSON array (a typical group-membership claim like `groups`), add the array claim to `custom_fields` and use `slicesContains`:

```gotemplate
{
  "username": {{toJson .Name}},
  "email": {{if index .IDPFields "email"}}{{toJson (index .IDPFields "email")}}{{else}}""{{end}},
  "status": 1,
  "permissions": {"/":["*"]},
  "groups": [
    {{- $roles := index .IDPFields "groups" }}
    {{- if slicesContains $roles "app.admin" }}
    {"type": 1, "name": "admins"}
    {{- else if slicesContains $roles "app.power-user" }}
    {"type": 1, "name": "power-users"}
    {{- else }}
    {"type": 1, "name": "standard-users"}
    {{- end }}
  ]
}
```

`slicesContains` accepts any slice, so the same shape works whether the claim is `[]string` or `[]any`.

---

## 5. Scheduled retention report via email

**Trigger:** schedule (cron). Chain: Retention Check → Email.

### Action 1 — Retention Check

```json
POST /api/v2/events/actions
{
  "name": "retention-home-90d",
  "type": 8,
  "options": {
    "retention_config": {
      "folders": [
        {"path": "/", "retention": 2160, "delete_empty_dirs": false}
      ]
    }
  }
}
```

### Action 2 — Email the report

**Subject (WebAdmin):**

```gotemplate
[SFTPGo] Retention report {{.Timestamp.UTC.Format "2006-01-02"}}
```

**Body (WebAdmin):**

```gotemplate
Retention check completed on {{.Timestamp.UTC.Format "2006-01-02T15:04:05Z"}}.

{{range .RetentionChecks -}}
User: {{.Username}} (email: {{stringJoin .Email ","}})
Action: {{.ActionName}} (type={{.Type}})
{{range .Results -}}
  path={{.Path}} retention={{.Retention}}h deleted_files={{.DeletedFiles}} deleted_size={{humanizeBytes .DeletedSize}}{{if .Error}} ERROR={{.Error}}{{end}}
{{end -}}
{{end -}}
```

```json
POST /api/v2/events/actions
{
  "name": "retention-email",
  "type": 3,
  "options": {
    "email_config": {
      "recipients": ["ops@example.com"],
      "subject": "[SFTPGo] Retention report {{.Timestamp.UTC.Format \"2006-01-02\"}}",
      "body": "Retention check completed on {{.Timestamp.UTC.Format \"2006-01-02T15:04:05Z\"}}.\n\n{{range .RetentionChecks -}}\nUser: {{.Username}} (email: {{stringJoin .Email \",\"}})\nAction: {{.ActionName}} (type={{.Type}})\n{{range .Results -}}\n  path={{.Path}} retention={{.Retention}}h deleted_files={{.DeletedFiles}} deleted_size={{humanizeBytes .DeletedSize}}{{if .Error}} ERROR={{.Error}}{{end}}\n{{end -}}\n{{end -}}"
    }
  }
}
```

### Rule

```json
POST /api/v2/events/rules
{
  "name": "nightly-retention",
  "status": 1,
  "trigger": 3,
  "conditions": {
    "schedules": [
      {"hour": "2", "minute": "0", "day_of_week": "*", "day_of_month": "*", "month": "*"}
    ]
  },
  "actions": [
    {"name": "retention-home-90d", "order": 1},
    {"name": "retention-email",    "order": 2}
  ]
}
```

---

## 6. ICAP scan on upload

**Trigger:** filesystem event `upload`.

### WebAdmin paste — ICAP `paths`

```json
{{.VirtualPath}}
```

(A single line — ICAP `paths` is an array; each array entry is an independent template.)

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "icap-scan",
  "type": 17,
  "options": {
    "icap_config": {
      "endpoint": "icap://icap.example.com:1344/virus_scan",
      "method": "REQMOD",
      "timeout": 30,
      "paths": ["{{.VirtualPath}}"],
      "block_action": 2,
      "adapt_action": 1,
      "failure_policy": 1
    }
  }
}
```

Values for the policy fields:

- `block_action` — what to do with an infected file: `1` ignore, `2` delete, `3` quarantine (requires `quarantine_folder` + `quarantine_path`).
- `adapt_action` — what to do when the ICAP server returns a modified file: `1` ignore, `2` delete, `3` quarantine, `4` overwrite.
- `failure_policy` — what to do when the scan itself fails (server unreachable, etc.): `1` ignore, `2` delete, `3` quarantine.

Paired with a filesystem rule:

```json
POST /api/v2/events/rules
{
  "name": "icap-on-upload",
  "status": 1,
  "trigger": 1,
  "conditions": {
    "fs_events": ["upload"]
  },
  "actions": [
    {"name": "icap-scan", "order": 1, "relation_options": {"execute_sync": true, "stop_on_failure": true}}
  ]
}
```

---

## 7. Command hook on upload

**Trigger:** filesystem event `upload`.

### WebAdmin paste — command config

- **Command:** `/usr/local/bin/log-upload.sh`
- **Timeout:** `10` (seconds; valid range 1–120)
- **Args:**
  - `--user={{.Name}}`
  - `--ip={{.IP}}`
  - `--path={{.VirtualPath}}`
  - `--size={{.FileSize}}`
  - `--ts={{.Timestamp.Unix}}`
- **Env vars:**
  - `SFTPGO_EVENT_ID={{.UID}}`

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "log-upload",
  "type": 2,
  "options": {
    "cmd_config": {
      "cmd": "/usr/local/bin/log-upload.sh",
      "timeout": 10,
      "args": [
        "--user={{.Name}}",
        "--ip={{.IP}}",
        "--path={{.VirtualPath}}",
        "--size={{.FileSize}}",
        "--ts={{.Timestamp.Unix}}"
      ],
      "env_vars": [
        {"key": "SFTPGO_EVENT_ID", "value": "{{.UID}}"}
      ]
    }
  }
}
```

**Pre-requisite:** the absolute path of the command must be listed under `common.event_manager.enabled_commands` in `sftpgo.json`, or via env var `SFTPGO_COMMON__EVENT_MANAGER__ENABLED_COMMANDS=/usr/local/bin/log-upload.sh,...`. Commands not on the allow-list are rejected at action-save time. The command runs as the user under which `sftpgo` itself is running.

**Note on `ssh_cmd` events.** The actual SSH command string (e.g. `scp -t /tmp`, `sha1sum file`) is **not** exposed in the Event Manager template context — only the legacy `actions.execute_on` external hook receives it via the `SFTPGO_ACTION_SSH_CMD` env var. Inside an Event Manager template the only path-related fields populated for `ssh_cmd` are `.VirtualPath` (the path the command operated on) and `.ObjectName` (its basename). If the SSH command itself matters for the integration, prefer the legacy external action hook over the Event Manager command action.

---

## 8. Filesystem — delete directory contents older than N days (via retention + trailing slash)

This combines a retention action with a filesystem delete to clear a specific staging area at the end of the day. The trailing `/` on the delete path empties the directory without removing it.

### Actions

```json
POST /api/v2/events/actions
{
  "name": "clear-staging-contents",
  "type": 9,
  "options": {
    "fs_config": {
      "type": 2,
      "deletes": ["/staging/{{.Name}}/"]
    }
  }
}
```

The rule pairs it with a Schedule trigger (or with a `first-upload`-chained rule if the semantics are "clear staging when user starts a new session").

---

## 9. HTTP multipart — upload a VFS file + metadata JSON part

### REST API payload

```json
POST /api/v2/events/actions
{
  "name": "forward-to-crm",
  "type": 1,
  "options": {
    "http_config": {
      "endpoint": "https://crm.example.com/intake",
      "method": "POST",
      "timeout": 60,
      "headers": [
        {"key": "Authorization", "value": "Bearer {{toJsonUnquoted .UID}}"}
      ],
      "parts": [
        {
          "name": "file",
          "filepath": "{{.VirtualPath}}",
          "headers": [
            {"key": "Content-Type", "value": "application/octet-stream"}
          ]
        },
        {
          "name": "metadata",
          "body": "{\"user\":{{toJson .Name}},\"virtual_path\":{{toJson .VirtualPath}},\"size\":{{.FileSize}},\"timestamp\":{{toJson (.Timestamp.UTC.Format \"2006-01-02T15:04:05Z\")}}}",
          "headers": [
            {"key": "Content-Type", "value": "application/json"}
          ]
        }
      ]
    }
  }
}
```

`filepath` is rendered against the user's VFS — the referenced file is streamed as that multipart part.

---

## 10. IMAP — ingest mailbox attachments into the user's VFS

The IMAP action **reads** unread messages from a mailbox and saves their attachments into SFTPGo storage. Typical use: a mailbox where partners email attachments, and SFTPGo drops the attachments into the user's home or a virtual folder for downstream processing.

```json
POST /api/v2/events/actions
{
  "name": "imap-ingest",
  "type": 16,
  "options": {
    "imap_config": {
      "endpoint": "imaps://imap.example.com:993",
      "auth_type": 0,
      "username": "ingest@example.com",
      "password": {"status": "Plain", "payload": "<your_password>"},
      "mailbox": "TOProcess",
      "path": "/imap-inbox",
      "post_process_action": 1
    }
  }
}
```

Field semantics:

- `endpoint` — `imap://host:143` for plaintext, `imaps://host:993` for TLS. Schema is **required**; host and port follow URL syntax.
- `auth_type` — `0` for username + password Login, `1` for OAuth2 (then populate `oauth2.{provider, tenant, client_id, client_secret, refresh_token}`).
- `mailbox` — IMAP folder to read (default `INBOX`). Only **unread** messages (without `\Seen`) are processed.
- `path` — VFS path where attachments are written, relative to the user picked by the rule's conditions (rule conditions select the matching user; the action runs in that user's VFS context). Rendered as a Go template against the event context.
- `post_process_action` — `0` mark the message `\Seen` after processing, `1` delete it.
- `target_folder` *(optional)* — name of a virtual folder to mount as the destination filesystem; when set, the action runs **once per event** as the system executor (not per matching user).

This action is typically paired with a **scheduled** trigger (every N minutes) rather than a filesystem event.

```json
POST /api/v2/events/rules
{
  "name": "imap-poll",
  "status": 1,
  "trigger": 3,
  "conditions": {
    "schedules": [
      {"hour": "*", "minute": "*/5", "day_of_week": "*", "day_of_month": "*", "month": "*"}
    ]
  },
  "actions": [
    {"name": "imap-ingest", "order": 1}
  ]
}
```

---

## Patterns worth copying

- **Every JSON body** — always pipe dynamic strings through `toJsonUnquoted` (inside `"…"` positions) or emit them via `toJson` (unquoted positions). Never concatenate raw.
- **Every nil-able access** — guard `.Initiator.User` / `.Initiator.Admin` (both can be nil — `.Initiator` is only resolved for provider events), `.Object.User` / `.Object.Admin` with `{{if ...}}`.
- **Every time value** — use `.Timestamp.UTC.Format "layout"` where `layout` is the Go reference time `2006-01-02T15:04:05Z`. Don't try to format an int.
- **Every OIDC template** — the role and email are not auto-populated on `.Role` / `.Email` for IDP-login events. To use them inside the template, add the corresponding claim names to the binding's `custom_fields` and read with `{{index .IDPFields "claim"}}`.
