# SFTPGo — orientation guide

Entry point for the skill. Short overview of how SFTPGo is organized, with pointers into the public documentation for operational detail and into the sibling reference files for template / API depth.

The full API contract is in the OpenAPI spec:

- `https://sftpgo.com/assets/openapi.yaml` — always the **latest** publicly released version
- `openapi/openapi.yaml` inside the deployed installation — authoritative for **that** server (may be older)

The full user-facing documentation is published at `https://docs.sftpgo.com/enterprise/` and mirrored on GitHub at `https://github.com/sftpgo/docs` (branch `enterprise`). Raw markdown is fetchable at:

```text
https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md
```

Two pages are especially useful as fast indexes (dual-use, human + AI):

- `documentation-map.md` — task → doc file cheat sheet
- `glossary.md` — SFTPGo-specific terms disambiguated

For **installation, config-file editing, and environment-variable composition**, go to the sibling file `deployment.md` — the env-var naming rule and the `env.d/` recipes are there.

For **the fully-managed SaaS at sftpgo.com** (dedicated per-instance subdomain we assigned, no shell access, `bucket=default`+`region=default` in the config, pre-created `default` group), go to `saas.md`. Most SFTPGo content applies unchanged; what differs is mostly what to suggest (WebAdmin + REST API, assign users to the `default` group) and what to avoid (env vars, CLI commands, deployment recipes). When in doubt, ask the user *"Are you using the fully-managed SaaS from sftpgo.com, or are you running SFTPGo on your own server?"*

For **the Open Source edition or a third-party redistribution** (GitHub `drakkan/sftpgo`, Docker Hub `drakkan/sftpgo`, managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, unofficial Docker images, open-source cloud-marketplace listings), go to `editions.md`. Most of the skill targets Enterprise, so when a user is on OSS many of the ready-to-paste examples (ICAP, PGP, event reports, WOPI, email-authenticated shares, the full template engine with `toJson` et al.) do not apply. `editions.md` is the detection + "what Enterprise adds" + answer-phrasing reference. Flag Enterprise-only features plainly, once, with no sales pitch.

---

## 1. Mental model

SFTPGo has two distinct principal types:

- **Admins** manage the server (WebAdmin + REST API). They do not transfer files.
- **Users** sign in via SFTP, FTP/S, WebDAV, HTTPS, or the WebClient.

Everything else hangs off users: a home directory (on any supported storage backend), permissions, optional virtual folders (additional mounts), groups (reusable settings bundles), shares (public links), and automation via the Event Manager.

All configuration is stored in the **data provider** (SQLite / PostgreSQL / MySQL / CockroachDB / embedded Bolt). WebAdmin forms and REST API endpoints edit the same records.

---

## 2. REST API onramp

### 2.1 Obtain a token

```bash
ENDPOINT="https://sftpgo.example.com"
ADMIN_USER="admin"
ADMIN_PASSWORD="your_admin_password"

TOKEN=$(curl -s --anyauth -u "$ADMIN_USER:$ADMIN_PASSWORD" \
  "$ENDPOINT/api/v2/token" | jq -r .access_token)
```

Tokens from `/token` expire after a short window. For automation, create an **API key** and send it as `X-SFTPGO-API-KEY: <key>` — no refresh dance. See `rest-api.md` in the public docs.

### 2.2 Create the simplest working user

```bash
curl -X POST "$ENDPOINT/api/v2/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "initial-password",
    "status": 1,
    "home_dir": "/srv/sftpgo/data/alice",
    "permissions": {"/": ["*"]}
  }'
```

For a local-backend user that's all you need: `username`, `password`, `home_dir`, `permissions["/"]`.

### 2.3 Payload conventions the spec cannot fully express

- **Tagged unions**: `filesystem.provider` is an integer that selects which `*config` sub-object applies (e.g. `provider: 1` → `s3config`). Other sub-objects are ignored. The same pattern is used on Event Action configurations (`BaseEventActionOptions`): `type` picks the active sub-config. The enum descriptions in the OpenAPI spec document this mapping — but nothing enforces it in JSON Schema terms, so agents must remember it.
- **Secret envelope**: any field that holds a secret (passwords in filesystem configs, service account JSON, API keys for event actions, SMTP passwords, etc.) uses the shape `{"status": "Plain", "payload": "<value>"}` on write. On read the status becomes the encrypted-status of the active KMS (`Secretbox` for the default local KMS, otherwise `AES-256-GCM` / `AWS` / `GCP` / `VaultTransit` / `AzureKeyVault` / `OracleKeyVault`) with `payload` set to the ciphertext and `key` / `additional_data` blanked out. To **preserve** the current secret during an update, send back any non-Plain non-empty envelope — the simplest form is `{"status": "Secretbox"}` (use whatever encrypted status the `GET` showed). Sending `{"status": "Plain", "payload": "newvalue"}` replaces it; `{"status": "Plain", "payload": ""}` clears it. The User-level `password` scalar (top-level, not a secret envelope) is preserved by **omitting** it from the body.
- **Idempotency**: `POST` creates every time (no upsert). `PUT /users/{username}` is a full-object replace; omitted fields reset to default. Use `GET` → mutate → `PUT` to update safely.
- **Status codes**: `201 Created` with `Location` header on creation; `200 OK` on update; `400` for validation (message is the user-visible reason); `403` for permission; `409` for unique-key conflicts.

