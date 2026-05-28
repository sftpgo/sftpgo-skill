# Pitfalls

Common mistakes when writing SFTPGo Event Action templates. Each item includes the broken version, the fix, and the reason.

## 1. Missing dot prefix — `{{Name}}` instead of `{{.Name}}`

**Broken:**

```gotemplate
User {{Name}} uploaded file {{VirtualPath}}
```

**Fix:**

```gotemplate
User {{.Name}} uploaded file {{.VirtualPath}}
```

**Why.** SFTPGo uses Go `text/template`. `{{Name}}` is parsed as a reference to a function or method named `Name` in the FuncMap — it is not registered, so the template **fails to parse** at action-save time and the API returns a validation error.

A handful of legacy documentation examples (including some in the OpenAPI spec) show the dotless form — those examples are incorrect. Always use the dot.

## 2. `.IDPFields "claim"` for a claim that isn't in `custom_fields`

**Broken** (OIDC binding has `custom_fields: ["email"]`, but template reads `sftpgo_role`):

```gotemplate
{{- $role := index .IDPFields "sftpgo_role" -}}
{{- if eq $role "app.power-user" -}}
  "groups": [{"type":1,"name":"power-users"}]
{{- else -}}
  "groups": [{"type":1,"name":"standard-users"}]
{{- end }}
```

**Fix** — list the claim in the OIDC binding's `custom_fields` array (e.g. `custom_fields: ["email", "sftpgo_role"]`). Then `index .IDPFields "sftpgo_role"` returns the value as expected.

**Why.** `.IDPFields` is a selective projection of the OIDC token claims, controlled by the binding's `custom_fields` config. Claims not listed there are **never** copied into `.IDPFields`. SFTPGo's IDP login event does **not** auto-populate `.Role` or `.Email` from the role/email claims either — the value the binding's `role_field` resolves to drives the JIT user/admin shape but is not exposed on the EventParams. So if your template needs the role inside the rendered output, the only way is to also list the role claim in `custom_fields`.

This was the #1 real-world customer pitfall on OIDC integrations: the template silently defaulted to the "else" branch because the conditional field was always nil.

## 3. `.IDPFields.claim` returns a string **or** a slice

**Broken** — the template assumes the claim is always a string:

```gotemplate
{{- if eq (index .IDPFields "groups") "admins" -}}
```

**Fix** — treat it as a slice and use `slicesContains`:

```gotemplate
{{- if slicesContains (index .IDPFields "groups") "admins" -}}
```

**Why.** OIDC claims can be a single string or a JSON array depending on the IdP and the user. When an IdP sends a multi-group user, `groups` is `[]any`; for single-group users, some IdPs emit a bare string. If you must support both, branch on type:

```gotemplate
{{- $g := index .IDPFields "groups" -}}
{{- if eq (printf "%T" $g) "string" -}}
  {{- if eq $g "admins" -}}yes{{- end -}}
{{- else -}}
  {{- if slicesContains $g "admins" -}}yes{{- end -}}
{{- end -}}
```

## 4. `.Initiator.User.Email` without a guard

**Broken:**

```gotemplate
Changed by {{.Initiator.User.Username}} ({{.Initiator.User.Email}})
```

**Fix:**

```gotemplate
{{- if .Initiator.Admin }}
Changed by admin: {{.Initiator.Admin.Username}} ({{.Initiator.Admin.Email}})
{{- else if .Initiator.User }}
Changed by user: {{.Initiator.User.Username}} ({{.Initiator.User.Email}})
{{- else }}
Changed by: system (no initiator)
{{- end }}
```

**Why.** `.Initiator.User` and `.Initiator.Admin` can both be nil — `.Initiator` is only resolved for **provider events**. For filesystem events, IP blocks, certificate events, schedules, and on-demand runs both accessors are nil. Even on provider events, when `.User` is set `.Admin` is nil and vice versa. Accessing a field on nil inside `text/template` renders empty without an error — confusing in the output. The three-way guard above makes the intent explicit and produces a sensible message for the system / fs-event case.

For filesystem events specifically, the acting user's identity is exposed directly on the event: `.Name`, `.Role`, `.Email`, `.Protocol`, `.IP`. Don't reach for `.Initiator` there.

## 5. `.Object.Username` instead of `.Object.User.Username`

**Broken:**

```gotemplate
{{.Object.Username}}
```

**Fix:**

```gotemplate
{{- if .Object.User}}{{.Object.User.Username}}{{end}}
```

Or when you want the full JSON:

```gotemplate
{{.Object.JSON}}
```

**Why.** `.Object` is a lazy wrapper, not the concrete object. It exposes typed accessors (`.User`, `.Admin`, `.Group`, `.Share`) and a `.JSON` method. Accessing `.Username` directly on `.Object` does not work because the wrapper has no such field.

## 6. Unescaped dynamic content in a JSON body

