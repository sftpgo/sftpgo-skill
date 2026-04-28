# SFTPGo — Single-File AI Reference

**Source of truth**: this file. Self-contained; paste it into any AI's context (ChatGPT, Gemini, Cursor, Copilot, Claude, …) before asking for help on SFTPGo REST API payloads, WebAdmin configuration, server deployment, environment variables, or Event Action templates. Verified against SFTPGo v2.7.x.

**Scope covered**

1. Where the authoritative sources live (OpenAPI spec, public docs, GitHub).
2. REST API payload conventions the OpenAPI spec cannot fully express.
3. Deployment & configuration — install methods, config file format, environment-variable naming rule, `env.d/` recipes.
4. Event Action templates — context, function map, trigger-specific field availability, action-type sub-configs, pitfalls.
5. Workflow suggestions for AI agents producing SFTPGo artefacts.

---

## 0. Where to find more detail

| Need                                                                      | Source                                                                                                                                                                                  |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Field names, enum values, schema shape, required fields                   | OpenAPI spec: `https://sftpgo.com/assets/openapi.yaml` (latest release) or the `openapi/openapi.yaml` shipped with the deployed installation (older but authoritative for that server). |
| Operational how-to (install, storage, groups, shares, OIDC, tutorials, …) | Public docs at `https://docs.sftpgo.com/enterprise/` — mirrored at `https://github.com/sftpgo/docs` branch `enterprise`. Raw markdown: `https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/<file>.md` |
| Task → file cheat sheet for the public docs                               | `documentation-map.md` in the docs repo.                                                                                                                                                |
| Disambiguation of SFTPGo-specific terms                                   | `glossary.md` in the docs repo.                                                                                                                                                         |
| Ready-to-paste Event Action templates with both WebAdmin and REST forms   | [`skills/sftpgo/references/examples.md`](skills/sftpgo/references/examples.md) in this repo.                                                                                            |

When in doubt about a specific field, enum value, or required property: **read the OpenAPI spec**. This reference documents what the spec cannot easily encode (tagged unions, template semantics, cross-field dependencies).

---

## 1. REST API payload conventions

These rules apply to every `/api/v2/*` endpoint and, by equivalence, to every WebAdmin form (the UI posts the same JSON shape).

### 1.1 Authentication

- **JWT token** — `POST /api/v2/token` with admin HTTP Basic auth. Short-lived; refresh flow provided.
- **API key** — created via `/api/v2/apikeys`; sent as header `X-SFTPGO-API-KEY: <key>`. Preferred for non-interactive automation.
- **OIDC** — only for interactive WebAdmin / WebClient login, not for REST API calls.

### 1.2 Tagged unions (the most common trap)

Many fields use an integer selector to pick which sub-configuration is active; the other sub-objects are ignored.

- `User.filesystem.provider` — `0` local, `1` S3, `2` GCS, `3` Azure Blob, `4` encrypted local, `5` SFTP, `6` HTTP, `7` FTP. Only the matching `*config` sub-object is read (`s3config`, `gcsconfig`, `azblobconfig`, `cryptconfig`, `sftpconfig`, `httpconfig`, `ftpconfig`).
- `EventAction.options.type` — selects which of `http_config`, `cmd_config`, `email_config`, `retention_config`, `fs_config`, `pwd_expiration_config`, `idp_config`, `user_inactivity_config`, `imap_config`, `icap_config`, `share_expiration_config`, `event_report_config` applies. See §4 for the full enum.
- `fs_config.type` (inside a filesystem action) — selects the operation (rename / delete / mkdirs / exist / compress / copy / pgp / metadata-check / decompress).

Send the matching sub-object for the selected `type`. Other sub-objects are zeroed on save (the server enforces this).

### 1.3 Secret envelope

Every secret-typed field on the JSON payload (backend credentials, API keys, service account JSON, IMAP/SMTP passwords, OIDC client secret, share password, …) uses an envelope:

```json
{"status": "Plain", "payload": "your-secret"}
```

The server validates, encrypts, and stores it. On `GET` the same envelope comes back with the encrypted form — by default `Secretbox` (NaCl AEAD) for the built-in local KMS. With an external provider the `status` can be `AWS`, `GCP`, `VaultTransit`, `AzureKeyVault`, `OracleKeyVault`, or `AES-256-GCM`. The `key` and `additional_data` fields are blanked out before serialization:

```json
{"status": "Secretbox", "payload": "<ciphertext>", "key": "", "additional_data": ""}
```

**To preserve** the current secret on a `PUT`, send any envelope whose `status` is the encrypted status the server is using — the payload can be empty. The simplest form is just the status field:

```jsonc
// PUT /api/v2/users/alice — keep the existing access_secret, change only home_dir
{
  "username": "alice",
  "home_dir": "/srv/sftpgo/data/alice-2026",
  "filesystem": {
    "provider": 1,
    "s3config": {
      "bucket": "my-bucket",
      "region": "eu-west-1",
      "access_key": "AKIA…",
      "access_secret": {"status": "Secretbox"}
    }
  }
}
```

Use whatever `status` value the `GET` response showed — `Secretbox` for the default local KMS, `AWS` / `GCP` / `VaultTransit` / `AzureKeyVault` / `OracleKeyVault` / `AES-256-GCM` for the corresponding external KMS providers. Round-tripping the full envelope from `GET` works too — extra fields are ignored.

Sending `{"status": "Plain", "payload": "newvalue"}` replaces the secret. Sending `{"status": "Plain", "payload": ""}` clears it.

For the top-level User `password` field (a hashed scalar, not a secret envelope) preservation is by **omission**: drop the field from the body and the server keeps the current hash. Sending `"password": ""` clears it.

### 1.4 Idempotency and update shape

- `POST` creates every time; there is no built-in upsert. If the unique key already exists the endpoint returns `409 Conflict`.
- `PUT /api/v2/<resource>/<id>` is a **full-object replace**. Omitted fields are reset to default.
- Safe update pattern: `GET` → mutate in memory → `PUT`. When mutating only a few fields, make sure you round-trip the ones you did not touch.
- Status codes: `201 Created` + `Location` header on creation, `200 OK` on update, `400` validation error (message is the actual reason — read it), `403` permission, `404` not found, `409` unique-key conflict.

### 1.5 User payload minimum (per backend)

Every user must end up with a `home_dir`. If the payload omits it, the server fills it as `<users_base_dir>/<username>` when `data_provider.users_base_dir` is set. The Linux APT / RPM packages set `users_base_dir` to `/srv/sftpgo/data` by default, so on packaged Linux installs `home_dir` is effectively optional in the API payload. The shipped `sftpgo.json`, the Docker image, and the Windows installer leave it empty — there `home_dir` must be supplied (or `users_base_dir` configured) for every new user.

`home_dir` is held in the database for every backend, including cloud / remote ones. For S3 / GCS / Azure / SFTP / FTP / HTTP it isn't used for the actual data path (files stream through pipes), but it still has to exist as an absolute path on disk; the simplest answer is to set `users_base_dir` once and stop thinking about it.