---

## 3. Concepts — what you need to know, where to dig

### 3.1 Users and permissions

Per-backend required fields. `home_dir` is mandatory in the database for every user, but the API auto-fills it when the payload omits it: it joins `data_provider.users_base_dir` (set to `/srv/sftpgo/data` by default on Linux APT/RPM packages, empty on the shipped `sftpgo.json` / Docker / Windows installer) with the username. So on packaged Linux installs `home_dir` is effectively optional in the payload; on Docker / Windows install it (or set `users_base_dir` once).

| Provider             | Integer | Required fields                                                | Auth (when explicit fields are omitted)                                                                                                                  |
| -------------------- | ------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Local                | 0       | *(none — files live under `home_dir`)*                         | n/a                                                                                                                                                      |
| S3                   | 1       | `s3config.bucket`, `region` (or embedded in `endpoint`)        | Empty `access_key` / `access_secret` → AWS SDK default chain (env vars, EC2 instance profile, ECS / EKS task role, IRSA). `role_arn` works on top.       |
| Google Cloud Storage | 2       | `gcsconfig.bucket`                                             | `automatic_credentials: 1` → Application Default Credentials (GCE/GKE metadata, Workload Identity, `GOOGLE_APPLICATION_CREDENTIALS`).                    |
| Azure Blob           | 3       | `azblobconfig.container`, `account_name` *(or `sas_url`)*      | Empty `account_key` and `sas_url` → `DefaultAzureCredential` (managed identity, env vars, workload identity, Azure CLI).                                 |
| Encrypted local      | 4       | `cryptconfig.passphrase`                                       | n/a                                                                                                                                                      |
| SFTP backend         | 5       | `sftpconfig.endpoint`, `username`, `password` or `private_key` | n/a — explicit only.                                                                                                                                     |
| HTTP filesystem      | 6       | `httpconfig.endpoint`                                          | Whatever the remote HTTP service requires.                                                                                                               |
| FTP backend          | 7       | `ftpconfig.endpoint`, `username`, `password`                   | n/a                                                                                                                                                      |

**Managed credentials.** On AWS / GCP / Azure deployments where the host already carries an identity (instance profile, GKE workload identity, AKS managed identity, IRSA, etc.), leave the credential fields empty — the SDKs pick the host identity up automatically. This is the recommended posture in cloud deployments: nothing sensitive in the SFTPGo data provider, IAM scopes managed at the platform level.

**GCS ≠ Google Drive.** SFTPGo integrates Google Cloud Storage (object store, service-account auth), not Drive. If a customer asks for Drive, the honest answer is "use GCS, or sync Drive to GCS externally".

Per-backend deep-dive (auth variants, IAM role, SAS, workload identity, etc.):

- [localfs.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/localfs.md)
- [s3.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/s3.md)
- [google-cloud-storage.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/google-cloud-storage.md)
- [azure-blob-storage.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/azure-blob-storage.md)
- [sftpfs.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/sftpfs.md)
- [dare.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/dare.md) (encrypted local)

**Permissions** are a map `virtual path → list of operation tokens`. Allowed tokens: `*` (all), `list`, `download`, `upload`, `overwrite`, `delete` / `delete_files` / `delete_dirs`, `rename` / `rename_files` / `rename_dirs`, `create_dirs`, `create_symlinks`, `chmod`, `chown`, `chtimes`, `copy`. The map **must** contain `/`; longest-prefix match wins. See the public docs for narrative, and the OpenAPI `User` schema for the exact field list.

### 3.2 Groups — condensed merge rules

Three group types. Only **one** can be Primary per user; any number of Secondary and Membership.

| Type       | JSON `type` | Count per user | Contributes                                                                                                                               |
| ---------- | ----------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Primary    | `1`         | At most one    | Base configuration: filesystem, home dir, quotas, bandwidth, password policy, session limits. Used only when the user's field is 0/empty. |
| Secondary  | `2`         | Any number     | Additive: extra virtual folders, per-path permissions, IP filters, share policies, WebClient permissions.                                 |
| Membership | `3`         | Any number     | **Nothing inherited.** Pure tag for event-rule conditions / admin scoping.                                                                |

