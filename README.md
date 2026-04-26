# sftpgo-skill

Reference skill for [SFTPGo](https://sftpgo.com) — the managed file transfer platform. Helps AI coding agents and humans drive SFTPGo correctly across the whole journey: installing and deploying the server, configuring it via the REST API or the WebAdmin, and automating file-transfer workflows with the Event Manager.

## What this skill helps you do

Following the order a real operator hits:

- **Install and deploy SFTPGo.** Pick the right distribution channel (self-hosted APT / YUM / Windows installer / Docker image / Helm chart, cloud-marketplace VM or container, or the fully-managed SaaS at `sftpgo.com`), lay out the config file, compose the `SFTPGO_…` environment variables correctly (prefix, `__` separator, `__N__` for list-of-struct — the bits the spec can't enforce and that silently no-op if wrong), and turn on the standard production knobs: HTTPS / ACME, an external data provider, the Defender, telemetry and metrics, the HAProxy PROXY protocol, licensing. Includes Kubernetes specifics (Service for raw TCP, PVC, shared host keys, memory pipes) and SaaS-specific conventions (`bucket=default`, `region=default`, pre-created `default` group).
- **Configure SFTPGo day-to-day** — REST API and WebAdmin. The WebAdmin form fields map 1-to-1 to the REST API JSON payload, so the same knowledge covers both. The skill focuses on the conventions the OpenAPI spec alone can't express: tagged unions (`provider` / `type` integer → which sub-config is active), the secret envelope (`{status, payload}` with the `[**redacted**]` sentinel), required-field matrices, cross-field dependencies, idempotency and PUT-replacement semantics. Examples are shown in both forms: raw string (paste into a WebAdmin field) and full JSON (POST to the REST API).
- **Automate with the Event Manager.** The deepest area of the skill. Full coverage of the trigger × action matrix, the Go template context, the registered function map, per-trigger placeholder availability, and ready-to-paste examples (upload notifications, chat webhooks for Slack / Teams / Discord / Google Chat / Mattermost, OIDC JIT user provisioning, retention reports, ICAP scanning, staged-upload actions, command hooks, PGP encryption).

Cross-cutting: the skill knows the difference between the **Open Source** and **Enterprise** editions and between the self-hosted, cloud-marketplace, and fully-managed SaaS distributions — so it won't suggest an Enterprise-only action to an OSS user, or a shell command to a SaaS customer.

When in doubt about a specific field, an AI loading this skill should fall back to the authoritative OpenAPI spec and re-check.

## Skill contents

A single skill, [`sftpgo`](skills/sftpgo/), organized under `references/` by topic. It is intentionally generic: new topics go into `skills/sftpgo/references/` alongside the existing files, not into a separate skill directory. The files follow the install → configure → automate journey.

### Start here

- `references/user-guide.md` — short orientation: mental model, REST API onramp, concepts cheat sheet (users / groups / virtual folders / shares / authentication / Event Manager), troubleshooting shortlist, and a log-reading micro-guide. Points into the public documentation (raw GitHub URLs) for operational depth and into the sibling reference files for specifics.

### Install, deploy, configure

- `references/deployment.md` — for system administrators: installation decision tree, config file format, **`SFTPGO_…` environment-variable naming rule** (the part an AI must get exactly right), where env vars are read from (systemd `EnvironmentFile`, `env.d/`, Docker, Kubernetes), ready-to-paste production recipes (Postgres data provider, HTTPS / ACME, Defender, telemetry / pprof, HAProxy PROXY, license activation, memory pipes), the Kubernetes shape (Service for TCP, PVC, host keys, license secret), and the sysadmin pitfall list including the permission-denied and cloud-temp-file traps.
- `references/saas.md` — for users on the fully-managed SaaS at `sftpgo.com`: how to detect the SaaS environment (`bucket=default`+`region=default`, no shell access), what's preconfigured (integrated S3, `default` group with `users/%username%/`, audit logs, Defender, recycle bin, managed backups), tier-gated features (Geo-IP, HIPAA, LDAP, OIDC, PGP), the custom-domain CNAME flow, which changes trigger an automated restart, and what **not** to suggest (CLI commands, env vars, deployment recipes).
- `references/editions.md` — Open Source vs Enterprise awareness: the distribution channels (official APT-YUM / Windows / Docker registry / Helm / cloud marketplaces vs `github.com/drakkan/sftpgo`, Docker Hub `drakkan/sftpgo`, managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, unofficial Docker images and Kubernetes manifests), the canonical list of what Enterprise adds over OSS (richer Event Manager, full OIDC, WOPI, email-auth shares, cloud-backend optimisations, clustering, UI-based advanced config, Enterprise builds of `eventstore` / `eventsearch`), how to detect which the user is running, and how to phrase answers neutrally when the user is on OSS — no sales pitch.

### Automate — Event Manager

- `references/trigger-types.md` — each Event Rule trigger type and which fields of the template context it populates.
- `references/action-types.md` — each Event Action type and the shape of its sub-configuration.
- `references/template-context.md` — the full set of keys available to Go templates.
- `references/template-functions.md` — the registered template function map.
- `references/examples.md` — ready-to-paste templates for upload notifications, chat webhooks (Slack, Microsoft Teams, Discord, Google Chat, Mattermost), OIDC JIT user provisioning, scheduled retention reports, ICAP scanning, staged-upload actions, command hooks. Every example is shown both as a raw string (to paste into a WebAdmin form) and as a full REST API payload.
- `references/pitfalls.md` — common mistakes authoring templates and payloads, grouped by category.

## The authoritative sources

This repo does not duplicate the OpenAPI spec or the SFTPGo user-facing documentation. It **indexes** them.

**OpenAPI spec** — field names, enum values, required fields:

- `https://sftpgo.com/assets/openapi.yaml` — updated on every public release, so it always describes the **latest** publicly released SFTPGo version.
- The `openapi/openapi.yaml` file shipped with the installation — authoritative for the **currently deployed** server, which may be older than the latest release and therefore missing some fields or endpoints present in the public copy.

Use the public URL to reason about the product's current API surface; use the installed file when the user is integrating against a specific deployed version and needs to confirm that a given field is actually available on that server.

**Public documentation** — operational how-to for every SFTPGo feature (install, storage backends, groups, shares, authentication, hooks, Event Manager tutorials, metrics, etc.). Published at `https://docs.sftpgo.com/enterprise/`; GitHub source at `https://github.com/sftpgo/docs` (branch `enterprise`). Raw markdown is fetchable at:

```text
https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md
```

Each indexed file carries a `description:` frontmatter line — AI agents should read that first to decide whether to fetch the full file.

Two pages are especially useful as fast task-oriented indexes:

- `documentation-map.md` — task → file cheat sheet
- `glossary.md` — disambiguation of SFTPGo-specific terms

**This skill** (everything under `skills/sftpgo/`) covers what the OpenAPI spec and the public docs cannot easily express: enum↔sub-config mappings, per-trigger placeholder availability, the template context, the template function map, ready-to-paste Event Action templates, and common pitfalls.

## Installing / using with different AI agents

The skill is **AI-agnostic**. Pick the integration that fits your tool.

### Claude Code (Anthropic)

This repository is a Claude Code plugin *and* a single-plugin marketplace. The recommended install is via the marketplace commands — one-time registration, then one command to install and `/plugin update` afterwards for upgrades:

```text
/plugin marketplace add sftpgo/sftpgo-skill
/plugin install sftpgo-skill
```

If you prefer to clone the repo manually (for example to experiment with local edits), both user-level and project-level plugin locations are supported:

```bash
# User-level: available in every Claude Code project on your machine
mkdir -p ~/.claude/plugins
git clone https://github.com/sftpgo/sftpgo-skill ~/.claude/plugins/sftpgo-skill

# Project-level: scoped to a single project
cd your-project
mkdir -p .claude/plugins
git clone https://github.com/sftpgo/sftpgo-skill .claude/plugins/sftpgo-skill
```

The skill manifest is at `skills/sftpgo/SKILL.md`. Claude Code auto-discovers skills inside the `skills/` subdirectory of a plugin, so no further registration is needed after install or clone.

Trigger phrases that should activate the skill: any question about the SFTPGo REST API, WebAdmin configuration, Event Actions, Event Rules, share/folder/user creation, IDP integration, or Go-template authoring for SFTPGo.

### OpenAI Codex CLI

[Codex CLI](https://github.com/openai/codex) is OpenAI's open-source terminal agent. It honours the `AGENTS.md` convention (a cross-agent standard adopted by Codex and a growing list of other agentic tools), read from the project root or from `~/.codex/AGENTS.md`.

Drop a pointer into either location:

```markdown
## SFTPGo reference

For any question about SFTPGo REST API payloads, WebAdmin configuration, or
Event Action templates, consult:
- https://raw.githubusercontent.com/sftpgo/sftpgo-skill/main/reference.md
- https://sftpgo.com/assets/openapi.yaml  (authoritative field contract)
```

For deeper coverage, clone this repository alongside your project and reference specific files under `skills/sftpgo/references/` from `AGENTS.md` — for example `deployment.md` for sysadmin work, `examples.md` + `template-context.md` + `template-functions.md` for Event Action templates.

**ChatGPT Agent mode / ChatGPT Codex (web app)** — point the agent at this repository and ask it to read `reference.md` first, then answer your question.

### Cursor / Windsurf

Cursor reads rules from `.cursorrules` or `.cursor/rules/*.md`. Drop a pointer into your project rules:

```bash
mkdir -p .cursor/rules
curl -L https://raw.githubusercontent.com/sftpgo/sftpgo-skill/main/reference.md \
  -o .cursor/rules/sftpgo.md
```

The file `reference.md` in this repo is a single self-contained document suitable for being loaded as an IDE rule.

### GitHub Copilot

Add a pointer in `.github/copilot-instructions.md` (or in your personal Copilot settings):

```markdown
For SFTPGo integrations, follow the reference at
https://github.com/sftpgo/sftpgo-skill/blob/main/reference.md
and load https://sftpgo.com/assets/openapi.yaml as the authoritative API contract.
```

### Gemini Code Assist / Gemini CLI

Both Gemini Code Assist and the Gemini CLI support custom knowledge sources. Clone this repo locally (or keep it as a git submodule of your project) and point Gemini at `reference.md` through whichever knowledge-source mechanism your Gemini version exposes — refer to the Gemini documentation for the exact configuration format, which evolves across releases.

### ChatGPT / generic LLMs (paste-as-context)

`reference.md` is a single-file, self-contained reference. Paste it into the system prompt (or attach it) before asking the model to produce SFTPGo payloads, configurations, or templates. Pair it with `https://sftpgo.com/assets/openapi.yaml` for full API coverage.

A Custom GPT / Assistant can use the contents of `reference.md` as its system prompt.

## Using the skill with browser-based chatbots

Most of the integrations above assume an **agentic** AI that has access to your filesystem (Claude Code, Cursor, Copilot in an IDE, Gemini CLI, etc.). If you are instead chatting with a model through a web UI — `claude.ai`, `chat.openai.com`, `gemini.google.com`, `chat.mistral.ai`, Microsoft Copilot — there is no auto-discovery: you bring the context to the model. There are three practical approaches, in order of convenience.

### 1. One-off: paste `reference.md` at the start of the conversation

This is exactly what `reference.md` is for. Copy the file contents, paste it as the first message together with a short preamble, then ask your question:

> *"I'm asking questions about SFTPGo. Use the text below as the authoritative reference; if something isn't covered, tell me and we'll look it up together.*
> *`<paste reference.md>`*
> *First question: …"*

`reference.md` is ~25-30K tokens — comfortable on every modern web chatbot's context window (Claude Opus / Sonnet, GPT-4/5, Gemini 2.5, Mistral Large, etc.).

### 2. Targeted: attach the specific reference files you need

Both `claude.ai` and `chat.openai.com` accept file uploads. For a sysadmin task you can attach just `skills/sftpgo/references/deployment.md`; for Event Action templates attach `action-types.md` + `template-context.md` + `template-functions.md` + `examples.md` + `pitfalls.md`. You use less context and get more focused answers than when pasting the monolithic `reference.md`.

### 3. Persistent: Custom GPT / Claude Project / Gemini Gem

This is the best option when you will ask many SFTPGo questions over time — for a sysadmin team, set it up once and reuse forever.

- **ChatGPT** → *Create a GPT* (or *Projects*). Upload `reference.md` (and optionally individual reference files from `skills/sftpgo/references/`) as knowledge files.
- **Claude.ai** → *Projects* (Project Knowledge). Same idea — attach the files once, they are available in every chat inside the project.
- **Gemini** → *Gems* (in Gemini app and Google AI Studio). Same setup.
- **Mistral Le Chat, Microsoft Copilot, Cohere Coral**: most other chatbots with a "custom assistant" feature work the same way — system prompt + attached files.

A starting system prompt:

> *SFTPGo assistant. Before answering, (1) check whether the user is on the fully-managed SaaS at sftpgo.com, a cloud-marketplace VM, a self-hosted Enterprise install, or the Open Source edition — ask if unclear. (2) Use the attached reference files as ground truth. (3) For field-level API details, consult the OpenAPI spec at `https://sftpgo.com/assets/openapi.yaml`. (4) For operational how-to, cite `https://docs.sftpgo.com/enterprise/`. (5) When the user is on Open Source and a requested feature is Enterprise-only, state it once, neutrally, and provide an OSS-compatible alternative if one exists — no sales pitch.*

### Notes for browser chatbot users

- The **SFTPGo OpenAPI spec** is a single YAML file at `https://sftpgo.com/assets/openapi.yaml`. Attaching it to a Custom GPT / Project pins the field-level contract and keeps your answers accurate even when a new SFTPGo release ships.
- The **public documentation** is indexed by search engines: any chatbot with web-browsing enabled (ChatGPT *Search*, Claude *Web*, Gemini, Le Chat) can reach `https://docs.sftpgo.com/enterprise/` directly. If the chatbot has no browsing, it can still fetch pages that you paste or attach explicitly.
- **Model size matters.** Entry-level or quota-capped models (e.g. free tiers, `-mini` / `-flash` variants) have smaller context windows — prefer targeted file attachment (option 2) or a persistent Custom GPT (option 3) so the knowledge is loaded on demand rather than pasted in every turn.

## Versioning & upstream sync

The reference data is verified against SFTPGo runtime behavior. When the template context, the registered template function map, or the action-type enum change in a future SFTPGo release, the content here is updated and tagged.

| This repo tag | SFTPGo version verified against |
| ------------- | ------------------------------- |
| `v0.5.0`      | SFTPGo `v2.7.x`                 |

Open an issue if you find a drift — template or payload examples that used to validate but now fail are the main signal.

## Contributing

- Keep reference files **authoritative, not opinionated**.
- Keep examples **copy-pasteable** — show both the raw template (to paste in the WebAdmin) and the wrapping JSON payload (for the REST API).
- Never use customer names, role claim values, group names, bucket names, or any other detail derived from a real customer configuration. Always rewrite to generic placeholders.
- Add pitfalls only when they have caused real issues. Speculation bloats the context window without paying for itself.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
