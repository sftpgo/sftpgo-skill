---
name: sftpgo
description: "Reference skill for SFTPGo — the managed file transfer platform. Load this skill when the user is writing, debugging, or generating SFTPGo configuration, REST API payloads, or Event Action / IDP / email / ICAP / filesystem templates for either the WebAdmin UI or the REST API. Current scope: authoring guide for Go text/template strings used across Event Actions (trigger↔context mapping, action-type↔sub-config mapping, full template context, registered functions, pitfalls, ready-to-paste examples including OIDC JIT provisioning). Expected to grow over time to cover other SFTPGo aspects (REST API idioms, user/group configuration, storage backends) — check the references/ directory for what is currently covered."
user-invocable: true
license: Apache-2.0
compatibility: Claude Code and any AI coding agent that consumes Markdown-based skill manifests. Content verified against SFTPGo v2.7.x runtime behavior.
allowed-tools: Read Edit Write Glob Grep Bash
metadata:
  author: "Nicola Murino"
  version: "0.5.0"
  upstream:
    site: "https://sftpgo.com"
    rest_api_docs: "https://sftpgo.com/rest-api"
    verified_against: "v2.7.x"
  current_scope:
    - Event Action templates — Go text/template authoring for email / HTTP / command / IDP / IMAP / ICAP / filesystem actions, as accepted both by the REST API (`/api/v2/events/actions`, `/api/v2/events/rules`) and by the WebAdmin UI form fields.
    - Deployment & configuration — installation methods, `sftpgo.<ext>` config file, `SFTPGO_…` environment variable convention, `env.d/` recipes, sysadmin pitfalls.
    - SaaS (fully-managed offering at `sftpgo.com`) — how to detect, `default` group convention, integrated S3 (`bucket=default`, `region=default`), tier-gated features, configuration changes that trigger an automated restart, what to avoid suggesting (CLI / env vars / config files).
    - Editions and distribution channels — Open Source vs Enterprise, third-party redistributions (managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, unofficial Docker images), what Enterprise adds, detection signals, and the rule to flag Enterprise-only features plainly when the user is on OSS.
  references:
    - references/user-guide.md
    - references/editions.md
    - references/deployment.md
    - references/saas.md
    - references/template-context.md
    - references/template-functions.md
    - references/action-types.md
    - references/trigger-types.md
    - references/examples.md
    - references/pitfalls.md
---

# SFTPGo reference skill

Generic reference skill for SFTPGo. Content is organized under `references/` by topic and grows over time. The skill is the primary entry point for **any** SFTPGo-related task — REST API usage, WebAdmin configuration, and Event Action template authoring. The deepest current coverage is on Event Action templates; the REST API and WebAdmin topics lean heavily on the authoritative OpenAPI spec (below) plus the enum and cross-field notes in `references/`.

## When this skill applies

Load this skill when the user is:

- **Installing, deploying, or configuring an SFTPGo server** — picking the install method, writing `sftpgo.json` / env files, composing `SFTPGO_…` environment variables correctly, setting up the data provider, enabling HTTPS / ACME / Defender / metrics, activating a license. See `references/deployment.md`.
- **Calling the SFTPGo REST API** — any request to `/api/v2/*` (users, groups, folders, event rules, event actions, shares, admins, API keys, quota, retention, etc.). The skill teaches the shape conventions (tagged unions, required fields, secret envelopes, status codes) that make payloads validate on the server.
- **Configuring SFTPGo through the WebAdmin** — filling the forms for users, groups, virtual folders, event rules, event actions, shares, server config. The WebAdmin fields map 1-to-1 to the REST API JSON shape, so the same knowledge applies.
- **Writing Event Action templates** — Body/Subject/Endpoint/Headers/Paths/args/env/template_user/template_admin on any action type (HTTP, Command, Email, IMAP, ICAP, Filesystem, IDP account check). Both the raw paste-into-WebAdmin form and the full REST API JSON payload form are covered in `references/examples.md`.
- **Debugging** a template that "renders empty", a payload that gets rejected, or a rule that fires but produces the wrong output.

## Where to get the authoritative sources

This skill is deliberately thin: it indexes the rest. Three external sources carry the ground truth, and this skill tells you *which* to consult for *which* question.

**OpenAPI spec** — for field names, enum values, required fields, schema shape:

- `https://sftpgo.com/assets/openapi.yaml` — refreshed on every public release, always the **latest** publicly released SFTPGo version.
- `openapi/openapi.yaml` inside the deployed SFTPGo installation — authoritative for **that** server (may be older than the public copy).

