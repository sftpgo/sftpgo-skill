# SFTPGo editions — Open Source vs Enterprise, and what to do when you meet each one

SFTPGo ships in two editions and through several distribution channels. Most questions an AI agent gets about SFTPGo apply to both editions, but a non-trivial set of features exist only in Enterprise — suggesting one of those to a user on the Open Source edition wastes everyone's time and produces "this doesn't work" back-and-forth. This file explains how to identify the edition, what differs, and how to phrase answers accordingly.

**This skill is not a sales funnel.** When a requested feature is not available on the Open Source edition, state the limitation plainly, offer an OSS-compatible alternative when one exists, and stop. No repeated "upgrade to Enterprise" pitches.

## 1. The editions

- **SFTPGo Open Source** — AGPLv3, source at `github.com/drakkan/sftpgo`, releases on GitHub and Docker Hub (`drakkan/sftpgo`). Includes the full protocol stack, all storage backends, the WebAdmin and WebClient, the REST API, OIDC (basic), and a basic Event Manager. Docs live at `https://docs.sftpgo.com/latest/` (mirror at `https://github.com/sftpgo/docs` branch `main`, raw markdown `https://raw.githubusercontent.com/sftpgo/docs/main/docs/<file>.md`).
- **SFTPGo Enterprise** — proprietary license, binary distributed from `download.sftpgo.com` (APT / YUM / Windows installer) and `registry.sftpgo.com/sftpgo/sftpgo` (Docker). Includes everything from OSS plus the additions listed in §3. Docs live at `https://docs.sftpgo.com/enterprise/` (raw markdown `https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md`).

Both editions are actively maintained. New features are developed for Enterprise first and may be backported to OSS over time.

## 2. Distribution channels

| Channel                                                       | Edition                                            | Notes                                                                     |
| ------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------- |
| `github.com/drakkan/sftpgo` releases                          | **OSS**                                            | Canonical self-host source. Binaries for Linux / macOS / Windows / BSD.   |
| Docker Hub `drakkan/sftpgo`                                   | **OSS**                                            | Free image.                                                               |
| `download.sftpgo.com` APT / YUM                               | **Enterprise**                                     | Requires license (trial or paid). Installs package `sftpgo`.              |
| Windows installer from `download.sftpgo.com`                  | **Enterprise**                                     | Installs as "SFTPGo Enterprise" service.                                  |
| `registry.sftpgo.com/sftpgo/sftpgo` Docker registry           | **Enterprise**                                     | Requires license; `-plugins`, `-distroless`, `-distroless-plugins` tags.  |
| Helm chart `ghcr.io/sftpgo/helm-charts/sftpgo`                | **Edition-agnostic** (default image points at OSS) | Default `image.repository` is `ghcr.io/drakkan/sftpgo` (OSS). Override it (typically to `registry.sftpgo.com/sftpgo/sftpgo`) to deploy Enterprise. See `deployment.md`. |
| AWS / Azure / GCP marketplace — **Enterprise** listings       | **Enterprise**                                     | Pre-configured VM or container. License bundled with the subscription.   |
| AWS / Azure / GCP marketplace — **Open Source** listings      | **OSS**                                            | Same VM shape as above but the OSS binary. Explicitly labelled.           |
| `sftpgo.com` fully-managed SaaS                               | **Enterprise**                                     | We host. See `saas.md`.                                                   |
| Managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, third-party Docker images, unofficial Kubernetes manifests | **OSS** (unless the listing explicitly says "SFTPGo Enterprise") | Third-party repackagings of the OSS code. No license needed; no Enterprise features. |

The line to draw: if the distribution source is `sftpgo.com`, `download.sftpgo.com`, `registry.sftpgo.com`, or a cloud-marketplace listing explicitly marked "SFTPGo Enterprise", it is Enterprise. The Helm chart is edition-agnostic — look at the configured `image.repository` to tell. Anything else is OSS.

### A note on third-party distributions

Any distribution that packages the OSS binary from outside the official channels (managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, third-party Docker images, unofficial Kubernetes manifests) is **not endorsed by the SFTPGo project**. Users on those distributions typically have:

- Access to the OSS feature set (no Enterprise features).
- Whatever support the distribution itself chooses to provide.
- The standard OSS channels — issues and discussions on `github.com/drakkan/sftpgo` — for reporting reproducible bugs in the upstream binary.
- The option to move to the official Enterprise edition if they need a commercial support channel.

When an OSS user's question is actually about how the wrapper configured their install (config file location moved, env vars injected differently, custom image layout), point them at the wrapper's own documentation — the SFTPGo project cannot speak for those layouts.

## 3. What Enterprise adds over Open Source

