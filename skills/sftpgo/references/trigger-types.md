# Trigger types — what populates the template context

An event rule declares a `trigger` (integer). The trigger determines which keys of the template context are populated. Using a field that the trigger doesn't populate yields an empty string at render time, not a template error — the #2 source of "silent bugs" after `.IDPFields`.

**Source of truth:** SFTPGo Event Manager runtime behavior (verified against v2.7.x).

## Table

| `trigger` | Name                | Populates                                                                                                                                                                                                                                    | Typical uses                                                                |
| --------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 1         | Filesystem event    | `.Name`, `.ExtName`, `.Event`, `.Status`, `.VirtualPath`, `.FsPath`, `.VirtualTargetPath`, `.FsTargetPath`, `.ObjectName`, `.FileSize`, `.Elapsed`, `.Protocol`, `.IP`, `.Role`, `.Email`, `.Timestamp`, `.UID`, `.Metadata`, `.Errors`, `.Shares` (lazy) | Upload notifications, ICAP scans, webhook on download, compress on upload.  |
| 2         | Provider event      | `.Name`, `.Event` (`add`/`update`/`delete`), `.Status = 1`, `.ObjectName`, `.ObjectType`, `.Object` (lazy), `.Role`, `.Email` (when the object is User/Admin/Share), `.IP`, `.Timestamp`, `.UID`, `.Initiator` (lazy)                          | Webhook on user add, CRM sync, audit trail, notify admin of share creation. |
| 3         | Schedule (cron)     | `.Name` = rule name, `.Event = "Schedule"`, `.Status = 1`, `.Timestamp`, `.UID`. Plus whatever sub-actions populate (e.g. retention action populates `.RetentionChecks`).                                                                    | Nightly retention, scheduled email reports, scheduled backups.              |
| 4         | IP blocked          | `.Event = "IP Blocked"`, `.IP`, `.Status = 1`, `.Timestamp`, `.UID`                                                                                                                                                                          | Alert to SOC, sync to external WAF.                                         |
| 5         | Certificate renewal | `.Name` = domain (or comma-joined domain list), `.Event = "Certificate renewal"`, `.Status` (`1` success / `2` error), `.Errors` (on failure), `.Timestamp`, `.UID`                                                                          | Notify ops on ACME renewal success/failure.                                 |
| 6         | On demand           | `.Name` = rule name, `.Event = "On demand"`, `.Status = 1`, `.Timestamp`, `.UID`. Otherwise empty unless earlier chained actions populate fields.                                                                                            | Manually triggered via `POST /events/rules/{name}/run`.                     |
| 7         | IDP login           | `.Name`, `.Event = "IDP login user"` or `"IDP login admin"`, `.Protocol = "OIDC"`, `.Status = 1`, `.IP`, `.Timestamp`, `.UID`, `.IDPFields`                                                                                                  | JIT user/admin provisioning after OIDC login.                               |

## What "populates" means precisely

The template context is a single map — every key exists in every invocation. Fields not populated for the current trigger are the **zero value of their type**: empty string, zero int/int64, empty map, empty slice, zero `time.Time` (`0001-01-01T00:00:00Z`). A template referencing such a field renders nothing (for strings) or `0` (for numeric), silently.

If you need different behavior per trigger, guard with `{{if .X}}...{{end}}`.

## Per-trigger details

### trigger 1 — Filesystem event

Rule condition filter (`fs_events`) selects among:

- `upload`, `pre-upload`, `first-upload` — `.FsPath` / `.VirtualPath` set; `.FileSize` set after completion (post-upload), `0` for `pre-upload`.
- `download`, `pre-download`, `first-download` — same path fields; `.FileSize` set post-download.
- `delete`, `pre-delete` — path fields set; `.FileSize` is the size being deleted.
- `rename` — both `.VirtualPath` / `.FsPath` (source) and `.VirtualTargetPath` / `.FsTargetPath` (destination) set.
- `mkdir`, `rmdir` — path fields; no size.
- `copy` — source and target paths set.
- `ssh_cmd` — `.VirtualPath` / `.FsPath` are the path the SSH command operated on (e.g. the target of `scp -t /tmp/file`), and `.ObjectName` is its basename. The actual SSH command string (`scp`, `sha1sum`, …) is **not** exposed to the template — only the legacy external `actions.execute_on` hook receives it via the `SFTPGO_ACTION_SSH_CMD` env var.

Sync vs async: only `upload`, `pre-upload`, `pre-download`, `pre-delete` are accepted in a rule whose action has `relation_options.execute_sync = true`. All other fs events run async regardless of the flag.