If the user is deploying an older SFTPGo, prefer the installed file. If the user is reasoning about the current product, the public URL is the reference.

**Public documentation** (`https://docs.sftpgo.com/enterprise/`, GitHub source at `sftpgo/docs` branch `enterprise`) — for operational how-to (install, storage backend setup, groups, shares, authentication, hooks, metrics, etc.). Fetch raw markdown via:

```text
https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md
```

Two dual-use index pages to orient fast without reading the full navigation:

- `documentation-map.md` — task → file cheat sheet
- `glossary.md` — disambiguation of SFTPGo-specific terms

Every indexed file has a `description:` frontmatter line: read that first to decide whether the file is worth fetching.

**This skill's reference files** — for what the OpenAPI spec and the public docs cannot easily express: enum↔sub-config mappings, per-trigger placeholder availability, template context shape, quoting rules, ready-to-paste templates, common pitfalls.

## What Event Action templates use under the hood

This section describes the templating used in Event Action fields (Body / Subject / Endpoint / Headers / Paths / args / env / template_user / template_admin) — **not** the WebAdmin / WebClient page templates, which are a separate concern.

The engine differs by edition:

- **Enterprise** — full **Go `text/template`** (not Handlebars, not Jinja, not `html/template`). Supports `{{if}}` / `{{range}}` / `{{with}}` / pipes (`{{.X | toJson}}`), an extended `FuncMap` (see `references/template-functions.md`), and structured accessors like `{{.Object.User.Email}}` / `{{index .IDPFields "claim"}}`. Templates are compiled at action-save time, so a syntax error rejects the save with a validation error; a semantic error (referencing a field the trigger does not populate) silently renders as the empty string at runtime.
- **Open Source** — a flat `strings.Replacer` pass over a fixed list of placeholders. The dot-prefixed syntax (`{{.Name}}`, `{{.VirtualPath}}`, …) is identical in shape to the Enterprise form, but it is **string substitution only** — no conditionals, no ranges, no functions, no nested accessors. OIDC custom fields are exposed as flat tokens like `{{.IDPField<claim>}}` (e.g. `{{.IDPFielddepartment}}`), there is no `{{.IDPFields}}` map and no `{{index ... }}`. Provider-event object data is reachable only through the raw `{{.ObjectData}}` / `{{.ObjectDataString}}` placeholders. See `references/editions.md` for the full list of placeholders that exist on OSS.

Most of the rest of this skill targets the Enterprise template engine. When the user is on OSS, drop conditionals / pipes / functions and stay within the OSS placeholder list — the rendered output is otherwise often "the literal placeholder string", which looks like a missing-field bug.

Both editions share these rules:

- Dot prefix is **mandatory** for field access: `{{.Name}}`, not `{{Name}}`.
- A field that the trigger does not populate renders as the empty string (Enterprise) or stays as the literal placeholder text (OSS) — neither is an error.

## Workflow

**When the user's question is about SFTPGo in general** (creating users, groups, shares, setting up event rules, troubleshooting, picking a backend, etc.), start from `references/user-guide.md` — short orientation with pointers into the public docs. Fetch the public doc file (raw GitHub URL) when you need operational depth; its frontmatter description tells you if it's the right one before you commit tokens to the full read.

**When the user's question is about installation, server configuration, or environment variables** (package install, Docker/K8s, editing `sftpgo.json`, composing `SFTPGO_…` env vars, enabling HTTPS / ACME / Defender, data-provider setup, licensing), start from `references/deployment.md`. It contains the env-var naming rule (the part an AI must get exactly right), ready-to-paste `env.d/` recipes, and the sysadmin pitfall list.

**When the user is on the fully-managed SaaS** (dedicated per-instance subdomain we assigned, custom domain via CNAME onto it, `bucket=default`+`region=default` in their config, mentions "managed service" / "SaaS" / "sftpgo.com plan"), start from `references/saas.md` — the customer has no shell access, so deployment recipes, env vars, and CLI commands are off-limits, and there is a pre-created `default` group wired to the integrated storage that is the right answer for most user-creation questions. If detection is ambiguous, ask the user directly: *"Are you using the fully-managed SaaS from sftpgo.com, or are you running SFTPGo on your own server?"*