The **canonical and most up-to-date list** is the "What Enterprise adds" table on the Enterprise docs landing page: [index.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/index.md) (section `## Open Source and Enterprise editions`). Fetch that when you need to confirm a specific feature's edition; the summary below is intentionally short and may lag behind.

**Event Manager** (this is where the skill spends most of its pages)

- **OSS**: basic automation — you can write rules that fire on events and invoke simple actions. Action template fields (HTTP body, email subject/body, command args, …) are processed through a flat **`strings.Replacer`** pass over a fixed list of dot-prefixed placeholders such as `{{.Name}}`, `{{.VirtualPath}}`, `{{.IDPField<claim>}}` and the timestamp variants. There is no conditional, no range, no pipe, no FuncMap, no nested accessor: anything outside the placeholder list survives verbatim in the rendered output. See [`reference.md`](../../../reference.md) §3.1 for the full OSS placeholder list.
- **Enterprise adds**: the full Go `text/template` engine with helper functions (`toJson`, `humanizeBytes`, `pathJoin`, …), conditions (`{{if}}`), loops (`{{range}}`), structured accessors (`{{.Object.User.Email}}`, `{{index .IDPFields "claim"}}`); virtual-folder integration for cross-backend operations; the **ICAP** action (antivirus / DLP), **IMAP** action (email ingestion), **event reports**, **PGP** encryption / decryption; **Execute Before File Publish** for staged upload processing; enhanced **copy** action with source disposition, glob patterns, retries; **data retention with archival**.
- Practical implication: almost every example in [`examples.md`](examples.md), every template function in [`template-functions.md`](template-functions.md), and most of the action types in [`action-types.md`](action-types.md) assume Enterprise. A user on OSS asking to "send a webhook body built with `toJson`" or "scan uploads with ICAP" is asking for Enterprise-only features. A `{{if .Initiator.User}}` guard pasted on OSS will appear in the rendered webhook body as the literal string `{{if .Initiator.User}}` — the engine does not error.

**OpenID Connect**

- **OSS**: basic OIDC sign-in.
- **Enterprise adds**: configurable role mapping, PKCE without client secret, session control (`max_age`, `prompt`), Azure B2C compatibility, customizable login labels.

**WebClient**

- **Enterprise adds**: WOPI / Collabora document editing and real-time collaboration, TUS resumable uploads. WOPI / Collabora is license-gated; **TUS works in limited mode too** (no license needed). TUS is opt-in per HTTPD binding — set `upload_chunk_size > 0` (env: `SFTPGO_HTTPD__BINDINGS__0__UPLOAD_CHUNK_SIZE=10`) to enable TUS endpoints for both the WebClient and the REST API on that binding; `0` disables them. Has nothing to do with memory pipes.

**Sharing**

- **Enterprise adds**: email-based authentication on shares, group-based delegation and governance policies, path and scope restrictions.

**Storage**

- **Enterprise adds**: optimized cloud backend performance for small files, in-memory transfers (memory pipes — see `deployment.md`), GCS Hierarchical Namespace, SFTP backend SOCKS proxy, FTP as storage backend.

**Administration**

- **Enterprise adds**: clustering with near real-time propagation; extended WebAdmin UI for OIDC, LDAP, Geo-IP, TLS certificates, email templates, SSH host keys; API key management from the UI.

**Plugins**

- OSS supports the plugin interface and a set of open-source plugins under `github.com/sftpgo` (pubsub, geoipfilter, kms, auth, **eventstore**, **eventsearcher**, and others). The Enterprise edition ships its own builds of `eventstore` and `eventsearcher` with a different codebase and more features. The WebAdmin audit-logs browsing UI is present in both editions — what differs is the depth of the data captured and queryable, driven by the plugin build in use. KMS providers (the `kms` plugin and the built-in providers) are currently the same in both editions.
- Audit-logs is delivered as **two cooperating plugins** (`eventstore` for write/persist, `eventsearcher` for query); both must be loaded for the audit-logs UI to function. On marketplace tiers that cap the number of active plugins (Starter caps it at 2), audit-logs alone fills the quota — there is no spare slot for a third plugin without dropping audit-logs or moving up a tier.

## 4. How to detect the edition

Direct signals — treat as confirmed:

