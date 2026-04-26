# Action types — enum ↔ config mapping

SFTPGo Event Actions are discriminated by the integer `type` field on the `BaseEventAction` schema. The matching sub-configuration lives in `BaseEventActionOptions` under a dedicated JSON key. **Only the sub-config matching the `type` is read; all others are zeroed on validation.**

**Source of truth:** SFTPGo Event Manager runtime behavior (verified against v2.7.x).

## Table

| `type` | Name                      | JSON sub-field                | Template-rendered fields                                                                                                         | License tier |
| ------ | ------------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| 1      | HTTP                      | `http_config`                 | `endpoint`, `body`, `query_parameters[].value`, `headers[].value`, `parts[].filepath`, `parts[].body`, `parts[].headers[].value` | Standard     |
| 2      | Command                   | `cmd_config`                  | `args[]`, `env_vars[].value`                                                                                                     | Standard     |
| 3      | Email                     | `email_config`                | `recipients[]`, `bcc[]`, `subject`, `body`, `attachments[]`                                                                      | Standard     |
| 4      | Backup                    | *(none — no template fields)* | —                                                                                                                                | Standard     |
| 5      | User quota reset          | *(none)*                      | —                                                                                                                                | Standard     |
| 6      | Folder quota reset        | *(none)*                      | —                                                                                                                                | Standard     |
| 7      | Transfer quota reset      | *(none)*                      | —                                                                                                                                | Standard     |
| 8      | Data retention check      | `retention_config`            | *(none — configuration is per-folder, no templates)*                                                                             | Standard     |
| 9      | Filesystem                | `fs_config`                   | Depends on `fs_config.type` — see [§9 Filesystem](#type-9--filesystem)                                                           | Standard     |
| 11     | Password expiration check | `pwd_expiration_config`       | *(none)*                                                                                                                         | Standard     |
| 12     | User expiration check     | *(none)*                      | —                                                                                                                                | Standard     |
| 13     | IDP account check         | `idp_config`                  | `template_user`, `template_admin`                                                                                                | **Premium**  |
| 14     | User inactivity check     | `user_inactivity_config`      | *(none)*                                                                                                                         | Standard     |
| 15     | Rotate logs               | *(none)*                      | —                                                                                                                                | Standard     |
| 16     | IMAP                      | `imap_config`                 | `email[]`, `subject`, `body`, `attachments[]`, `target_folder`, `flags[]`                                                        | Standard     |
| 17     | ICAP                      | `icap_config`                 | `paths[]`                                                                                                                        | **Premium**  |
| 18     | Share expiration check    | `share_expiration_config`     | *(none)*                                                                                                                         | Standard     |
| 19     | Event report              | `event_report_config`         | *(none — filters are literal, report data surfaces via `.EventReports`)*                                                         | **Premium**  |

Type `10` is reserved and is not a valid action type.

## Per-type notes

### type 1 — HTTP

`http_config` fields:

| Field                      | Template?    | Notes                                                                  |
| -------------------------- | ------------ | ---------------------------------------------------------------------- |
| `endpoint`                 | yes          | Full URL. Can interpolate `{{.Name}}`, `{{.VirtualPath}}`, etc.        |
| `method`                   | no           | `GET`, `POST`, `PUT`, `DELETE`.                                        |
| `timeout`                  | no           | Seconds (`1..120`).                                                    |
| `headers[].key`            | no (literal) |                                                                        |
| `headers[].value`          | yes          |                                                                        |
| `query_parameters[].key`   | no (literal) |                                                                        |
| `query_parameters[].value` | yes          |                                                                        |
| `username`, `password`     | no           | Basic auth, credentials are stored as secrets.                         |
| `body`                     | yes          | Raw body. Only for single-part requests.                               |
| `parts[].name`             | no (literal) | For multipart.                                                         |
| `parts[].filepath`         | yes          | Optional file path in the user's VFS — the file is streamed as a part. |
| `parts[].body`             | yes          | Inline part body.                                                      |
| `parts[].headers[].value`  | yes          | Per-part header values.                                                |

Either `body` or `parts` must be set (not both). Multipart parts need unique `name`s.

### type 2 — Command

`cmd_config`:

| Field              | Template?              | Notes                                                                                                                  |
| ------------------ | ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `cmd`              | no                     | Absolute path or binary name — must be listed in `EnabledActionCommands` (see `SFTPGO_HOOK__ENABLED_ACTION_COMMANDS`). |
| `args[]`           | **yes** — each element | Argument values are rendered.                                                                                          |
| `env_vars[].key`   | no (literal)           |                                                                                                                        |
| `env_vars[].value` | yes                    |                                                                                                                        |
| `timeout`          | no                     | Seconds.                                                                                                               |

### type 3 — Email

`email_config`:

| Field                   | Template? | Notes                                                                                                   |
| ----------------------- | --------- | ------------------------------------------------------------------------------------------------------- |
| `recipients[]`, `bcc[]` | yes       | Each address rendered separately.                                                                       |
| `subject`               | yes       | Single line.                                                                                            |
| `body`                  | yes       | Text or HTML (set `content_type`).                                                                      |
| `attachments[]`         | yes       | Each entry is a virtual path rendered against the user's VFS.                                           |
| `attach_event_report`   | no        | Bool — auto-attaches the JSON/CSV event report when an Event Report action has run earlier in the rule. |
| `content_type`          | no        | `0` = text (default), `1` = HTML.                                                                       |

### type 9 — Filesystem

`fs_config.type` discriminates the sub-action. Each sub-action has its own paths/config block; the unused ones are ignored. See `references/trigger-types.md` for which triggers allow this action type.

| `fs_config.type` | Name           | Config field                                                     | Template-rendered                                      |
| ---------------- | -------------- | ---------------------------------------------------------------- | ------------------------------------------------------ |
| 1                | Rename         | `renames[]` (old, new)                                           | both paths per entry                                   |
| 2                | Delete         | `deletes[]` (`[]string`)                                         | each path (trailing `/` = delete *contents*, keep dir) |
| 3                | Mkdirs         | `mkdirs[]` (`[]string`)                                          | each path                                              |
| 4                | Exist          | `exist[]` (`[]string`)                                           | each path                                              |
| 5                | Compress       | `compress` (`paths[]`, `name`)                                   | both                                                   |
| 6                | Copy           | `copy[]` (source, target)                                        | both paths per entry                                   |
| 7                | PGP            | `pgp` (`paths[]`, private/public key, passphrase, mode, profile) | `paths[]`                                              |
| 8                | Metadata check | `folders[]`                                                      | none (folders are literal)                             |
| 9                | Decompress     | `decompress` (source path, target path)                          | both paths                                             |

Trailing-slash delete semantics: a path ending with `/` deletes the **contents** of that directory without removing the directory itself. Without trailing slash the directory is removed recursively.

### type 13 — IDP account check

`idp_config`:

| Field            | Template?             | Notes                                                                          |
| ---------------- | --------------------- | ------------------------------------------------------------------------------ |
| `mode`           | no                    | `0` = create-or-update, `1` = create-only (do not modify an existing account). |
| `template_user`  | **yes** — entire JSON | Rendered, then parsed as a JSON `User` object.                                 |
| `template_admin` | **yes** — entire JSON | Rendered, then parsed as a JSON `Admin` object.                                |

At least one of `template_user`/`template_admin` must be set. Which one is used depends on the IDP binding (user-login vs admin-login).

This action is designed for **JIT (just-in-time) provisioning** after OIDC login. Only supported when the rule is triggered by an IDP login event (`trigger = 7`).

See `examples.md` for full templates and `pitfalls.md` for the `.IDPFields` / `custom_fields` gotcha — the role and email claims are exposed inside the template only if they are listed in the OIDC binding's `custom_fields` array.

### type 16 — IMAP

`imap_config` (similar to `email_config` but uploaded into a mailbox via IMAP APPEND):

| Field                                             | Template?    | Notes                                                                                                                             |
| ------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `host`, `port`, `username`, `password`, `use_tls` | no           | Connection settings (password is a secret).                                                                                       |
| `email[]`                                         | yes          | Envelope recipients (rendered per entry).                                                                                         |
| `subject`                                         | yes          |                                                                                                                                   |
| `body`                                            | yes          |                                                                                                                                   |
| `attachments[]`                                   | yes          | User VFS paths, rendered.                                                                                                         |
| `target_folder`                                   | yes          | Target IMAP mailbox (rendered — common values: `INBOX`, `Archive`). Non-empty target → action runs once as system (not per-user). |
| `flags[]`                                         | no (literal) | Optional IMAP message flags (e.g. `\Seen`).                                                                                       |

### type 17 — ICAP

`icap_config`:

| Field               | Template?              | Notes                                                                                                                                                        |
| ------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `endpoint`          | no                     | ICAP URI (`icap://host:port/service` or `icaps://…`). Default port 1344.                                                                                     |
| `method`            | no                     | Only `REQMOD` is supported.                                                                                                                                  |
| `timeout`           | no                     | Seconds to wait for the final ICAP response.                                                                                                                 |
| `skip_tls_verify`   | no                     | Bool; for `icaps://` endpoints with self-signed certs.                                                                                                       |
| `headers[]`         | no (literal key/value) | Additional ICAP request headers (e.g. `X-Client-IP`).                                                                                                        |
| `paths[]`           | **yes** — each path    | Virtual paths to scan. For on-upload scanning, the canonical value is `{{.VirtualPath}}`.                                                                    |
| `block_action`      | no                     | What to do when the ICAP server flags the file as infected. Enum: `1` ignore, `2` delete, `3` quarantine.                                                    |
| `adapt_action`      | no                     | What to do when the ICAP server returns a modified file. Enum: `1` ignore, `2` delete, `3` quarantine, `4` overwrite the original with the modified content. |
| `failure_policy`    | no                     | What to do when the scan itself fails (server unreachable, timeout, etc.). Enum: `1` ignore, `2` delete, `3` quarantine.                                     |
| `quarantine_folder` | no                     | Virtual folder name where quarantined files are moved. Required when any of the policy fields is set to `3` (quarantine).                                    |
| `quarantine_path`   | yes                    | Target path inside `quarantine_folder`; supports template placeholders so quarantined files can be partitioned by user/date.                                 |

Paths must resolve inside the user's VFS. For staged upload rules the active upload is stored on the temp path — `.VirtualPath` is resolved correctly against the temp location in staged flows.

### type 19 — Event report

`event_report_config`:

| Field                                                            | Template? | Notes              |
| ---------------------------------------------------------------- | --------- | ------------------ |
| `fs_actions[]`, `statuses[]`, `time_window`                      | no        | Filters (literal). |
| `split_reports`                                                  | no        | Bool.              |
| `attach_event_report` *(on the Email action that consumes this)* | no        |                    |

Event report action populates `.EventReports` for downstream email/HTTP actions. The template consumers use `.EventReports` directly.