| Provider             | Integer | Required fields                                                | Auth (when explicit fields are omitted)                                                                                                                                                                              |
| -------------------- | ------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Local                | 0       | *(none — files live under `home_dir`)*                         | n/a                                                                                                                                                                                                                  |
| S3                   | 1       | `s3config.bucket`, `region` (or embedded in `endpoint`)        | If `access_key` + `access_secret` are both empty, the AWS SDK falls back to its **default credential chain** — env vars (`AWS_*`), shared config, EC2 instance profile, ECS / EKS task role, IRSA. `role_arn` works on top of any of these. |
| Google Cloud Storage | 2       | `gcsconfig.bucket`                                             | Set `automatic_credentials: 1` to use **Application Default Credentials** (GCE/GKE metadata, Workload Identity, `GOOGLE_APPLICATION_CREDENTIALS`, `gcloud` ADC). When `0`, supply `credentials` (service-account JSON envelope).            |
| Azure Blob           | 3       | `azblobconfig.container`, `account_name` *(or `sas_url`)*      | If both `account_key` and `sas_url` are empty, the SDK falls back to `DefaultAzureCredential` — managed identity, env vars, workload identity, Azure CLI, Visual Studio sign-in, in that order.                                            |
| Encrypted local      | 4       | `cryptconfig.passphrase`                                       | n/a                                                                                                                                                                                                                  |
| SFTP backend         | 5       | `sftpconfig.endpoint`, `username`, `password` or `private_key` | n/a — SFTP authentication is always explicit.                                                                                                                                                                        |
| HTTP filesystem      | 6       | `httpconfig.endpoint`                                          | Auth (`username` / `password` / `api_key`) is only what the remote HTTP service requires.                                                                                                                            |
| FTP backend          | 7       | `ftpconfig.endpoint`, `username`, `password`                   | n/a                                                                                                                                                                                                                  |

**Managed-credential rule of thumb.** When SFTPGo runs on AWS / GCP / Azure with the host's identity already wired (instance profile, GKE workload identity, AKS managed identity, IRSA, etc.), leave the credential fields empty on the user / group / virtual-folder filesystem config — the SDKs pick the host identity up automatically. This is the recommended posture in cloud deployments: nothing sensitive in the SFTPGo data provider, IAM scopes managed at the platform level.

Every user must have `permissions["/"]` set. Permissions are a map `virtual path → operation tokens`: `*` (all), `list`, `download`, `upload`, `overwrite`, `delete` (or `delete_files` + `delete_dirs`), `rename` (or `rename_files` + `rename_dirs`), `create_dirs`, `create_symlinks`, `chmod`, `chown`, `chtimes`, `copy`. Longest-prefix match wins.

### 1.6 Groups condensed

| Type       | JSON `type` | Count per user | Contributes                                                                                                                                |
| ---------- | ----------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Primary    | `1`         | At most one    | Base configuration (filesystem, home, quotas, bandwidth, password policy). Applied only when the user's field is 0/empty.                  |
| Secondary  | `2`         | Any number     | Additive: extra virtual folders, per-path permissions, IP filters, share policies, WebClient permissions.                                  |
| Membership | `3`         | Any number     | **Nothing is inherited.** Pure tag for event-rule conditions.                                                                              |

Placeholders accepted inside group values: `%username%`, `%role%`, `%custom1%…%customN%` (from `filters.custom_placeholders`). Apply in `home_dir`, `key_prefix`, `starting_directory`, virtual-folder `virtual_path`, SFTP backend `username`.

### 1.7 Shares

Scope integers: `1` read, `2` write, `3` read+write. Options: password, expiration, IP allow-list, `options.auth_mode: 1` for email authentication.

### 1.8 External identity providers

SFTPGo supports **OIDC** for external IdPs (Microsoft Entra ID / B2C, Google Workspace, Okta, Keycloak, Auth0, Authentik, …) and **LDAP / Active Directory** as an authentication source. SAML is not implemented.

---

## 2. Deployment & configuration

### 2.1 Install methods

| Target                           | Method                                                                                                                                                                           |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Debian / Ubuntu                  | APT repo `https://download.sftpgo.com/apt` — installs the `sftpgo` package and registers a systemd unit.                                                                         |
| RHEL / CentOS / Rocky / SUSE     | YUM / zypper repo `https://download.sftpgo.com/yum/<arch>`.                                                                                                                      |
| Windows                          | Signed Windows installer (the official method — registers a service, default config dir `C:\ProgramData\SFTPGo Enterprise`). Also available via `winget install -e --id drakkan.SFTPGoEnterprise` for hosts that have winget.                                                                        |
| Docker                           | Official registry (license required for Enterprise).                                                                                                                             |
| Kubernetes                       | Official Helm chart.                                                                                                                                                             |
| AWS / Azure / GCP                | Marketplace listings (Starter / Premium). License bundled with subscription.                                                                                                     |

Without a valid license, SFTPGo runs in **limited mode**. Limits: 2 concurrent transfers, plugins disabled, local filesystem only, no PGP / ICAP / IDP account check / Event report actions.

**Cloud marketplace offerings** ship preconfigured: license activated, audit-logs enabled, in-memory transfer pipes on, data provider initialized for standalone use. Audit-logs is implemented as **two** plugins (`eventstore` + `eventsearcher`) wired together — both must be loaded for the audit-logs UI to work. The Starter marketplace tier allows **2 active plugins total**, which audit-logs already fills, so on Starter there is no spare slot for a third plugin without dropping audit-logs. Premium and higher tiers have no plugin limit.

**Marketplace OS images.** AWS AMIs (x86_64 and arm64) are based on **Amazon Linux 2023** (RHEL-based, `dnf`). Azure and Google Cloud Linux VMs are based on **Debian 13** (`apt`). Both SFTPGo and the OS are updated through the distribution's standard package manager — `sudo dnf update` on AWS, `sudo apt update && sudo apt upgrade` on Azure / GCP. The SFTPGo service is restarted automatically by the package's post-install scriptlet when the `sftpgo` package is upgraded — no manual `systemctl restart` is required. Check the running version with `sftpgo --version`; the same version string is also logged at service startup (`starting SFTPGo v2.7.x ...`, visible via `journalctl -u sftpgo` or in the configured log file). Marketplace offerings hide the version in the WebAdmin by default, so on those instances the CLI and the startup logs are the canonical sources. Container-based marketplace listings have their own update procedure documented on each listing page (the AWS Container listing distributes images via the AWS marketplace registry; the Azure AKS listing is a CNAB bundle upgraded through its CNAB mechanism). **Pricing.** All marketplace listings — both Enterprise and open-source-based — are billed entirely through the cloud provider; there is no separate fee from SFTPGo on top. Enterprise listings include an activated SFTPGo Enterprise license (nothing else to purchase or activate); open-source-based listings ship the SFTPGo Open Source edition (AGPLv3) and do not enable Enterprise features. Do not speculate about how the marketplace charge is split between compute and software — answer customer pricing questions in terms of what they pay for and what they get, not in terms of internal margin. Authoritative reference: [installation.md `## Commercial Marketplaces`](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md).

### 2.2 Config file location and format

The binary looks for a file named `sftpgo.<ext>` in the configured *config directory*. Supported extensions (via viper): **JSON** (default shipped), **YAML**, **TOML**, **HCL**, **Java properties**, **envfile**.

| Platform                 | Default config dir                  |
| ------------------------ | ----------------------------------- |
| Debian / RPM (systemd)   | `/etc/sftpgo`                       |
| Docker image             | `/var/lib/sftpgo` (override with `-c`) |
| Windows installer        | `C:\ProgramData\SFTPGo Enterprise`  |