- User mentions installing from `github.com/drakkan/sftpgo`, from `drakkan/sftpgo` Docker Hub, from a managed-hosting provider, a NAS or home-lab app store, a community Linux-package archive, or any other repackaging not sourced from the official channels → **OSS**.
- User mentions they "don't have a license", "are on the free version", "started the trial" (trial is Enterprise with a time-limited license), or their WebAdmin shows *"License inactive, limited functionality"* — this is the Enterprise binary in **limited mode** (2 concurrent transfers, plugins disabled, local filesystem only).
- User's WebAdmin banner / login page says "SFTPGo" without "Enterprise" → likely OSS, but branding customization can hide this.
- Systemd unit name is `sftpgo` and the binary was installed from `apt install sftpgo` *without* configuring the `download.sftpgo.com` repo → distro package, OSS.
- Docker image is `drakkan/sftpgo:...` → **OSS**; `registry.sftpgo.com/sftpgo/sftpgo:...` → **Enterprise**.
- User references the docs at `docs.sftpgo.com/latest/` → OSS docs; `docs.sftpgo.com/enterprise/` → Enterprise docs.

Indirect signals — likely but verify:

- User says the feature they want "doesn't exist in the WebAdmin". Many Enterprise-only UI sections (LDAP config, Geo-IP config, event report action type, Collabora) simply don't appear on OSS.
- User gets a "this feature requires a valid license" error or "plugins are disabled" → Enterprise binary without a valid license (limited mode).
- User asks about a template function like `toJson` and their Event Manager save rejects it → OSS basic Event Manager.

**When uncertain, ask.** A single question avoids back-and-forth: *"Are you on the Enterprise edition or the Open Source edition?"* (or "Did you install from `sftpgo.com` / the APT-YUM repo, or from `github.com/drakkan/sftpgo` / Docker Hub / a third-party package?").

## 5. How to answer once the edition is known

### When the user is on Open Source

- **Prefer OSS-compatible approaches first.** For example, if a user wants upload notifications, an HTTP action with a handwritten JSON body works on OSS (the basic Event Manager does run actions); what does not work is the full template engine with `{{.Name | toJson}}` pipes and the ready-made examples from `examples.md`. Explain the simpler path and cite [OSS docs](https://raw.githubusercontent.com/sftpgo/docs/main/docs/) if available for the feature.
- **Flag Enterprise-only features plainly, once.** Sample phrasing: *"This relies on \<feature\>, which is part of the Enterprise edition — on Open Source the closest equivalent is \<alternative\>."* Do not repeat the note; do not follow it with promotional text.
- **Do not fabricate OSS workarounds** for features that genuinely are Enterprise-only (ICAP, PGP, event reports, staged upload actions, WOPI, clustering, email-authenticated shares, LDAP, most UI configuration pages). Just say the feature is not available on OSS.
- **Audit logs on OSS.** OSS users can install the open-source `eventstore` and `eventsearch` plugins from `github.com/sftpgo` to persist and query audit events; the WebAdmin audit-logs browsing UI works against them on OSS too. The Enterprise builds of the same plugins are richer (different codebase, more features / fields captured), so an OSS user will see a reduced subset of what an Enterprise user sees through the same UI. For quick inspection without the plugins, the JSON log stream is always available — see [`logs.md`](https://raw.githubusercontent.com/sftpgo/docs/main/docs/logs.md).
- **Deployment topics** (install, config file, env vars, Docker, Kubernetes, Helm): most of [`deployment.md`](deployment.md) applies to OSS too, since the binary accepts the same configuration shape. The official Helm chart at `oci://ghcr.io/sftpgo/helm-charts/sftpgo` works on OSS by default — its default `image.repository` is `ghcr.io/drakkan/sftpgo`. Enterprise-specific values (license secret, private registry pull secret) just don't apply on OSS.
- **Third-party distributions** often wrap the OSS binary with their own config layout. The binary, and therefore the REST API, the WebAdmin form fields, and the `SFTPGO_…` env-var names, still work the same — but the wrapper may restrict which env vars the user can set or hide the config file path. When a recipe from `deployment.md` doesn't apply cleanly, suggest the user consult the wrapper's own documentation for the config injection point.

### When the user is on Enterprise

Assume the full skill applies. Default to the `enterprise` docs branch when fetching a reference.

### When the user is on the SaaS

See [`saas.md`](saas.md). The SaaS is Enterprise under the hood, with additional preconfiguration and restrictions (no shell access, pre-created `default` group, managed restart flow for OIDC/LDAP).

## 6. Quick reference — where to fetch the docs

| Edition     | Live site URL                               | Raw GitHub URL pattern for `<file>.md`                                           |
| ----------- | ------------------------------------------- | -------------------------------------------------------------------------------- |
| OSS         | `https://docs.sftpgo.com/latest/`           | `https://raw.githubusercontent.com/sftpgo/docs/main/docs/<file>.md`              |
| Enterprise  | `https://docs.sftpgo.com/enterprise/`       | `https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md`        |

Prefer the live site for human reading (with rendered images), the raw URL for programmatic fetch inside the skill workflow.