**Broken:**

```gotemplate
{"user":"{{.Name}}","path":"{{.VirtualPath}}"}
```

Breaks as soon as a username contains a `"` or a path contains a backslash. Also breaks at the action-save step if the template is validated against a JSON parser.

**Fix** — use `toJsonUnquoted` inside pre-quoted positions:

```gotemplate
{"user":"{{.Name | toJsonUnquoted}}","path":"{{.VirtualPath | toJsonUnquoted}}"}
```

Or skip quoting entirely and let `toJson` emit the quotes:

```gotemplate
{"user":{{toJson .Name}},"path":{{toJson .VirtualPath}}}
```

Never hand-roll quoting for dynamic data.

## 7. `stringReplace` and `stringJoin` argument order

Both helpers wrap the corresponding `strings` package functions and keep the standard-library order, which is what trips up people more than they expect when they write the args from memory:

- `stringReplace` is `(s, old, new)` — source first, then `old`, then `new`. Same as `strings.ReplaceAll(s, old, new)`.
- `stringJoin` is `(strs, sep)` — slice first, separator second. Same as `strings.Join(strs, sep)`.

```gotemplate
{{stringReplace .VirtualPath "/old" "/new"}}    {{/* OK: replace "/old" with "/new" in .VirtualPath */}}
{{stringJoin .Errors ", "}}                     {{/* OK: join the .Errors slice with ", " */}}
```

The most common mistake is calling `stringJoin` with the separator first (`stringJoin "," .Errors`) — Go templates raise a runtime error because the helper expects the slice in the first position and gets a string instead.

## 8. `pathJoin` expects a single `[]string` argument

**Broken:**

```gotemplate
{{pathJoin "/home" .Name "inbox"}}
```

**Fix:**

```gotemplate
{{pathJoin (stringSlice "/home" .Name "inbox")}}
```

**Why.** `pathJoin` is registered as `func(elems []string) string`, not variadic. Build the slice with `stringSlice`.

## 9. `.FileSize` for a `pre-upload` event is always 0

`pre-*` events fire before the data is seen, so `.FileSize` and `.Elapsed` are `0`. If you need size-based logic, condition on `.Event`:

```gotemplate
{{if and (eq .Event "upload") (gt .FileSize 104857600)}}
  Large file uploaded: {{humanizeBytes .FileSize}}
{{end}}
```

## 10. Trailing slash on a Filesystem Delete path

**Intentional behavior:**

```yaml
deletes:
  - "/user-inbox/"     # deletes the CONTENTS of /user-inbox, keeps the directory
  - "/user-archive"    # deletes the entire /user-archive directory recursively
```

This is documented upstream behavior, not a bug. If you template the path:

```gotemplate
{{pathJoin (stringSlice "/user-inbox/")}}   {{/* loses the trailing slash after path.Clean */}}
```