Override with `-c <path>` on any `sftpgo` subcommand. On Linux the systemd unit sources `/etc/sftpgo/sftpgo.env` and also reads every regular file under `/etc/sftpgo/env.d/` before starting. On Windows the service reads every regular file from `C:\ProgramData\SFTPGo Enterprise\env.d\` at startup. The `.env` suffix is only a convention — any file name is accepted.

**Recommendation**: do not edit the shipped `sftpgo.json`. Keep it at defaults and put all overrides in env files under `env.d/` — package upgrades that ship a new default config cannot then conflict with your edits.

**File config vs WebAdmin UI.** TLS certificates, ACME, SMTP, branding, OIDC, LDAP, and Geo-IP are all exposed in both file config and the WebAdmin UI. **Prefer the UI when available**: it persists in the database (encrypted when appropriate), replicates across HA nodes, and most changes take effect without a service restart. Reserve file-based config for what the UI doesn't cover (bindings, transport-level options, hooks, memory pipes, licensing) or for headless deployments.

**Two Configurations sections are hidden behind opt-in env-var toggles** (the underlying features are built into SFTPGo — the toggle just exposes the form):

| WebAdmin section                        | Required env var                |
| --------------------------------------- | ------------------------------- |
| OIDC                                    | `SFTPGO_HOOK__ENABLE_OIDC_UI=1` |
| TLS certificates (default key pair)     | `SFTPGO_HOOK__ENABLE_TLS_UI=1`  |

These are env-var-only flags (no UI / API equivalent) and they take effect at process startup, so after setting one you must restart SFTPGo for the section to appear. On marketplace and SaaS offerings the relevant flags are already enabled. Both are documented in the public [env-vars.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/env-vars.md).

LDAP and Geo-IP are plugin-based integrations: a separate plugin binary plus a plugin slot under the license are required, so the WebAdmin Configurations entries for them are wired to plugin setup rather than to a self-contained toggle. Refer the user to the [LDAP plugin guide](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/plugins/ldap-auth.md) and the Geo-IP plugin docs for the full setup; the SaaS / marketplace offerings come with these wired up already.

For **OIDC** specifically: the WebAdmin UI stores a single OIDC configuration in the database. The OIDC redirect URL is a single URL registered with the IdP, so the OIDC flow is served by the listener whose hostname matches the configured `redirect_base_url` — typically the primary one. To configure OIDC per HTTPD binding (e.g. different IdPs for internal admin and external client), use file or env-var config under `httpd.bindings[N].oidc` — the UI cannot express that.

### 2.3 Environment variable naming rule

```text
SFTPGO_<SECTION>__<KEY>[__<SUBKEY>…]
```

- Prefix: `SFTPGO_` (one underscore).
- Separator between struct levels: **double** underscore `__`.
- Keys are uppercased JSON keys (`defender` → `DEFENDER`).

Examples:

| JSON path                       | Env var                                |
| ------------------------------- | -------------------------------------- |
| `common.defender.enabled = true`      | `SFTPGO_COMMON__DEFENDER__ENABLED=true`   |
| `webdavd.cors.enabled = true`         | `SFTPGO_WEBDAVD__CORS__ENABLED=true`      |
| `data_provider.driver = "postgresql"` | `SFTPGO_DATA_PROVIDER__DRIVER=postgresql` |
| `telemetry.enable_profiler = true`    | `SFTPGO_TELEMETRY__ENABLE_PROFILER=true`  |

### 2.4 Lists

Scalar lists are comma-separated:

```shell
SFTPGO_COMMON__ACTIONS__EXECUTE_ON=upload,download
```

Lists of structs are indexed:

```shell
SFTPGO_SFTPD__BINDINGS__0__PORT=22
SFTPGO_SFTPD__BINDINGS__0__ADDRESS=0.0.0.0
SFTPGO_SFTPD__BINDINGS__1__PORT=2022
```

Indices must be contiguous: a gap (`__0__`, `__2__`) leaves index 1 unseen.

### 2.5 Env var precedence

1. **Process environment** (init system, `docker run -e`, `kubectl set env`).
2. **`<config-dir>/env.d/`** — every regular file in the directory is loaded at SFTPGo startup (the `.env` suffix in examples is only a convention — any file name works). Does **not** override existing env.
3. **`EnvironmentFile=/etc/sftpgo/sftpgo.env`** (Linux systemd) — loaded before the binary starts, acts like process env.

### 2.6 Common production recipes

```shell
# Bind SFTP on port 22
SFTPGO_SFTPD__BINDINGS__0__ADDRESS=
SFTPGO_SFTPD__BINDINGS__0__PORT=22

# PostgreSQL data provider
SFTPGO_DATA_PROVIDER__DRIVER=postgresql
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST=db.internal
SFTPGO_DATA_PROVIDER__PORT=5432
SFTPGO_DATA_PROVIDER__USERNAME=sftpgo
SFTPGO_DATA_PROVIDER__PASSWORD='your-password'
SFTPGO_DATA_PROVIDER__SSLMODE=1

# Defender (intrusion detection)
SFTPGO_COMMON__DEFENDER__ENABLED=true
SFTPGO_COMMON__DEFENDER__DRIVER=provider  # 'memory' on single-node, 'provider' for HA
SFTPGO_COMMON__DEFENDER__BAN_TIME=60

# ACME (Let's Encrypt) — prefer the WebAdmin UI (Server Manager → Configurations
# → ACME). For headless / scripted setups, use file-based config as below and
# run `sftpgo acme run --database-protocols 7` once to register and obtain the
# first certificate; renewals are then automatic.
#   --database-protocols bitmask: 1 HTTPS, 2 FTPS, 4 WebDAV (combinable; 7 = all)
#   key_type: 2048 / 3072 / 4096 / 8192 (RSA) or P256 / P384 (ECDSA)
SFTPGO_ACME__EMAIL=ops@example.com
SFTPGO_ACME__KEY_TYPE=4096
SFTPGO_ACME__DOMAINS=sftp.example.com
SFTPGO_ACME__HTTP01_CHALLENGE__PORT=80

# Telemetry + metrics + profiler (internal only; port 10000 is the convention,
# hardcoded by the official Helm chart and used throughout the docs)
SFTPGO_TELEMETRY__BIND_ADDRESS=127.0.0.1
SFTPGO_TELEMETRY__BIND_PORT=10000
SFTPGO_TELEMETRY__ENABLE_PROFILER=true
SFTPGO_TELEMETRY__AUTH_USER_FILE=/etc/sftpgo/telemetry.htpasswd

# HAProxy PROXY protocol
SFTPGO_COMMON__PROXY_PROTOCOL=2      # 2 = strict
SFTPGO_COMMON__PROXY_ALLOWED=10.0.0.0/8,192.168.0.0/16

# Memory pipes (required on read-only / ephemeral / distroless hosts)
SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1

# License key
SFTPGO_LICENSE_KEY=XXXX-XXXX-XXXX-XXXX
```

**About memory pipes.** On cloud storage backends (S3 / GCS / Azure Blob), each transfer streams through a per-upload pipe. By default that pipe is file-backed and requires a writable local temp directory. On distroless containers, read-only rootfs, and Kubernetes pods without a writable PVC there is no such directory, and uploads fail with errors like "unable to open pipe" / "read-only file system". Setting `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1` makes transfers fully in-memory. Higher RAM per concurrent transfer, but no filesystem dependency. Already the default on SFTPGo cloud-marketplace offerings.

### 2.7 Schema init & migrations

With the default `data_provider.update_mode = 0`, SFTPGo creates the schema on first startup and migrates on every subsequent startup — no manual step needed beyond creating the empty database (PostgreSQL / MySQL / CockroachDB) and granting DDL to the configured user. Run `sftpgo initprovider` only if you set `update_mode = 1` to disable automatic updates (e.g. runtime credentials have no DDL privileges).

**Concurrent migrations across multiple SFTPGo instances.** PostgreSQL serializes schema migrations via `pg_advisory_lock`, so concurrent SFTPGo startups against the same PostgreSQL cluster are safe. MySQL/MariaDB and CockroachDB don't expose an equivalent advisory-lock primitive that fits this pattern, so for those engines the operator coordinates startup instead — either start the SFTPGo instances sequentially on first deploy and on every version bump, or set `update_mode = 1` everywhere and run `sftpgo initprovider` from a single one-shot job (CI step, Helm post-upgrade hook, etc.) before rolling the rest out. The CockroachDB story is identical to MySQL here; it does not require a different runbook.

**MySQL transaction isolation.** MySQL/MariaDB default to `REPEATABLE READ`, which uses gap locks on index ranges. Under high write concurrency (e.g. a wave of bulk user updates, sync jobs hitting tens of users in parallel) this surfaces as `Deadlock found when trying to get lock; try restarting transaction` in the SFTPGo logs. Set `transaction-isolation = READ-COMMITTED` in `my.cnf` and restart MySQL — that removes the gap-lock contention. PostgreSQL has no such issue at the default isolation level.

### 2.8 Reload commands after config change

- Linux: `sudo systemctl restart sftpgo` (full restart), `sudo systemctl reload sftpgo` (sends SIGHUP — re-reads file-based TLS certs and CRLs immediately).
- Windows: `Restart-Service sftpgo`, `sftpgo service reload` (sends `paramchange` — same scope as SIGHUP).

File-based TLS certs and CRLs are also picked up automatically by a built-in 8-hour file watcher (size + mtime), so a manual reload after `certbot` / `cert-manager` updates is only needed if you can't wait up to 8 hours. Certs/CRLs stored in the database via the WebAdmin UI or refreshed by the built-in ACME client are reloaded immediately on save / renewal.

### 2.9 Docker — production shape

```shell
docker run -d --name sftpgo \
  -p 22:22 \
  -p 8080:8080 \
  -e SFTPGO_SFTPD__BINDINGS__0__PORT=22 \
  -e SFTPGO_LICENSE_KEY=XXXX-XXXX-XXXX-XXXX \
  -e SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1 \
  --mount type=bind,source=/srv/sftpgo-data,target=/srv/sftpgo \
  --mount type=bind,source=/srv/sftpgo-config,target=/var/lib/sftpgo \
  registry.sftpgo.com/sftpgo/sftpgo:<tag>