Placeholders accepted in group values: `%username%`, `%role%`, `%custom1%..%customN%` (from `filters.custom_placeholders`). Apply in `home_dir`, `key_prefix`, `starting_directory`, virtual folder `virtual_path`, SFTP backend `username`.

Full narrative (what *each* field merges how, common patterns, gotchas): [groups.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/groups.md). Worked example: [tutorials/groups-example.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/tutorials/groups-example.md).

### 3.3 Virtual folders

A virtual folder is a mount point: a backend-agnostic storage location made available at a specific path inside the user's namespace. Used when the user's home is on storage A but they also need access to storage B.

**When you actually need a virtual folder**: cross-backend mounts (home on local, `/archive` on S3); shared space across many users with its own quota; a folder whose backend credentials are managed separately from the user's.

**When you do NOT need one**: carving up a single backend into per-user sub-paths (use `home_dir` or `key_prefix` placeholders instead); giving one user multiple subdirs of their own home (just create them); tagging "this folder contains project data" (use a naming convention or `description`).

Full detail: [virtual-folders.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/virtual-folders.md).

### 3.4 Shares

Public or password-protected links that expose a subset of the user's namespace to external recipients. Scopes: `1` read, `2` write, `3` read+write. Options include password, expiration, IP allow-list, email authentication (`options.auth_mode: 1`).

Tutorial + operational detail: [tutorials/shares.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/tutorials/shares.md).

### 3.5 Authentication

| Need                                      | Mechanism                                                                                                     |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Local passwords                           | `password` field; bcrypt / argon2id hashing                                                                   |
| SSH public keys                           | `public_keys` list                                                                                            |
| OIDC (Entra ID, Okta, Google, Keycloak, Auth0, any compliant IdP) | [oidc.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/oidc.md) + [Event Manager JIT](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/tutorials/eventmanager-idp.md) |
| LDAP / Active Directory                   | [LDAP plugin](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/plugins/ldap-auth.md)             |
| 2FA                                       | [tutorials/two-factor-authentication.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/tutorials/two-factor-authentication.md) |
| API keys (automation)                     | REST API `/apikeys` — send as `X-SFTPGO-API-KEY`                                                              |
| Custom auth service (replace built-in)    | [external-auth.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/external-auth.md)            |
| Modify user on login                      | [dynamic-user-mod.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/dynamic-user-mod.md)      |

SFTPGo does not implement SAML — only OIDC.

**Prefer OIDC + Event Manager over pre-login hooks** for modern deployments. The IdP event + `template_user` / `template_admin` actions cover provisioning, role mapping, and group assignment without running a custom hook process.

### 3.6 Event Manager

Rules (`when`) + Actions (`what`). A rule has a single trigger type (filesystem / provider / schedule / ip-blocked / certificate / on-demand / IdP-login) and an ordered list of actions. Each action can use Go `text/template` strings to reference event data.

This is the deepest area of the skill — see the dedicated reference files:

- `trigger-types.md` — each trigger and which template fields it populates
- `action-types.md` — each action type and the sub-config it expects
- `template-context.md` — every key exposed to templates
- `template-functions.md` — the registered function map (`toJson`, `humanizeBytes`, …)
- `examples.md` — ready-to-paste templates (chat webhooks, OIDC JIT, retention reports, ICAP, staged uploads, command hooks)
- `pitfalls.md` — the common mistakes

Public docs entry points:

- [eventmanager.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/eventmanager.md) (overview)
- [placeholders.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/placeholders.md)
- [execute-before-file-publish.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/execute-before-file-publish.md)
- [event-report.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/event-report.md)
- [filesystem-actions.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/filesystem-actions.md)
- Tutorials: [eventmanager-*.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/tutorials/) (notifications, IdP provisioning, retention, copy, ICAP, PGP, recycle-bin, auto-dirs, backup, folders)

---

## 4. Working two interfaces at once

The WebAdmin and the REST API are equivalent — the forms submit the same JSON shape the API accepts. The skill's examples show both forms deliberately (raw string to paste into a WebAdmin field + full JSON payload). Pick the form that matches what the user asked for.

If the user asks *"how do I configure X in the WebAdmin"* and X involves a template field (Event Action Body, OIDC `template_user`, etc.), you need the **raw string** form — the form does not accept wrapping JSON. Look at `examples.md`.

If the user asks *"how do I automate X"* — produce the full POST payload.

---

## 5. Troubleshooting shortlist