`.Initiator.User` and `.Initiator.Admin` are **both `nil`** for filesystem events — the EventParams built for fs events does not set an initiator. The acting user is in `.Name` / `.Role` / `.Email` / `.Protocol` / `.IP` directly. Don't try `{{.Initiator.User.Email}}` — use `{{.Email}}`.

### trigger 2 — Provider event

Rule condition filters: `provider_objects` (`user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_rule`, `event_action`) and `provider_events` (`add`, `update`, `delete`).

`.Object` is a lazy wrapper around the object itself (see [`template-context.md`](template-context.md#object-lazy-rendering)). After an `update`, `.Object` reflects the **new** state — a fresh DB read happens at render time. The `delete` case is the exception: the pre-delete object snapshot is rendered without a reload (the row is gone from the DB).

`.Initiator` resolution depends on what triggered the change:

- **User self-edit** (a user changing their own profile through the WebClient or REST) → initiator is the object itself (`.Initiator.User` is set).
- **Share events with a sender** (the share owner doing share lifecycle ops) → the sender (typically the User).
- **Share events without a sender** (e.g. a share restored via `loaddata`) → resolved via `UserExists(share.Username)`.
- **Every other provider event** → resolved via `AdminExists(executor)` (`.Initiator.Admin` is set).
- **System executor / empty executor** (e.g. a `loaddata` call running as the system user) → both accessors return `nil`.

`.Email` is populated automatically when the affected object is a User (the user's email addresses, comma-joined), an Admin (the admin's email), or a Share (the share's contact emails).

### trigger 3 — Schedule (cron)

`.Name` is the rule name — the action runs as a system executor. To target specific users, use the rule's `conditions.options.names` and other filters.

No `.Event`, `.VirtualPath`, `.Status`, etc. — it's a wall-clock trigger. But if the rule chains actions like "1. Data Retention → 2. Email", the Email action will see `.RetentionChecks` populated by action 1. `.Initiator` resolves to no User and no Admin.

### trigger 4 — IP blocked

Fired by the Defender when an IP gets blocked (after the threat score crosses the threshold). The defender pushes a minimal payload onto `EventParams`: `.Event = "IP Blocked"`, `.IP`, `.Status = 1`, plus the timestamp / UID added by the dispatcher. **`.Protocol` and `.Errors` are not populated** — there is no per-attempt protocol context attached to a block decision (multiple failed attempts across protocols can contribute to the score) and the defender does not record a free-text reason.

### trigger 5 — Certificate renewal

Fired by the ACME client after a renewal attempt. `.Name` is the domain (or the comma-joined domain list when multiple are configured), `.Event = "Certificate renewal"`, `.Status` is `1` on success / `2` on error, and `.Errors` carries the failure reason. Pair with an Email or HTTP action to notify ops.

### trigger 6 — On demand

Triggered by `POST /api/v2/events/rules/{name}/run`. `.Name` is the rule name; per-user fan-out happens through chained actions that resolve users (e.g. retention, event report, share expiration), and those are the actions whose split mode then sets `.Name` to the per-user value.

### trigger 7 — IDP login

Fired after a successful OIDC token exchange, before the local account lookup. The typical rule chain is: *IDP login → IDP account check action* (type 13), optionally followed by a notification.

Relevant fields:

- `.Name` — the resolved username from the OIDC `username_field` claim.
- `.IP`, `.Protocol` (= `OIDC`), `.Timestamp`, `.UID`.
- `.IDPFields` — a map of **only** the claims listed in the binding's `custom_fields` array. If a claim is not listed there, it is not in `.IDPFields`, even if it's present in the ID token. Read with `{{index .IDPFields "claim_name"}}`.

Fields that are **not** auto-populated for IDP login:

- `.Role` — the role mapped from the binding's `role_field` setting drives the JIT user/admin shape but is not copied to `.Role`. To use it inside the template, list the same claim name in `custom_fields` and read it from `.IDPFields`.
- `.Email` — same story; `email` must be in `custom_fields`.
- `.ExtName`, `.VirtualPath`, `.FsPath`, `.FileSize`, `.Status`, `.Object`, `.Initiator`, `.Errors` — all empty.

Rule sub-filter: `idp_login_events` is `0` = any, `1` = user only, `2` = admin only.

**If the template refers to a claim, confirm the claim is in `custom_fields`.** Missing `custom_fields` entries are the top cause of "my OIDC template always goes to the default branch".