```

Host dirs must be writable by UID `1000` (`chown -R 1000:1000 /srv/sftpgo-data /srv/sftpgo-config` — no Linux user with UID 1000 needed on the host). With Docker / Docker Compose use `-e` flags or the `environment:` block / Compose `env_file:` directive — that's the idiomatic way to pass `SFTPGO_…` env vars in containers. The `<config-dir>/env.d/` mechanism is mostly useful for systemd-packaged installs; mounting it inside a container works but adds nothing over plain env vars. Distroless images on cloud-only deployments must set `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1`.

### 2.10 Kubernetes — shape and SFTPGo-specific decisions

Official Helm chart: `oci://ghcr.io/sftpgo/helm-charts/sftpgo`. The chart is **edition-agnostic** — its default `image.repository` is `ghcr.io/drakkan/sftpgo` (Open Source). For Enterprise, override `image.repository` (typically `registry.sftpgo.com/sftpgo/sftpgo`, with the appropriate `-plugins` / `-distroless` variant) and add the license key as a Kubernetes `Secret`. Everything else in this section applies to both editions.

The critical choices an AI agent must get right:

- **Expose SFTP via Service `LoadBalancer` / `NodePort`, not Ingress.** SFTP / FTP / WebDAV are plain TCP — an HTTP/L7 Ingress cannot forward them. The HTTP WebAdmin / WebClient can sit behind an Ingress. Cloud: AWS NLB, Azure TCP LB, GCP TCP LB. NGINX Ingress can forward TCP via the [TCP services ConfigMap](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/).
- **Multi-replica needs an external data provider.** SQLite / bolt are single-writer; for `replicaCount > 1` use PostgreSQL / MySQL / CockroachDB (remember: CockroachDB has no `pg_advisory_lock` → serialize migrations or use `update_mode=1`).
- **Persistence depends on the backend mix.** Local-home users need a PVC (RWO for single replica, RWX for multi). Cloud-backend-only deployments can run fully ephemeral with memory pipes enabled.
- **Memory pipes are strongly recommended on Kubernetes** — set `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1` (preconfigured on EKS / AKS marketplace offerings). With file-backed pipes the cloud-backend transfer path tries to create unlinked files in the configured temp directory; it works only if a writable, large-enough volume is mounted there, and that's an avoidable footgun on ephemeral / read-only / distroless pods. Memory pipes are an Enterprise feature; on the Open Source build file-backed pipes are the only option, so make sure the temp dir is on writable storage with enough headroom.
- **Health probes** → the chart wires them up automatically against `/healthz` on port `10000` (telemetry, hardcoded by the chart). No manual probe config needed.
- **License key + DB password** → Kubernetes `Secret` mounted via `envFrom`, never in committed values.yaml.
- **Shared SSH host keys across replicas.** Without this each pod generates its own keys and clients see mismatch warnings. Two patterns:
  1. **Paste the keys in the WebAdmin** (Server Manager → Configurations → SFTP) — stored encrypted in the data provider, loaded identically by every replica sharing the DB. Preferred for new deployments.
  2. **Mount a `Secret` with the key files** and point SFTPGo at the paths via `SFTPGO_SFTPD__HOST_KEYS` (comma-separated). The chart exposes `volumes:` / `volumeMounts:` / `envVars:` extension points for this:
     ```yaml
     volumes:
       - name: ssh-keys
         secret:
           secretName: sftpgo-host-keys
           optional: false
     volumeMounts:
       - name: ssh-keys
         mountPath: /etc/sftpgo/ssh-keys
         readOnly: true
     envVars:
       - name: SFTPGO_SFTPD__HOST_KEYS
         value: "/etc/sftpgo/ssh-keys/id_rsa,/etc/sftpgo/ssh-keys/id_ecdsa"
     ```

### 2.11 Operational recipes

- **Reset a lost admin password**: `sftpgo resetpwd --admin <username>`. Interactive prompt; also disables 2FA on that admin. Not for the memory provider. Stop SFTPGo first when using embedded bolt / SQLite; safe to run live on shared PostgreSQL / MySQL / CockroachDB. On Windows: `"C:\Program Files\SFTPGo\sftpgo.exe" resetpwd --admin <username> -c "C:\ProgramData\SFTPGo Enterprise"`.
- **Rotate logs on demand**: `kill -USR1 $(pgrep -x sftpgo)` on Linux; `sftpgo service rotatelogs` on Windows.
- **Reload TLS certs after swapping files on disk** (file-based certs only): `systemctl reload sftpgo` (Linux) / `sftpgo service reload` (Windows, sends `paramchange`). Without a manual reload the built-in 8-hour file watcher will pick the new files up on its own. UI- and ACME-managed certs reload immediately on save / renewal.
- **Dump / restore data**: `POST /api/v2/dumpdata` / `POST /api/v2/loaddata` or WebAdmin Maintenance. Portable across DB types and supported upgrade paths.

### 2.12 Reading the logs

SFTPGo writes structured JSON, one object per line; each record carries a `sender` field. Locations:

| Install                       | Location                                                                                     |
| ----------------------------- | -------------------------------------------------------------------------------------------- |
| Debian / RPM systemd          | `/srv/sftpgo/logs/sftpgo.log` + `journalctl -u sftpgo`                                        |
| Docker / Kubernetes           | stdout — `docker logs <name>` / `kubectl logs <pod>`                                         |
| Windows installer service     | `C:\ProgramData\SFTPGo Enterprise\logs\sftpgo.log` (the application log; the Windows Service Control Manager logs only service-lifecycle events — start / stop / crash — to the System log, not application data) |

Useful `sender` values: `login` (successful logins), `connection_failed` (auth failures, client aborts, timeouts — `error` has the reason), `Upload` / `Download`, command senders (`Rename`, `Mkdir`, `Rmdir`, `Remove`, `Chmod`, `Chown`, `Chtimes`, `Copy`, `Truncate`, `SSHCommand`), `httpd` (REST / WebAdmin / WebClient).

```shell
# Failed login attempts
tail -n 1000 /srv/sftpgo/logs/sftpgo.log \
  | jq -c 'select(.sender=="connection_failed") | {time,username,client_ip,protocol,error}'

# 5xx REST API responses
grep '"sender":"httpd"' /srv/sftpgo/logs/sftpgo.log \
  | jq -c 'select(.resp_status >= 500) | {time,method,uri,resp_status,remote_addr}'
```

### 2.13 Deployment pitfalls