| Symptom                                                      | Likely cause                                                                                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| Login rejected                                               | `status: 0`, `expiration_date` in the past, `filters.denied_login_methods` excludes the protocol, or IP not in `filters.allowed_ip`  |
| "Permission denied" on an operation                          | `permissions` for the longest matching virtual path does not include the needed token (remember: `/` is the fallback, sub-paths win) |
| Local user logs in OK but first upload / `mkdir` fails with "permission denied" | OS-level: the service runs as the `sftpgo` system user (Linux APT/YUM) or `LOCAL SYSTEM` (Windows). A custom `home_dir` outside `/srv/sftpgo` (Linux) / the installer default (Windows) needs to be writable by that account. On Linux: `chown -R sftpgo:sftpgo <home>` and `chmod +x` the parents. |
| File visible in `ls` but cannot be opened                    | `filters.file_patterns` with `deny_policy: 0` for the path                                                                           |
| File not visible at all                                      | `filters.file_patterns` with `deny_policy: 1` (hide)                                                                                 |
| Event rule doesn't fire                                      | `options.names` / `options.role_names` / `options.group_names` conditions exclude the subject, or the trigger type doesn't match     |
| Template renders empty string                                | Referenced key is not populated for that trigger (see the "Placeholder picker" table in SKILL.md)                                    |
| Share returns 404                                            | Share expired, IP restriction, path no longer exists, or owning user disabled                                                        |
| Secret decryption error after restore                        | KMS key changed / missing; secrets are encrypted with the active KMS at write time                                                   |
| REST API returns `400 validation_error`                      | Message in the body is the exact reason — read it first, the skill's pitfalls file has the common ones                               |
| Cloud-backend upload fails with "unable to open pipe" / "read-only file system" / "no space left" | The host has no writable temp dir for per-transfer pipes. Fix: `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1`. See `deployment.md §5.11`. Default on marketplace offerings. |

### Reading the logs

SFTPGo writes **structured JSON**, one object per line. Every record has a `sender` field that identifies the category — use it to filter.

**Where the logs live**

| Install                                          | Location                                                                                     |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Debian / RPM (systemd)                           | `/srv/sftpgo/logs/sftpgo.log` + `journalctl -u sftpgo`                                        |
| Docker / Kubernetes                              | Container stdout — `docker logs <name>` / `kubectl logs <pod>`                               |
| Windows (installer service)                      | `C:\ProgramData\SFTPGo Enterprise\logs\sftpgo.log` (the application log; only service-lifecycle events from the Windows Service Control Manager land in the System log)  |
| Anywhere with `--log-file-path` empty            | Standard error                                                                               |

**`sender` values worth knowing**

| `sender`                     | What it records                                                         | Typical use                                                 |
| ---------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------- |
| `login`                      | Successful logins (with protocol, method, client string).               | "Did this user actually sign in? When? From where?"         |
| `connection_failed`          | Failed connections: wrong password, unknown user, client abort, timeout. `error` field carries the reason. | The #1 source for login-denied troubleshooting.             |
| `Upload` / `Download`        | Transfer completion with bytes, elapsed ms, file path, connection id.   | Bandwidth / usage analysis, correlating with Event Manager. |
| `Rename` / `Mkdir` / `Rmdir` / `Remove` / `Chmod` / `Chown` / `Chtimes` / `Copy` / `Truncate` / `SSHCommand` | Filesystem commands.      | "Who deleted this file?" / audit for specific operations.   |
| `httpd`                      | REST API and WebAdmin / WebClient requests with method, URI, status.    | 4xx / 5xx investigation; slow-endpoint hunting.             |

**Useful jq one-liners**

```shell
# All failed login attempts in the last 1000 lines
tail -n 1000 /srv/sftpgo/logs/sftpgo.log \
  | jq -c 'select(.sender=="connection_failed") | {time, username, client_ip, protocol, error}'

# Successful logins for a specific user
journalctl -u sftpgo --since "1 hour ago" -o cat \
  | jq -c 'select(.sender=="login" and .username=="alice")'

# Uploads > 100 MB in the last day
grep '"sender":"Upload"' /srv/sftpgo/logs/sftpgo.log \
  | jq -c 'select(.size_bytes > 104857600) | {time, username, file_path, size_bytes, elapsed_ms}'

# REST API 5xx responses
grep '"sender":"httpd"' /srv/sftpgo/logs/sftpgo.log \
  | jq -c 'select(.resp_status >= 500) | {time, method, uri, resp_status, remote_addr}'
```

On Docker / Kubernetes the logs are on stdout, so the JSON is directly consumable by any log aggregator (Loki, CloudWatch, Stackdriver, Datadog, Splunk…) without a sidecar. For the full schema of each record type see [logs.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/logs.md) in the public docs.

---

## 6. How to use this skill efficiently

1. For a question about **concepts or operations**, fetch the relevant public doc file (the raw-GitHub URL template above). The frontmatter `description:` lets you pick without reading the whole file.
2. For a question about **Event Action templates**, use the dedicated reference files in this directory — that's where the skill adds value the public docs don't duplicate.
3. For a question about **exact field names / enum values / required fields**, consult the OpenAPI spec directly.
4. Before endorsing a user-provided template, run it through the red-flag checklist in `pitfalls.md`.