**Before committing to an answer on any unfamiliar installation** (especially when the user mentions a managed-hosting provider, a NAS or home-lab app store, a community Linux-package archive, a Docker Hub `drakkan/sftpgo` image, a "free version", or "no license"), consult `references/editions.md`. The skill's content overwhelmingly targets Enterprise; the Open Source edition has a reduced Event Manager, basic OIDC, no WOPI / LDAP / ICAP / PGP / staged-upload / event-report / clustering / audit-logs UI, and no support from third-party resellers. `editions.md` has detection signals, the canonical difference list, and the rule to flag Enterprise-only features plainly (once, no sales pitch) when answering an OSS user. When the edition is uncertain, ask.

**When the user is specifically authoring an Event Action template:**

1. **Identify the action type.** Map to one of the entries in `references/action-types.md`. This tells you which struct field holds the template string and which sub-fields are template-rendered.
2. **Identify the trigger type.** Some template fields are only populated for specific triggers (e.g. `.IDPFields` is only populated for `EventTriggerIDPLogin`; `.RetentionChecks` is only populated after a retention-check action has run earlier in the chain). See `references/trigger-types.md`.
3. **Pick the right fields to reference.** Use `references/template-context.md` — it lists every key in the template context with its type and provenance.
4. **Write defensively.** Nil-check `.Initiator.User` / `.Initiator.Admin` before accessing fields; use `toJson`/`toJsonUnquoted` when the output is embedded in JSON to avoid broken quoting. See `references/pitfalls.md`.
5. **Produce two outputs** when the user context is ambiguous:
   - The **raw template string** (ready to paste into the WebAdmin field).
   - The **full JSON payload** to POST to `/api/v2/events/actions` (for automation).
   Show both, briefly labelled, unless the user has already told you which one they want.
6. **Verify against the live OpenAPI spec** when a specific field name, enum value, or schema shape is in doubt. The reference files document the shape, the spec is the source of truth.

## Placeholder picker — by trigger

Before writing any template, settle which fields are actually populated for the chosen trigger. Using a placeholder that is empty at runtime is the #1 source of "webhook fires but body looks wrong" bugs.

| Trigger                                                                                                                                          | Always-safe placeholders                                                                                                                                                         | Extra placeholders available                                                                                                                                                        | Placeholders that are empty / zero — do not use                                                                                                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1 — Filesystem event** (`upload`, `download`, `delete`, `rename`, `mkdir`, `rmdir`, `copy`, `ssh_cmd`, and their `pre-*` / `first-*` variants) | `.Name`, `.Event`, `.Status`, `.VirtualPath`, `.FsPath`, `.ObjectName`, `.ObjectType`, `.Protocol`, `.IP`, `.Timestamp`, `.UID`, `.Role`, `.Email`, `.Errors`, `.Initiator.User` | `.VirtualTargetPath` / `.FsTargetPath` (rename/copy only); `.FileSize` / `.Elapsed` (**only on post-upload/download**, zero for `pre-*`); `.Metadata` (cloud backends only).        | `.IDPFields`, `.Initiator.Admin`, `.Object`, `.RetentionChecks`, `.EventReports`, `.ShareExpirationChecks`.                                                                                               |
| **2 — Provider event** (`add` / `update` / `delete` on user, group, share, folder, admin, …)                                                     | `.Event`, `.ObjectName`, `.ObjectType`, `.Object`, `.Object.JSON`, `.Timestamp`, `.UID`, `.Initiator` (one of `.User` / `.Admin` populated).                                     | `.Name` = affected username for user events; `.Object.User` / `.Object.Admin` / `.Object.Group` / `.Object.Share` depending on `.ObjectType`.                                       | `.VirtualPath`, `.FsPath`, `.FileSize`, `.Elapsed`, `.Protocol`, `.IP`, `.IDPFields`, `.Errors`.                                                                                                          |
| **3 — Schedule (cron)**                                                                                                                          | `.Timestamp`, `.UID`. Chained data from earlier actions in the same rule: `.RetentionChecks`, `.EventReports`, `.ShareExpirationChecks`.                                         | Chained Share Expiration can populate `.ShareExpirationResult` when split-mode is on.                                                                                               | `.Name`, `.VirtualPath`, `.FsPath`, `.Event`, `.Status`, `.FileSize`, `.Protocol`, `.IP`, `.Role`, `.Email`, `.IDPFields`, `.Object`. Everything that would require a subject user or file path is blank. |
| **4 — IP blocked**                                                                                                                               | `.IP`, `.Protocol`, `.Event`, `.Timestamp`, `.UID`, `.Errors`.                                                                                                                   | —                                                                                                                                                                                   | `.Name`, `.VirtualPath`, `.FsPath`, `.FileSize`, `.Object`, `.Initiator`, `.IDPFields`.                                                                                                                   |
| **5 — Certificate renewal (ACME)**                                                                                                               | `.Event` (`acme_ok` / `acme_error`), `.Timestamp`, `.UID`, `.Errors`.                                                                                                            | —                                                                                                                                                                                   | Everything else.                                                                                                                                                                                          |
| **6 — On demand**                                                                                                                                | `.Name` (when the POST body lists users), `.Timestamp`, `.UID`.                                                                                                                  | Whatever earlier chained actions populate.                                                                                                                                          | Depends on how the rule is invoked — default: same blanks as schedule.                                                                                                                                    |
| **7 — IDP login**                                                                                                                                | `.Name`, `.ExtName`, `.Role`, `.Email`, `.Protocol` (= `OIDC`), `.IP`, `.Timestamp`, `.UID`, `.IDPFields`.                                                                       | Anything in `.IDPFields` **must** match a claim listed in the OIDC binding's `custom_fields`; the role claim comes via `.Role` (populated from `role_field`), not via `.IDPFields`. | `.VirtualPath`, `.FsPath`, `.FileSize`, `.Event` (typed as IDP event, not filesystem), `.Object`, `.Errors`.                                                                                              |

