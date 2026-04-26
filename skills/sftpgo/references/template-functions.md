# Template functions

SFTPGo Event Action templates are rendered with Go `text/template`. The functions below are registered in the custom `FuncMap` and are available in **every** template field (email subject/body, HTTP body/endpoint/headers, command args/env, IDP user/admin JSON, filesystem paths, etc.).

**Source of truth:** SFTPGo template function registry (verified against v2.7.x).

In addition to these custom functions, all standard `text/template` functions and operators are available: `printf`, `print`, `println`, `len`, `index`, `slice`, `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `and`, `or`, `not`, `call`, `html`, `js`, `urlquery`.

## JSON / encoding

| Function         | Signature             | Example                                                                                                                                                            |                                                             |
| ---------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| `toJson`         | `(val any) string`    | `{{toJson .Metadata}}` → `{"author":"alice"}`. Use when embedding an arbitrary value into JSON. **Quotes the output if it's a string.**                            |                                                             |
| `toJsonUnquoted` | `(val any) string`    | `{{toJsonUnquoted .Name}}` → for string input, returns JSON-escaped contents **without** the outer quotes. Use inside pre-quoted JSON positions: `"user": "{{.Name | toJsonUnquoted}}"`. For non-strings, behaves like `toJson`. |
| `toBase64`       | `(val string) string` | `{{toBase64 (printf "%s:%s" .Name .Role)}}` — useful for Basic auth headers.                                                                                       |                                                             |
| `toHex`          | `(val string) string` | `{{toHex .UID}}`                                                                                                                                                   |                                                             |

Choosing between `toJson` and `toJsonUnquoted` is the #1 source of JSON-body bugs. Rule of thumb:

- **Field position already quoted** (`"key": "<here>"`): use `{{.Field | toJsonUnquoted}}`.
- **Field position unquoted** (value is the whole token, e.g. object/array, or you want automatic quoting for strings): use `{{toJson .Field}}`.

## URL helpers

| Function        | Signature             | Notes                                                                                |
| --------------- | --------------------- | ------------------------------------------------------------------------------------ |
| `urlEscape`     | `(val string) string` | `net/url.QueryEscape`. Use for query parameter values.                               |
| `urlPathEscape` | `(val string) string` | `net/url.PathEscape`. Use for path segments. Keeps `/` unescaped (per `PathEscape`). |

## Path helpers

| Function       | Signature                 | Notes                                                                                        |
| -------------- | ------------------------- | -------------------------------------------------------------------------------------------- |
| `pathDir`      | `(path string) string`    | `path.Dir` — slash-separated. Always use this for virtual paths.                             |
| `pathBase`     | `(path string) string`    | `path.Base`.                                                                                 |
| `pathExt`      | `(path string) string`    | `path.Ext` (includes the leading dot, e.g. `.jpg`).                                          |
| `pathJoin`     | `(elems []string) string` | Cleaned slash-join, **then normalized to an absolute POSIX path** (the result always starts with `/`). Input is a `[]string`, build with `stringSlice`. Trailing slashes are stripped by the cleaning step — see the trailing-slash pitfall in `pitfalls.md` for the delete-contents semantics. |
| `filePathJoin` | `(elems []string) string` | OS-specific (`filepath.Join`). Use only for command args that must hit the local filesystem. |

Example:

```gotemplate
{{pathJoin (stringSlice "quarantine" (pathBase .VirtualPath))}}
```

## String helpers

| Function           | Signature                            | Maps to                                                                                                          |
| ------------------ | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `stringSlice`      | `(args ...string) []string`          | Build a slice literal.                                                                                           |
| `stringJoin`       | `(strs []string, sep string) string` | `strings.Join` — slice first, separator second.                                                                   |
| `stringTrimSuffix` | `(s, suffix string) string`          | `strings.TrimSuffix`                                                                                             |
| `stringTrimPrefix` | `(s, prefix string) string`          | `strings.TrimPrefix`                                                                                             |
| `stringReplace`    | `(s, old, new string) string`        | `strings.ReplaceAll` — source first, then old, then new.                                                          |
| `stringHasPrefix`  | `(s, prefix string) bool`            | `strings.HasPrefix`                                                                                              |
| `stringHasSuffix`  | `(s, suffix string) bool`            | `strings.HasSuffix`                                                                                              |
| `stringContains`   | `(s, substr string) bool`            | `strings.Contains`                                                                                               |
| `stringToLower`    | `(s string) string`                  |                                                                                                                  |
| `stringToUpper`    | `(s string) string`                  |                                                                                                                  |

## Slice / map helpers

| Function         | Signature                            | Notes                                                         |
| ---------------- | ------------------------------------ | ------------------------------------------------------------- |
| `slicesContains` | `(slice []any, val any) bool`        | Membership test. Works on any slice type.                     |
| `mapToString`    | `(val any, m map[any]string) string` | Look up `val` in `m`. Returns empty string if missing.        |
| `createDict`     | `(entries ...any) map[any]string`    | Build a map inline. **Requires an even number of arguments.** |

Example — status code decoding:

```gotemplate
{{- $statusMap := createDict 1 "OK" 2 "ERROR" 3 "QUOTA EXCEEDED" -}}
Status: {{mapToString .Status $statusMap}}
```

## Byte / time helpers

| Function        | Signature                | Notes                                                                                |
| --------------- | ------------------------ | ------------------------------------------------------------------------------------ |
| `humanizeBytes` | `(bytes int64) string`   | IEC formatting: `1048576` → `1.0 MiB`.                                               |
| `fromMillis`    | `(msec int64) time.Time` | Converts a Unix timestamp in milliseconds to `time.Time`. Chain with `.UTC.Format`.  |
| `fromNanos`     | `(nsec int64) time.Time` | Same for nanoseconds. Use on `.Events[i].Timestamp` inside an event report template. |

Example:

```gotemplate
{{.Timestamp.UTC.Format "2006-01-02T15:04:05Z"}}
{{- range .EventReports }}
{{- range .Events }}
  {{(fromNanos .Timestamp).UTC.Format "2006-01-02 15:04:05"}} {{.Action}} {{.VirtualPath}}
{{- end }}
{{- end }}
```

**Reminder:** `.Timestamp` at the top level of the template context is already a `time.Time` — call methods directly, no conversion needed. The `fromMillis` / `fromNanos` helpers are for numeric int64 timestamps you find nested inside `.EventReports[i].Events[j].Timestamp`.

## Control flow (standard `text/template`)

Go's standard control structures work exactly as documented at <https://pkg.go.dev/text/template>:

- `{{if pipeline}}...{{else if pipeline}}...{{else}}...{{end}}`
- `{{range pipeline}}...{{else}}(empty)...{{end}}`
- `{{range $idx, $el := pipeline}}...{{end}}`
- `{{with pipeline}}...{{end}}` (narrows dot)
- `{{define "name"}}...{{end}}` and `{{template "name" pipeline}}` — discouraged inside Event Actions because SFTPGo compiles each field as an independent template, so defines are not shared across fields.

Whitespace trimming: `{{-` trims preceding whitespace, `-}}` trims following whitespace. Use these aggressively when generating JSON to avoid stray newlines that break parsers.

## What is **not** available

Do not use:

- Helm / Sprig functions (`default`, `coalesce`, `nindent`, `trim`, …) — these are not registered.
- Jinja-style filters with `|` that look like Sprig — only the functions above work with the pipe.
- `printf` with non-Go verbs (e.g. no Python `%(name)s` syntax).
- External includes, partials, or file reads.

If you need a function that isn't listed here, combine the primitives (e.g. `default` = `{{if .X}}{{.X}}{{else}}fallback{{end}}`).
