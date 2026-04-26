# Template context

Every SFTPGo Event Action template is rendered with a single map as the dot-context. The keys below are available in **every** template, regardless of trigger. Some keys are only meaningfully populated for specific triggers — the "Populated for" column says when.

**Source of truth:** SFTPGo Event Manager runtime behavior (verified against v2.7.x).

## Top-level keys

| Key                      | Type                                    | Populated for                | Notes                                                                                                                                                                                                                                                              |
| ------------------------ | --------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `.Name`                  | `string`                                | most                         | Username of the event subject. For filesystem events: the connected user. For provider events: the affected entity's primary key. For IDP login: the local username resolved from the OIDC `username_field` claim. For schedule and on-demand: the rule name. For IP-blocked / certificate-renewal: empty (or the certificate's domain for cert events).                                                                                      |
| `.ExtName`               | `string`                                | fs (subset)                  | External username carried over from the user's stored `ExternalUsername` field — populated for filesystem events when the connected user was JIT-provisioned via OIDC and the external claim differs from the local username. Empty for provider, IDP-login, IP-blocked, certificate, schedule, and on-demand events.                                                                                                                                                     |
| `.Event`                 | `string`                                | all                          | Event name. Filesystem: `upload`, `pre-upload`, `first-upload`, `download`, `pre-download`, `first-download`, `delete`, `pre-delete`, `rename`, `mkdir`, `rmdir`, `copy`, `ssh_cmd`. Provider: `add`, `update`, `delete`. IDP login: `IDP login user` or `IDP login admin`. IP-blocked: `IP Blocked`. Certificate: `Certificate renewal`. Schedule: `Schedule`. On-demand: `On demand`. |
| `.Status`                | `int`                                   | most                         | `1` success, `2` generic error, `3` quota exceeded (filesystem only). Set by every event type that constructs `EventParams` — `1` for schedule / on-demand / IDP login / IP-blocked, `1` or `2` for certificate-renewal, full range for filesystem. For `pre-*` filesystem events: `1`.                                                                                                                                                                          |
| `.VirtualPath`           | `string`                                | fs                           | Virtual path as seen by the end user (e.g. `/home/file.txt`). Cleaned. Provider events on a share carry the share path here too.                                                                                                                                                                                             |
| `.FsPath`                | `string`                                | fs                           | Filesystem path (e.g. `/srv/data/user1/file.txt` for local FS; for cloud backends this is the object key including any prefix).                                                                                                                                    |
| `.VirtualTargetPath`     | `string`                                | fs rename/copy               | Destination virtual path. Empty for other events.                                                                                                                                                                                                                  |
| `.FsTargetPath`          | `string`                                | fs rename/copy               | Destination filesystem path.                                                                                                                                                                                                                                       |
| `.ObjectName`            | `string`                                | fs / provider                | Filesystem: base name of the affected file/dir (`path.Base(.VirtualPath)`). Provider: the primary key of the object (username for users, name for folders/groups, share ID for shares, etc.). Empty for IDP login, IP-blocked, certificate, schedule, on-demand.                                                                                                   |
| `.ObjectType`            | `string`                                | provider                     | One of `user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_rule`, `event_action`. Empty for every other trigger.                                                                                                |
| `.FileSize`              | `int64`                                 | fs upload/download           | Bytes transferred. `0` for `pre-*` and non-fs events.                                                                                                                                                                                                              |
| `.Elapsed`               | `int64`                                 | fs upload/download           | Transfer duration in **milliseconds**.                                                                                                                                                                                                                             |
| `.Protocol`              | `string`                                | fs, IDP login                | `SFTP`, `SCP`, `SSH`, `FTP`, `DAV`, `HTTP`, `HTTPShare`, `OIDC`. Empty for the other triggers (including IP-blocked — the defender does not record the protocol on the params).                                                                                                                                                                                                   |
| `.IP`                    | `string`                                | fs, IDP login, IP-blocked    | Client IP address. Empty for provider / schedule / on-demand / certificate-renewal events.                                                                                                                                                            |
| `.Role`                  | `string`                                | fs, provider                 | The user's role from the data provider. Populated for filesystem events (the connection's user), provider events on a user-typed object, and chained actions that resolve a user. **Not** populated on IDP login: the role mapped from the OIDC `role_field` binding setting drives the JIT user/admin shape but is not copied to `.Role`. To use the role inside an IDP-login template, also list the same claim name in the binding's `custom_fields` and read it from `.IDPFields`.                                             |
| `.Email`                 | `string`                                | fs, provider                 | Subject's email address — comma-joined list when the user has additional emails. For IDP login this is empty; expose the `email` claim through `custom_fields` and read `{{index .IDPFields "email"}}`.                                                                                           |
| `.Timestamp`             | `time.Time`                             | all                          | Event timestamp. **This is a `time.Time`**, so you can call `.UTC`, `.Format`, `.Unix`, `.UnixNano` on it. Example: `{{.Timestamp.UTC.Format "2006-01-02T15:04:05"}}`.                                                                                             |
| `.UID`                   | `string`                                | all                          | Unique event identifier (xid). Useful for log correlation, deduplication, idempotency keys.                                                                                                                                                                        |
| `.Metadata`              | `map[string]string`                     | fs                           | Custom metadata attached to the file (e.g. S3 object metadata).                                                                                                                                                                                                    |
| `.IDPFields`             | `map[string]any`                        | IDP login                    | **Only contains the claims that were explicitly listed in the OIDC binding's `custom_fields` config.** Access with `{{index .IDPFields "claim_name"}}`. See `pitfalls.md`.                                                                                         |
| `.Errors`                | `[]string`                              | fs / certificate / on-failure | Error messages accumulated via `params.AddError`. Populated for filesystem events that failed (also reflected in `.Status != 1`) and for certificate-renewal failures. Empty for provider events, IDP login, IP-blocked, scheduled and on-demand runs unless a chained action records errors.                                                                                                                                                       |
| `.Object`                | *lazy*                                  | provider events              | Wraps the object that triggered the event. Typed accessors only exist for User / Admin / Group / Share — for any other object type (folder, api_key, event_rule, event_action) only `.Object.JSON` is meaningful. See [§Object](#object-lazy-rendering) below.                                                                                               |
| `.Initiator`             | *lazy*                                  | provider events              | Lazy User / Admin of who triggered the change. Always exposed in the template (never nil at the top level), but **`.User` and `.Admin` are both nil** for filesystem events, IP blocks, certificate events, schedules, and on-demand runs — those events run without an "initiator" concept on the EventParams. See [§Initiator](#initiator-lazy-rendering) below.                                                                                                  |
| `.Shares`                | *lazy*                                  | post-fs events               | Lazy resolver for the shares owned by the connecting user. Wired only by the post-fs `ExecuteActionNotification` path; for every other trigger `.Shares.Load` returns an empty slice. Call `{{range .Shares.Load}}...{{end}}`.                                                                                                                                                                                    |
| `.RetentionChecks`       | array of retention check records        | after Data Retention action  | Populated by a Data Retention action earlier in the same rule. See [§RetentionChecks](#retentionchecks).                                                                                                                                                           |
| `.EventReports`          | array of per-user event report records  | after Event Report action    | Populated by an Event Report action earlier in the same rule. See [§EventReports](#eventreports).                                                                                                                                                                  |
| `.ShareExpirationChecks` | array of share expiration check records | after Share Expiration Check | Populated by a Share Expiration Check action earlier in the same rule.                                                                                                                                                                                             |
| `.ShareExpirationResult` | single share expiration result          | after Share Expiration Check | Single-share variant of the above (used in split-event mode).                                                                                                                                                                                                      |

## `.Object` (lazy rendering)

`.Object` wraps the provider object that triggered the event. It is **lazy** — accessing `.JSON` or one of the typed methods triggers a fresh DB read. Methods:

| Accessor        | Returns                                               | Use when                                                                |
| --------------- | ----------------------------------------------------- | ----------------------------------------------------------------------- |
| `.Object.JSON`  | `string` — full JSON of the object (secrets stripped) | You want the entire object as a JSON blob to include in a webhook body. |
| `.Object.User`  | User object or `nil`                                  | The object is a User. Check with `{{if .Object.User}}...{{end}}`.       |
| `.Object.Admin` | Admin object or `nil`                                 | The object is an Admin.                                                 |
| `.Object.Group` | Group object or `nil`                                 | The object is a Group.                                                  |
| `.Object.Share` | Share object or `nil`                                 | The object is a Share.                                                  |

Typed accessors strip confidential data (password hashes, API key plaintexts, TOTP secrets, recovery codes, etc.).

Example:

```gotemplate
{{- with .Object.User -}}
User: {{.Username}} <{{.Email}}> role={{.Role}} status={{.Status}}
Permissions on /: {{stringJoin (index .Permissions "/") ", "}}
{{- end}}
```

## `.Initiator` (lazy rendering)

`.Initiator` resolves to the User or Admin who triggered a **provider event** (user/group/folder/share/admin/api_key/event_rule/event_action add / update / delete). It is wired in the provider-event callback only; for every other trigger (filesystem events, IP blocks, certificate events, schedules, on-demand runs) the `Initiator` value is exposed as an empty wrapper, so `.Initiator.User` and `.Initiator.Admin` are both `nil`.

| Accessor           | Returns                                                       |
| ------------------ | ------------------------------------------------------------- |
| `.Initiator.JSON`  | Full JSON of the resolved entity. Empty `{}` when no initiator. |
| `.Initiator.User`  | User object or `nil`                                          |
| `.Initiator.Admin` | Admin object or `nil`                                         |

Guard patterns — both `.User` and `.Admin` may be nil, so always cover the fall-through:

```gotemplate
{{- if .Initiator.Admin }}
Changed by admin: {{.Initiator.Admin.Username}} ({{.Initiator.Admin.Email}})
{{- else if .Initiator.User }}
Changed by user: {{.Initiator.User.Username}}
{{- else }}
Changed by: system
{{- end }}
```

Initiator resolution rules for provider events (as implemented by the Event Manager):

- User self-edit (a user changing their own profile) → initiator is the object itself.
- Share events with a sender → sender.
- Share events without sender (e.g. restore) → resolved via `UserExists(share.Username)`.
- All other provider events → resolved via `AdminExists(executor)`.
- Empty / system executor → both accessors return `nil`.

## `.Shares` (lazy)

Returns the list of shares owned by the subject user. Call `.Load` to iterate:

```gotemplate
{{- range $s := .Shares.Load }}
- {{$s.Name}} ({{$s.ShareID}}) expires at {{$s.ExpiresAt}}
{{- end }}
```

## `.RetentionChecks`

Array of records produced by a Data Retention action. Each element:

| Field         | Type                               | Notes                          |
| ------------- | ---------------------------------- | ------------------------------ |
| `.Username`   | `string`                           | User whose files were checked. |
| `.Email`      | `[]string`                         | User's email address(es).      |
| `.ActionName` | `string`                           | Name of the retention action.  |
| `.Type`       | `int`                              | `0` = delete, `1` = archive.   |
| `.Results`    | array of per-folder result records | Per-folder results.            |

Each element of `.Results`:

| Field           | Type            |
| --------------- | --------------- |
| `.Path`         | `string`        |
| `.Retention`    | `int` (hours)   |
| `.DeletedFiles` | `int`           |
| `.DeletedSize`  | `int64`         |
| `.Elapsed`      | `time.Duration` |
| `.Info`         | `string`        |
| `.Error`        | `string`        |

## `.EventReports`

Array of per-user reports produced by an Event Report action. Each element:

| Field        | Type                      | Notes                                                                                                |
| ------------ | ------------------------- | ---------------------------------------------------------------------------------------------------- |
| `.Username`  | `string`                  |                                                                                                      |
| `.Email`     | `[]string`                |                                                                                                      |
| `.Truncated` | `bool`                    | True if the report hit the server-side cap (`SFTPGO_HOOK__EVENT_REPORT_MAX_RESULTS`, default 10000). |
| `.Events`    | array of fs event records | File system events for that user.                                                                    |

Each element of `.Events`:

| Field                                | Type                                     |
| ------------------------------------ | ---------------------------------------- |
| `.ID`                                | `string`                                 |
| `.Timestamp`                         | `int64` (nanos; convert via `fromNanos`) |
| `.Action`                            | `string` (upload/download/…)             |
| `.Username`, `.ExternalUsername`     | `string`                                 |
| `.VirtualPath`, `.VirtualTargetPath` | `string`                                 |
| `.FileSize`, `.Elapsed`              | `int64`                                  |
| `.Status`                            | `int` (1 OK, 2 KO, 3 quota exceeded)     |
| `.Protocol`, `.IP`, `.SSHCmd`        | `string`                                 |

## `.ShareExpirationChecks`

Array produced by a Share Expiration Check action. Each element:

| Field      | Type            | Notes                                                                |
| ---------- | --------------- | -------------------------------------------------------------------- |
| `.User`    | `User`          | The owning user. Confidential data is stripped before rendering.     |
| `.Results` | array of single | Per-share results (see `.ShareExpirationResult` below).              |

## `.ShareExpirationResult`

Single share-expiration result, as a struct. Populated either as the per-element shape inside `.ShareExpirationChecks[i].Results`, or directly at the top level when the action runs in split-event mode (one rendering per share). Fields:

| Field         | Type        | Notes                                                                                           |
| ------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `.Share`     | `Share`     | The Share object. Confidential data is stripped.                                                |
| `.Action`    | `int`       | `1` notify, `2` delete (the action the rule applied to this share).                              |
| `.Reason`    | `string`    | Free-form description of why the share was flagged.                                             |
| `.Expiration`| `time.Time` | The share's expiration timestamp.                                                                |

## What is **not** available

There is no ambient reference to "current admin", "now()", "hostname", "env var", or other runtime helpers. Everything a template can see is in the map above. If you need a literal current timestamp, use `.Timestamp` (it is the event time, which is close enough for notifications). If you need an env var, inject it at action-creation time as a literal string in the template.