Rule of thumb:

- If the user is writing a chat / email notification for **uploads**, go straight to `.Name` + `.VirtualPath` + `humanizeBytes .FileSize` + `.Protocol` + `.IP` — these cover 90% of real notifications.
- For **provider events**, lead with `.Event` + `.ObjectType` + `.ObjectName` and fall back to `.Object.JSON` when you need full detail.
- For **scheduled reports**, the subject/body must be written around `.RetentionChecks` or `.EventReports` — user-scoped placeholders like `.Name` are empty.
- For **IDP JIT**, `.Role` drives the group mapping and `.IDPFields` carries the rest of the configured claims. Nothing else is reliably populated.

## Quick-start templates for common cases

Full worked examples with both WebAdmin-paste form and REST API JSON form are in `references/examples.md`. Highlights:

- **Email on upload** — body uses `{{.Name}}`, `{{.VirtualPath}}`, `{{humanizeBytes .FileSize}}`.
- **Chat webhooks** — Slack (text + Block Kit), Microsoft Teams (MessageCard and Adaptive Card), Discord (text + embed), Google Chat, Mattermost — all under `references/examples.md` §2. The body is JSON; always wrap dynamic strings with `{{… | toJsonUnquoted}}` inside pre-quoted positions, or emit them with `{{toJson …}}` unquoted. When in doubt, pipe through `toJson` — it handles quoting for strings automatically.
- **Generic audit webhook** with full `.Object.JSON` for provider events.
- **IDP user provisioning from any OIDC IdP** (`template_user`; covers the #1 real-world pitfall: `.Role` vs `.IDPFields`).
- **Scheduled retention report via email** (`attach_event_report` / `{{.RetentionChecks}}`).
- **ICAP scan-on-upload** (`paths` with `{{.VirtualPath}}`).
- **Command hook** (`args: ["--user={{.Name}}", "--path={{.VirtualPath}}"]`).

## Red flags to watch for in user-provided templates

Before endorsing a user's template, scan for:

- `{{Name}}` / `{{Event}}` without a leading dot → **wrong, won't compile in `text/template`** even though some old documentation examples show this syntax.
- Access to `.Initiator.User.Email` without an enclosing `{{if .Initiator.User}}` guard → will panic-render empty with no error but produce confusing output when the initiator is an admin or `nil`.
- Use of `{{.Object.Username}}` directly → `.Object` is a lazy renderer, not a plain struct. Use `.Object.User.Username` / `.Object.Admin.Username` or `.Object.JSON`.
- Access to `{{index .IDPFields "some_claim"}}` where `some_claim` is not in the OIDC binding's `custom_fields` array → always `nil`. The role claim is in `.Role`, populated separately.
- Embedding multi-line or quoted content in JSON without `toJson` / `toJsonUnquoted` → broken payload.

See `references/pitfalls.md` for the full list with reproducible cases.

## Keeping content accurate

Every claim in this skill's reference files is verified against SFTPGo runtime behavior. When SFTPGo releases introduce a new template function, a new key in the template context, or a new action type, update the reference files and bump the `metadata.version` and `metadata.upstream.verified_against` in this manifest. A template that the skill vouches for must still compile and behave correctly.