`pathJoin` (and Go's `path.Clean`) strips trailing slashes. For delete-contents semantics, **do not run the path through `pathJoin`**; write the literal path with the slash, or re-append the slash after cleaning:

```gotemplate
{{printf "%s/" (pathJoin (stringSlice "/some/templated/path"))}}
```

## 10b. PGP target derived via `pathDir` needs an explicit trailing slash

The PGP action treats a target ending with `/` as a destination directory and derives the per-file output filename (encrypt: append `.pgp`; decrypt: strip the last extension or append `.dec`). Without the trailing slash the target is a literal output path.

A common config to "encrypt next to the source" templates the parent directory:

```yaml
target: /{{ pathDir .VirtualPath }}/   # trailing slash REQUIRED
```

Drop the slash and the rule works only by luck for files at root: `/{{ pathDir "/file.txt" }}` renders to `//`, which `path.Clean`s to `/`. Root is intrinsically a directory, so the derive branch fires anyway and the file lands at `/file.txt.pgp`. For a non-root upload like `/sub/file.txt` the same template renders to `/sub` (no trailing slash → literal output), and PGP would write the encrypted bytes overwriting (or colliding with) the directory `/sub`. Always include the trailing slash for portability across upload paths.

## 10c. Source-only PGP / Compress: rejected at validate

PGP (encrypt and decrypt) and Compress reject configurations with `Folder` set and `TargetFolder` empty. These actions produce derived output that has no sensible per-user destination when the source is shared: per-user iteration would replicate identical output across every matching user's home, or race with disposition on the shared source.

Canonical pattern for in-place batch processing inside a shared folder: **set both `Folder` and `TargetFolder` to the same value**. Explicit intent, no inference. Same-folder configurations are valid and runtime collision detection is path-level (`PGP.validate()` rejects `key == value` unconditionally; runtime `checkSamePath` resolves virtual paths through `IsSameResource`).

Copy and Extract are intentionally **not** subject to this rule: their source-only configuration is the documented "distribute a shared resource into every user's home" pattern.

## 11. Schedule trigger sees no `.VirtualPath` / `.Event`

If the rule trigger is `schedule` (3) and the template references `.VirtualPath`, the output is empty. Schedules have no filesystem context. Use chained actions (retention action → email template with `.RetentionChecks`) or restructure as a provider/fs-event rule.

## 12. "Template validated OK but webhook receives empty fields"

Root cause is almost always **trigger / field mismatch** — see pitfall 11 above and `trigger-types.md`. The template **compiles** because the keys always exist in the map; the values are just zero-valued for this trigger.

## 13. System-executed actions don't fire other rules

If a rule action runs **as the system** rather than on behalf of a real user, the events that action emits do not chain into other rules. Both axes are filtered:

- **Filesystem operations from system-context actions** — Filesystem actions (rename, copy, delete, compress, PGP, etc.), Data Retention deletes (user- or folder-scoped), and any other fs operation issued by the event manager. The connection protocol (`EventAction` or `DataRetention`) is filtered before fs-event rules are matched. A separate "rename on upload" rule will not see the file the first action wrote.
- **Provider events from system-driven operations** — provider rules (`Add user`, `Update share`, etc.) explicitly skip events whose executor is `__system__`. Adding/updating users from another rule, automated disables from IP-blocked rules, JIT-provisioning side effects: none of these fire provider-event rules.

The intent is to keep the rule graph predictable and prevent cascading triggers.

**Supported pattern for multi-step workflows:** list multiple actions on the **same** rule, in the desired order. Within a rule, actions execute sequentially and share `EventParams` — no chaining via emitted events is needed (or possible).

**For "trash on delete" or "archive on retention" workflows:** use the **Archive folder** option built into the Data Retention action, not a separate fs-event rule. The retention's own pipeline does the move; a downstream rule would never trigger.

Mitigation: lint templates against the trigger. A human-scale check is to look at `trigger-types.md` "Populates" column for the chosen trigger and confirm every dotted field you reference is in that list.

## 14. Name / role / group filters on a system-context action skip the action

Symptom: an fs action that worked yesterday now logs `no <op> executed` after the rule was edited to add a name, role, or group filter. Or a freshly-built scheduled rule that copies/PGP-encrypts/deletes inside a virtual folder never runs anything despite the schedule firing.

Cause. The synthetic system identity used by system-context actions has no real username, role, or groups (it has the literal name `__system__` and empty role/groups). Per-user condition filters describe real users; checking them against the system identity always fails, and the dispatcher reports "no <op> executed" because no iteration succeeded. This applies to:

- Filesystem actions (rename, delete, mkdir, exist, metadata-check, copy, pgp, compress, extract) when configured with both source and target folders, or with a source folder for source-only ops.
- IMAP action when a target folder is set.
- Folder-scoped data retention.

Fix. Remove the user/role/group filter from the rule. System actions describe shared resources, not users. Use trigger and event-status filters to scope a system rule; reserve user filters for per-user rules.

## 15b. Microsoft IMAP OAuth2 — `AUTHENTICATE failed` after a successful token refresh

Symptom: an IMAP action against `outlook.office365.com:993` logs `new oauth2 token saved …` immediately followed by `unable to login using OAuth2 to the IMAP server …: AUTHENTICATE failed.` The token endpoint succeeded, so it's not a scope or client-secret problem.

Cause. SFTPGo sends the IMAP `Username` field in the SASL XOAUTH2 string; Exchange Online validates that the user who authorized the OAuth flow has access to **that mailbox**. If the action's `Username` points at a shared/service mailbox (e.g. `inbound@domain.com`) but the operator clicked **Get** while signed in as a different account (typically an admin without explicit Full Access on that mailbox), Exchange rejects the AUTHENTICATE even though the token itself is valid.

Fix.
1. Grant a delegate user Full Access to the target mailbox: `Add-MailboxPermission -Identity inbound@domain.com -User delegate@domain.com -AccessRights FullAccess` (Exchange Online PowerShell).
2. Revoke the existing app consent from [My Apps](https://myapps.microsoft.com/) so the previous refresh token is invalidated.
3. Re-run **Get** in the IMAP action form, signing in as `delegate@domain.com`. The new refresh token is bound to that identity and IMAP AUTHENTICATE succeeds.

## 15. Backup chain cannot delete or rename the backup file

Symptom: an action chained after the Backup action that tries to delete or move the just-produced dump file is denied with a permission error.

Cause. The synthetic user used by the backup chain is granted list and download access only on the backups directory. Email and HTTP attachments can read the dump (PermDownload), but Filesystem actions (delete, rename, copy) targeting the backups directory are refused at the connection layer. This is a security guarantee: even a misrouted or compromised downstream action cannot tamper with backups.

Fix. Don't chain destructive operations after Backup. To clean up old dumps, configure data retention separately on the backups directory (folder-scoped retention with the backups folder as source).

