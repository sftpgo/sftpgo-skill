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

## 11. Schedule trigger sees no `.VirtualPath` / `.Event`

If the rule trigger is `schedule` (3) and the template references `.VirtualPath`, the output is empty. Schedules have no filesystem context. Use chained actions (retention action → email template with `.RetentionChecks`) or restructure as a provider/fs-event rule.

## 12. "Template validated OK but webhook receives empty fields"

Root cause is almost always **trigger / field mismatch** — see pitfall 11 above and `trigger-types.md`. The template **compiles** because the keys always exist in the map; the values are just zero-valued for this trigger.

Mitigation: lint templates against the trigger. A human-scale check is to look at `trigger-types.md` "Populates" column for the chosen trigger and confirm every dotted field you reference is in that list.