- **Avoid setting `common.temp_path`.** Leave empty unless you have a specific reason — a path on a different filesystem from user homes causes atomic renames to fall back to copy+delete.
- **Cloud backend upload fails with "unable to open pipe" / "read-only file system" / "no space left"** → SFTPGo cannot create a file-backed pipe in the configured temp dir. Fix: enable memory pipes (`SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1`). Mandatory for distroless / read-only rootfs / pods without a writable PVC; already default on marketplace offerings.
- **Local user "permission denied" on upload while login succeeds** → the APT/YUM packages run under a dedicated `sftpgo` system account; a `home_dir` outside `/srv/sftpgo` you created as root is unreadable/unwritable by it. Fix on Linux: `chown -R sftpgo:sftpgo <home>` and `chmod +x` the parents. On Windows: grant the service account (`LOCAL SYSTEM` by default) write access on the chosen tree via `icacls`.
- **`proxy_protocol=2` without populated `proxy_allowed`** → every connection is rejected. Configure both together.
- **`allowlist_status=1` before populating the list** → drops everything.
- **Changing a tagged-union `driver`/`provider` without its sub-config** → service starts but cannot reach the target (DB, backend, IdP).
- **`env.d` does not override process env**. If a value is exported at the systemd / container level, changes under `env.d/` are ignored until the outer source is cleared.
- **TLS file-based cert reload latency** → after swapping cert files on disk, in-memory keys stay stale until the built-in 8-hour file watcher picks them up. To force an immediate reload use `systemctl reload sftpgo` / `sftpgo service reload`. UI- and ACME-managed certs reload immediately on save / renewal — no action needed.
- **Non-contiguous list-of-struct indices** → a `__0__` + `__2__` without `__1__` leaves index 1 unset.
- **Running `initprovider` when not needed** → harmless but unnecessary with `update_mode = 0`. Needed only with `update_mode = 1`.
- **CockroachDB + concurrent migrations** → no `pg_advisory_lock`. Serialize instance startup or set `update_mode=1` everywhere and run `initprovider` from a single operator job.
- **Windows service account change + privileged ports** → make sure the new account still has the right to bind low ports.
- **Windows path escaping in env files** → single quotes are literal (`'C:\path'`); double quotes require backslash escaping (`"C:\\path"`); forward slashes also work.
- **Per-replica host keys in K8s** → without explicit sharing each pod generates its own keys and clients see key-mismatch warnings. Store host keys in the data provider via WebAdmin (Server Manager → Configurations → SFTP) or mount a Kubernetes Secret with the private keys.

### 2.14 Fully-managed SaaS at sftpgo.com

Separate distribution channel: we own the host, the customer only sees WebAdmin + WebClient + SFTP + REST. **No shell, no config file, no env vars.** Detection signals: host is a dedicated per-instance subdomain we assigned (or a customer-owned subdomain CNAME'd onto it), `bucket=default` + `region=default` in the config, customer mentions "managed service" / "SaaS" / "sftpgo.com plan". When in doubt, ask the user.

Preconfigured on every plan:

- Integrated S3: sentinel values `bucket=default`, `region=default`; credentials managed server-side and rotated periodically with no customer action.
- **`default` group, already created** and pre-wired with the integrated storage and key prefix `users/%username%/`. Assigning a user to `default` as **Primary** is the canonical answer to "how do I add a storage-isolated user on SaaS" — the user inherits the filesystem from the group, no per-user filesystem fields needed.
- Simplified user form for the shipped default admin (Groups picker hidden, `default` applied automatically). Additional admins the customer creates see the full form.
- Audit logs, Defender, recycle bin (3-day retention; bin contents count against the user's quota) enabled.
- TLS / Let's Encrypt cert on the default domain.
- Daily backups managed by us. The **Backup event action is disabled** as a result; for an additional copy, use an Event Manager rule that copies files to storage the customer owns, or pull a JSON dump via REST.

Tier gates: Small+ for Geo-IP and HIPAA; Standard+ for PGP, LDAP, OIDC, ICAP, IDP account check; Document editing (Collabora) is an add-on.

**HIPAA mode** (Small+): we enable on request, signed BAA. Instance deployed in the US, idle timeouts halved (20 → 10 min), SSE-C **mandatory** on the integrated storage (without a key SFTPGo refuses to save users/groups/folders that use the integrated storage), audit logs retained 6 years and replicated off-site, companion read-only install for CSV exports.

Configuration changes that trigger a short restart: **OIDC**, **LDAP**, **first-time Allow List**, and **adding a custom domain in Automatic Certificates (Let's Encrypt)** when self-served. HIPAA does not appear in this list (we enable it via CLI). Tell the customer to save the config and wait. We are **notified automatically**; in Italy business hours (09:00–20:00 CET / CEST) we verify and restart within a few minutes. No ticket required. Off-hours urgencies: email support.

Custom domain: DNS **CNAME** pointing a customer subdomain (e.g. `sftp.example.com`) to the dedicated address we emailed the customer at activation. The IP behind that address is static under normal operation, so an A record technically works; we recommend a CNAME because the IP can change in rare events such as a region migration or major infrastructure rework — we always notify the customer ahead of time. Apex / root domains can't use CNAME (DNS standards limitation); use a subdomain or a provider-specific ALIAS / CNAME-flattening record. Once the CNAME resolves, the customer either declares the new domain themselves from **Server Manager → Configurations → Automatic Certificates (Let's Encrypt)** (one of the changes that triggers a short restart, see above), or emails support and we apply it server-side via CLI. The Let's Encrypt certificate is issued during that step, not automatically from the CNAME alone. Whichever path is chosen, **keep the original assigned hostname in the cert list** — it's how we monitor the instance.

What NOT to suggest to a SaaS user: `systemctl` / `Restart-Service` / `docker run` / Helm values / `sftpgo resetpwd` / editing `sftpgo.json` / setting `SFTPGO_…` env vars / the Backup event action. All of §2.1–2.13 above is off-limits on the SaaS. Use WebAdmin forms and REST API only.

**Marketplace offerings ≠ SaaS.** AWS / Azure / GCP cloud-marketplace listings are self-hosted — the customer owns the VM or managed K8s namespace. They can edit env files and run CLI commands. Use §2.1–2.13 for them.

### 2.15 Open Source edition and third-party distributions

Almost everything in this reference targets the **Enterprise** edition. The **Open Source** edition (`github.com/drakkan/sftpgo`, Docker Hub `drakkan/sftpgo`) has the full protocol stack, every storage backend, the WebAdmin and WebClient, the REST API, basic OIDC, and a basic Event Manager — but it does not have:

- The full Go template engine with `toJson` / helpers / conditions / loops (most examples in §3 and in `examples.md` assume this).
- Action types: **ICAP**, **IMAP**, **event reports** (type 19), **PGP**, **execute-before-file-publish** staged uploads, enhanced copy with source disposition / glob / retries, data retention with archival.
- OIDC role mapping, PKCE-without-secret, session control, Azure B2C, custom labels.
- WebClient WOPI / Collabora, TUS resumable uploads.
- Share email authentication, group governance, path / scope restrictions.
- Cloud backend performance optimizations (memory pipes / small-file speedups), GCS HNS, SFTP backend SOCKS proxy, FTP backend.
- Clustering, the WebAdmin UI for LDAP / Geo-IP / email templates / SSH host keys, API key management UI, and the Enterprise builds of the `eventstore` / `eventsearcher` plugins that power the audit-logs browsing UI (open-source builds of those plugins are also available on OSS, and the browsing UI itself works in both editions — what differs is the depth of the captured data).

Canonical comparison: the "What Enterprise adds" table on the Enterprise docs landing page (`https://docs.sftpgo.com/enterprise/`, raw: `https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/index.md`).

Distribution channels for the OSS binary include GitHub releases and Docker Hub, plus third-party repackagings by managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, and unofficial Docker images / Kubernetes manifests. **These third-party distributions are not endorsed by the SFTPGo project.** They deliver the OSS feature set; their own support channels are separate; bug reports about the upstream binary belong on `github.com/drakkan/sftpgo`.

Detection signals for OSS: user installed from `github.com/drakkan/sftpgo`, image is `drakkan/sftpgo:...`, install came from any of the third-party wrapper types above, "free version" / "no license", the cloud-marketplace listing is explicitly labelled "SFTPGo Open Source". Docs for OSS live at `https://docs.sftpgo.com/latest/` (raw: `https://raw.githubusercontent.com/sftpgo/docs/main/docs/<file>.md`).

When answering an OSS user:

- Lead with OSS-compatible suggestions when they exist (basic HTTP action without `toJson` still works for simple webhooks).
- When the feature is Enterprise-only, state it once, neutrally — *"this is an Enterprise-only feature; on OSS the closest equivalent is X, or there is none"* — and stop. No repeated upgrade pitches.
- Do not invent OSS workarounds for things that genuinely require Enterprise (ICAP, PGP, event reports, WOPI, clustering, email-authenticated shares, LDAP, most of the UI-based configuration pages).
- When uncertain whether the user is on OSS or Enterprise, ask.

---

## 3. Event Action templates

### 3.1 What these templates are

SFTPGo Event Actions execute in response to triggers (filesystem events, provider events, cron schedules, IDP logins, IP blocks, certificate renewals, on-demand runs). Several fields on each action are rendered against a per-event context before being sent or executed: email body/subject, HTTP body/headers, command args, IDP `template_user` / `template_admin`, ICAP and filesystem paths, IMAP subject/body.

The engine differs by edition — and sections §3.2 through §3.6 below all describe the **Enterprise** behavior:

- **Enterprise** — full Go `text/template` (not Handlebars, not Jinja, not `html/template`). Conditionals (`{{if}}{{end}}`), iteration (`{{range}}{{end}}`), `{{with}}`, pipes (`{{.X | toJson}}`), and an extended `FuncMap` (§3.3). Structured accessors like `{{.Object.User.Email}}` and `{{index .IDPFields "claim"}}` work. Templates are compiled at action-save time, so a syntax error rejects the `POST/PUT` with a validation error; a semantic mistake (referencing a field the trigger does not populate) silently renders as the empty string / zero value at runtime.
- **Open Source** — a flat `strings.Replacer` pass over a fixed list of dot-prefixed placeholders. The shape (`{{.Name}}`, `{{.VirtualPath}}`, …) is identical to Enterprise, but it is **string substitution only** — no conditionals, no ranges, no functions, no nested accessors. Tokens not in the fixed list survive verbatim in the output, which is the most common "why is `{{toJson .X}}` showing up literally in my webhook" symptom on OSS.

OSS placeholder list (everything else — `{{if}}`, `{{range}}`, pipes, `{{.Object.User.X}}`, `{{index .IDPFields "k"}}` — does **not** work on OSS):

`{{.Name}}`, `{{.Event}}`, `{{.Status}}`, `{{.StatusString}}`, `{{.VirtualPath}}`, `{{.EscapedVirtualPath}}`, `{{.FsPath}}`, `{{.VirtualTargetPath}}`, `{{.FsTargetPath}}`, `{{.VirtualDirPath}}` (when fs path is set), `{{.VirtualTargetDirPath}}`, `{{.TargetName}}` (rename/copy), `{{.ObjectName}}`, `{{.ObjectBaseName}}`, `{{.ObjectType}}`, `{{.Ext}}`, `{{.FileSize}}`, `{{.Elapsed}}`, `{{.Protocol}}`, `{{.IP}}`, `{{.Role}}`, `{{.Email}}`, `{{.Timestamp}}` (UnixNano as string), `{{.DateTime}}`, `{{.Year}}`, `{{.Month}}`, `{{.Day}}`, `{{.Hour}}`, `{{.Minute}}`, `{{.UID}}`, `{{.ErrorString}}`, `{{.Metadata}}` (JSON), `{{.MetadataString}}` (escaped JSON), `{{.ObjectData}}` / `{{.ObjectDataString}}` (provider-event raw JSON, when the action requested object data), `{{.IDPField<claim>}}` (one flat token per OIDC claim listed in `custom_fields`).

Both editions share these rules:

- Dot prefix is mandatory: `{{.Name}}`, not `{{Name}}`.
- A field that the trigger does not populate is empty (Enterprise) or stays as the literal placeholder text (OSS).

### 3.2 Template context

Every template sees the following keys. Some are populated only for certain triggers — see §3.5.

| Key                                                | Type                                   | Summary                                                                                                                                                                                                       |
| -------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.Name`                                            | `string`                               | Event subject username (uploader, affected entity).                                                                                                                                                           |
| `.ExtName`                                         | `string`                               | External username from IDP when it differs from the mapped local one.                                                                                                                                         |
| `.Event`                                           | `string`                               | Event name — see §3.5 for per-trigger values.                                                                                                                                                                 |
| `.Status`                                          | `int`                                  | `1` success, `2` generic error, `3` quota exceeded (fs events).                                                                                                                                               |
| `.VirtualPath`                                     | `string`                               | Virtual path (fs events).                                                                                                                                                                                     |
| `.FsPath`                                          | `string`                               | Filesystem path (fs events).                                                                                                                                                                                  |
| `.VirtualTargetPath`, `.FsTargetPath`              | `string`                               | Destination paths (rename / copy only).                                                                                                                                                                       |
| `.ObjectName`                                      | `string`                               | File basename (fs) or primary key of the object (provider).                                                                                                                                                   |
| `.ObjectType`                                      | `string`                               | E.g. `user`, `folder`, `group`, `share`, `admin`, `api_key`, `event_rule`, `event_action` for provider events.                                                                                                |
| `.FileSize`                                        | `int64`                                | Bytes transferred (0 on `pre-*` and non-fs).                                                                                                                                                                  |
| `.Elapsed`                                         | `int64`                                | Milliseconds (0 on `pre-*` and non-fs).                                                                                                                                                                       |
| `.Protocol`                                        | `string`                               | `SFTP`, `SCP`, `SSH`, `FTP`, `DAV`, `HTTP`, `HTTPShare`, `OIDC`.                                                                                                                                              |
| `.IP`                                              | `string`                               | Client IP.                                                                                                                                                                                                    |
| `.Role`                                            | `string`                               | The user's role from the data provider. Populated for filesystem events (the connection's user role), provider events on a user-typed object, and chained actions that resolve a user. **Not** populated for IDP login (the role mapped from the OIDC binding's `role_field` is consumed by SFTPGo internally to drive the JIT user/admin shape; if you need it inside the template, add the same claim name to `custom_fields` and read it from `.IDPFields`). |
| `.Email`                                           | `string`                               | Subject's email — comma-joined list when the user has additional emails. For IDP login it is empty; expose the `email` claim explicitly via `custom_fields` and read it from `.IDPFields.email`.              |
| `.Timestamp`                                       | `time.Time`                            | Event time. Call methods: `.UTC`, `.Format`, `.Unix`, `.UnixNano`.                                                                                                                                            |
| `.UID`                                             | `string`                               | Unique event id (xid).                                                                                                                                                                                        |
| `.Metadata`                                        | `map[string]string`                    | Custom fs metadata.                                                                                                                                                                                           |
| `.IDPFields`                                       | `map[string]any`                       | **Only the OIDC claims listed in the binding's `custom_fields` config.** Access with `index` or `.IDPFields.key`.                                                                                             |
| `.Errors`                                          | `[]string`                             | Error messages when `.Status != 1`.                                                                                                                                                                           |
| `.Object`                                          | lazy                                   | Trigger object for provider events. Accessors: `.Object.JSON`, `.Object.User`, `.Object.Admin`, `.Object.Group`, `.Object.Share`.                                                                             |
| `.Initiator`                                       | lazy, never nil                        | Who caused the event. Populated for **provider events** (resolves to the admin or — for `delete-share` / share events — the owning user). Always exposed in the template (never nil) but **resolves to no User and no Admin for filesystem events, IP blocks, certificate events, schedules, and on-demand runs** — those events run without an "initiator" concept on the EventParams. Accessors: `.Initiator.User` (`*User` or nil), `.Initiator.Admin` (`*Admin` or nil), `.Initiator.JSON` (returns `{}` when no initiator). Always guard with `{{if .Initiator.User}}` / `{{if .Initiator.Admin}}`. |
| `.Shares`                                          | lazy                                   | Shares owned by `.Name`. Call `.Shares.Load` to get `[]Share`.                                                                                                                                                |
| `.RetentionChecks`                                 | array of retention check records       | Populated after a Data Retention action has run earlier in the chain.                                                                                                                                         |
| `.EventReports`                                    | array of per-user event report records | Populated after an Event Report action.                                                                                                                                                                       |
| `.ShareExpirationChecks`, `.ShareExpirationResult` | array / single                         | Populated after a Share Expiration Check action.                                                                                                                                                              |

**`.Object` typed accessors**

- `.Object.User` → `*User` (or nil). Fields include `.Username`, `.Email`, `.Status`, `.Role`, `.HomeDir`, `.UID`, `.GID`, `.MaxSessions`, `.QuotaSize`, `.QuotaFiles`, `.ExpirationDate`, `.LastLogin`, `.Permissions` (`map[string][]string`, e.g. `{{index .Object.User.Permissions "/"}}`), `.Filters`, `.Description`, `.AdditionalInfo`. Confidential data (password, public keys, TOTP secrets) is stripped.
- `.Object.Admin` → `*Admin`. Fields: `.Username`, `.Email`, `.Status`, `.Role`, `.Permissions`, `.Description`, `.AdditionalInfo`, `.Groups`. Confidential data stripped.
- `.Object.Group` → `*Group`. Fields: `.Name`, `.Description`, `.UserSettings`, `.VirtualFolders`.
- `.Object.Share` → `*Share`. Fields: `.ShareID`, `.Name`, `.Description`, `.Scope`, `.Paths`, `.Username`, `.CreatedAt`, `.UpdatedAt`, `.LastUseAt`, `.ExpiresAt`, `.UsedTokens`, `.MaxTokens`, `.AllowFrom`.
- `.Object.JSON` → the object's full JSON.

**`.Initiator` accessors**

`.Initiator.User` / `.Initiator.Admin` return `*User` / `*Admin` or `nil`. Same field surface as `.Object.User` / `.Object.Admin`. `.Initiator.JSON` returns `{}` for no-initiator events (cron, IP blocked, etc.).

**`.RetentionChecks[i]` fields**

`.Username`, `.Email` (`[]string`), `.ActionName`, `.Type` (`0` delete / `1` archive), `.Results`. Each `.Results[j]`: `.Path`, `.Retention` (hours, `int`), `.DeletedFiles` (`int`), `.DeletedSize` (`int64`), `.Elapsed` (`time.Duration`), `.Info`, `.Error`.

**`.EventReports[i]` fields**

`.Username`, `.Email` (`[]string`), `.Truncated` (`bool`), `.Events` (array). Each `.Events[j]`: `.ID`, `.Timestamp` (int64 nanos; use `fromNanos`), `.Action`, `.Username`, `.ExternalUsername`, `.VirtualPath`, `.VirtualTargetPath`, `.FileSize`, `.Elapsed`, `.Status`, `.Protocol`, `.IP`, `.SSHCmd`.

### 3.3 Template functions

In addition to standard `text/template` built-ins (`printf`, `len`, `index`, `slice`, `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `and`, `or`, `not`, `call`, `html`, `js`, `urlquery`), the following are registered:

**JSON / encoding**

- `toJson(val) string` — JSON-encode, quoting strings.
- `toJsonUnquoted(val) string` — for strings, JSON-escape without outer quotes; for others, same as `toJson`. Use inside pre-quoted JSON fields.
- `toBase64(s) string`, `toHex(s) string`.

**URL**

- `urlEscape(s) string` (= `url.QueryEscape`), `urlPathEscape(s) string` (= `url.PathEscape`).

**Path**

- `pathDir(p) string`, `pathBase(p) string`, `pathExt(p) string`.
- `pathJoin(elems []string) string` — slash-join, cleaned.
- `filePathJoin(elems []string) string` — OS-specific.

**String**

- `stringSlice(args ...string) []string`.
- `stringJoin(strs []string, sep string) string` — **slice first, separator second** (same as Go's `strings.Join`).
- `stringTrimSuffix(s, suffix) string`, `stringTrimPrefix(s, prefix) string`.
- `stringReplace(s, old, new) string` — **same arg order as Go's `strings.ReplaceAll`** (source, old, new).
- `stringHasPrefix`, `stringHasSuffix`, `stringContains` — `(s, sub) bool`.
- `stringToLower`, `stringToUpper` — `(s) string`.

**Slice / map**

- `slicesContains(slice []any, val any) bool`.
- `mapToString(key any, m map[any]string) string`.
- `createDict(entries ...any) map[any]string` — alternating key/value pairs.

**Time / bytes**

- `humanizeBytes(int64) string` — IEC units.
- `fromMillis(msec int64) time.Time`, `fromNanos(nsec int64) time.Time`.

`.Timestamp` is already `time.Time`; use `fromMillis` / `fromNanos` for nested numeric timestamps inside `.EventReports[i].Events[j].Timestamp`.

### 3.4 Action types (enum → sub-config)

The action's integer `type` field selects which sub-configuration is read from `options`. Only the sub-configuration matching `type` is active; all others are zeroed on save.

| `type` | Name                      | `options.*` field         | Template fields                                                                                                                  |
| ------ | ------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 1      | HTTP                      | `http_config`             | `endpoint`, `body`, `query_parameters[].value`, `headers[].value`, `parts[].filepath`, `parts[].body`, `parts[].headers[].value` |
| 2      | Command                   | `cmd_config`              | `args[]`, `env_vars[].value`                                                                                                     |
| 3      | Email                     | `email_config`            | `recipients[]`, `bcc[]`, `subject`, `body`, `attachments[]`                                                                      |
| 4      | Backup                    | (none)                    | —                                                                                                                                |
| 5–7    | Quota resets              | (none)                    | —                                                                                                                                |
| 8      | Data retention check      | `retention_config`        | (none)                                                                                                                           |
| 9      | Filesystem                | `fs_config`               | Depends on `fs_config.type` — see below                                                                                          |
| 11     | Password expiration check | `pwd_expiration_config`   | (none)                                                                                                                           |
| 12     | User expiration check     | (none)                    | —                                                                                                                                |
| 13     | IDP account check         | `idp_config`              | `template_user`, `template_admin`                                                                                                |
| 14     | User inactivity check     | `user_inactivity_config`  | (none)                                                                                                                           |
| 15     | Rotate logs               | (none)                    | —                                                                                                                                |
| 16     | IMAP                      | `imap_config`             | `email[]`, `subject`, `body`, `attachments[]`, `target_folder`, `flags[]`                                                        |
| 17     | ICAP                      | `icap_config`             | `paths[]`                                                                                                                        |
| 18     | Share expiration check    | `share_expiration_config` | (none)                                                                                                                           |
| 19     | Event report              | `event_report_config`     | (none — results surface via `.EventReports`)                                                                                     |

(`10` is reserved.)

**`fs_config.type` values (for `type=9`)**

| `fs_config.type` | Name           | Config field(s)                          | Template-rendered                                        |
| ---------------- | -------------- | ---------------------------------------- | -------------------------------------------------------- |
| 1                | Rename         | `renames[]` (`{key: old, value: new}`)   | both per entry                                           |
| 2                | Delete         | `deletes[]` (`[]string`)                 | each entry; **trailing `/` = delete contents, keep dir** |
| 3                | Mkdirs         | `mkdirs[]`                               | each                                                     |
| 4                | Exist          | `exist[]`                                | each                                                     |
| 5                | Compress       | `compress.paths[]`, `compress.name`      | all                                                      |
| 6                | Copy           | `copy[]` (`{key, value}`)                | both per entry                                           |
| 7                | PGP            | `pgp.paths[]` + key config               | paths                                                    |
| 8                | Metadata check | `folders[]`                              | none (literal names)                                     |
| 9                | Decompress     | `decompress.source`, `decompress.target` | both                                                     |

**`idp_config` (type 13)**

```json
{
  "mode": 0,                 // 0 = create/update, 1 = create only
  "template_user": "…",      // Go template rendering JSON User
  "template_admin": "…"      // Go template rendering JSON Admin
}
```

At least one of `template_user` / `template_admin` must be set. Licensing: **Premium tier only**.

Other Premium-tier-only action types: `13` (IDP), `17` (ICAP), `19` (Event Report). Type `16` (IMAP) is available on Starter as well.

### 3.5 Trigger types (what populates the context)

| `trigger` | Name                | Populates                                                                                                                                                                                                                                  | Typical use                                    |
| --------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| 1         | Filesystem event    | `.Name`, `.Event`, `.Status`, `.VirtualPath`, `.FsPath`, `.VirtualTargetPath`, `.FsTargetPath`, `.ObjectName`, `.ObjectType`, `.FileSize`, `.Elapsed`, `.Protocol`, `.IP`, `.Role`, `.Email`, `.Timestamp`, `.UID`, `.Metadata`, `.Errors` | Upload webhook, ICAP scan, compress on upload. |
| 2         | Provider event      | `.Name`, `.Event` (`add` / `update` / `delete`), `.ObjectName`, `.ObjectType`, `.Object`, `.Role`, `.Timestamp`, `.UID`, `.Initiator` (lazy `User` / `Admin` of who triggered the change)                                                  | Audit, CRM sync on user add.                   |
| 3         | Schedule (cron)     | `.Timestamp`, `.UID`. Chain-populated fields (e.g. `.RetentionChecks`, `.EventReports`, `.ShareExpirationChecks`) come from earlier actions.                                                                                               | Nightly retention report.                      |
| 4         | IP blocked          | `.IP`, `.Protocol`, `.Event`, `.Timestamp`, `.UID`, `.Errors`                                                                                                                                                                              | SOC alert.                                     |
| 5         | Certificate renewal | `.Event`, `.Timestamp`, `.UID`, `.Errors`                                                                                                                                                                                                  | Ops alert on ACME.                             |
| 6         | On demand           | `.Name` = rule name, `.Timestamp`, `.UID`. Otherwise empty until earlier chained actions populate fields.                                                                                                                                  | Manual triggers.                               |
| 7         | IDP login           | `.Name`, `.Protocol = "OIDC"`, `.IP`, `.Timestamp`, `.UID`, `.IDPFields`                                                                                                                                                                   | JIT user / admin provisioning.                 |

`fs_events` sub-filter (trigger 1): `upload`, `pre-upload`, `first-upload`, `download`, `pre-download`, `first-download`, `delete`, `pre-delete`, `rename`, `mkdir`, `rmdir`, `copy`, `ssh_cmd`.

`provider_events` sub-filter (trigger 2): `add`, `update`, `delete`.

`provider_objects` sub-filter: `user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_rule`, `event_action`.

`idp_login_event` sub-filter (trigger 7): `0` any, `1` user, `2` admin.

A template that references a field the trigger does not populate **renders an empty string**, not an error.

### 3.6 Template pitfalls

1. **Missing dot prefix** — `{{Name}}` is invalid; use `{{.Name}}`. Old snippets that omit the dot are wrong.
2. **`.IDPFields` only contains what `custom_fields` lists.** The OIDC binding's `custom_fields` array is the single source of what ends up in `.IDPFields` — SFTPGo does not auto-fold the role or email claims into it. The `role_field` setting drives JIT user/admin shape but is **not** copied to `.IDPFields` or to `.Role` on the event. If you need the role or email in your template, add the same claim name to `custom_fields` and read it via `index .IDPFields "claim_name"`. Missing claim → `nil`. This is the #1 OIDC template bug.
3. **Claim values can be strings or arrays.** When the IdP returns a JSON array (e.g. Entra ID `groups`, Keycloak realm roles), `.IDPFields.claim` is `[]any`. Use `slicesContains` rather than `eq` to test membership: `{{if slicesContains (index .IDPFields "groups") "sftp-admins"}}…{{end}}`.
4. **Always guard `.Initiator.User` / `.Initiator.Admin`.** `.Initiator` is never nil, but `.User` and `.Admin` can both be nil — for filesystem events, IP blocks, certificate events, schedules, and on-demand runs the initiator is unset. Guard with `{{if .Initiator.User}}…{{end}}` / `{{if .Initiator.Admin}}`.
5. **`.Object` is a lazy wrapper, not the object.** Use `{{.Object.User.Username}}` or `{{.Object.JSON}}` — not `{{.Object.Username}}`.
6. **JSON embedding.** Inside `"…"` positions: `{{.X | toJsonUnquoted}}`. In unquoted positions: `{{toJson .X}}`. Never hand-roll quoting.
7. **`stringReplace` order.** `stringReplace s old new` — source first, then old, then new (same as `strings.ReplaceAll`).
8. **`pathJoin` takes a `[]string`.** Build with `stringSlice`: `{{pathJoin (stringSlice "/base" .Name)}}`.
9. **Trailing-slash delete.** `fs_config.type=2` with a path ending in `/` empties the directory but keeps it; without `/` the directory is removed. Running the path through `pathJoin` strips the slash.
10. **`.FileSize` / `.Elapsed` are 0 on `pre-*` events.** Condition on `.Event` if a size threshold matters.
11. **Schedule triggers don't have `.VirtualPath`.** Cron-triggered rules see only `.Timestamp` + chained data. Restructure, don't fight it.
12. **"Compiles but webhook body is empty"** is almost always a trigger/field mismatch — cross-check every dotted field against the trigger's "Populates" list in §3.5.

---

## 4. Workflow for AI agents

When asked to produce SFTPGo artefacts:

1. **Check the edition and distribution first.** If the user is on the Open Source edition or a third-party redistribution (see §2.15 — GitHub `drakkan/sftpgo`, Docker Hub, managed-hosting providers, NAS / home-lab app stores, community Linux-package archives, etc.), most of this reference's Event Manager, OIDC, sharing, and UI-configuration content is Enterprise-only and does not apply. If on the fully-managed SaaS (see §2.14), no CLI / env-var / deployment recipes apply. In both cases, if detection is ambiguous, ask the user.
2. **Classify the question.** Is it a REST API payload, a WebAdmin configuration, a deployment/env-var task, or an Event Action template? (A single question can span more than one — e.g. "set up OIDC JIT" involves §1 + §2 + §3.)
3. **For REST API payloads** start from §1. If a specific field name or enum value is in doubt, look up the OpenAPI spec. Pay special attention to §1.2 (tagged unions) and §1.3 (secret envelope).
4. **For WebAdmin forms**: same shape as §1 JSON; paste the template string directly where the form accepts a template, paste scalar fields raw.
5. **For deployment questions** start from §2. The env-var rule is exact — lists of structs (§2.4) are the most common source of silently-ignored vars. Prefer `env.d/` over editing the shipped config file. Skip this entirely on SaaS — see §2.14.
6. **For templates**:
   a. Identify the trigger (`trigger` integer) from §3.5.
   b. Identify the action type (`type` integer) from §3.4.
   c. Pick fields from §3.2 that are populated for that trigger.
   d. Apply §3.6 guards systematically: dot prefix, nil-safe accessors, JSON-safe encoding.
   e. Produce two forms when unsure: raw template (WebAdmin paste) and full REST API JSON payload. Label them clearly.
7. **Cross-check** every referenced field against §3.2 / §3.5 before returning an answer. A field that is not in the trigger's populated set is a bug.
8. **When the question is operational / how-to** (e.g. "how do I enable 2FA", "how do I set up the LDAP plugin", "how do I configure S3 authentication"), fetch the corresponding page from the public docs (§0) — they contain the operational walkthroughs this reference does not reproduce.

---

**Reference version:** v0.5.0 · verified against SFTPGo v2.7.x · upstream <https://sftpgo.com>
