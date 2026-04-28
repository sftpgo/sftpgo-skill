# SFTPGo deployment & configuration — for system administrators

This file covers the tasks an AI agent is most likely asked to help with when deploying, configuring, and maintaining an SFTPGo installation: picking the install method, editing the configuration file, producing the right environment variables, and avoiding the common traps.

The authoritative references are the public docs. Fetch them by raw URL when details matter:

- Installation: <https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md>
- Configuration file (every parameter): <https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/config-file.md>
- Environment variables: <https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/env-vars.md>
- Docker: <https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/docker.md>

---

## 1. Installation — pick a method

| Target                           | Method                                                                                                                                                                           |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Debian / Ubuntu                  | APT repo `https://download.sftpgo.com/apt` — installs the `sftpgo` package and registers a systemd unit.                                                                         |
| RHEL / CentOS / Rocky / SUSE     | YUM / zypper repo `https://download.sftpgo.com/yum/<arch>` — same package outcome.                                                                                               |
| Windows                          | Signed Windows installer from `https://download.sftpgo.com/windows/sftpgo_windows_x86_64.exe` (the official method — registers a service, default config dir `C:\ProgramData\SFTPGo Enterprise`). For hosts that have winget, `winget install -e --id drakkan.SFTPGoEnterprise` is an alternative.  |
| Docker                           | Official registry (license required for Enterprise). See Docker doc.                                                                                                             |
| Kubernetes                       | Official Helm chart.                                                                                                                                                             |
| AWS / Azure / Google Cloud       | Marketplace listings (Starter / Premium). License is bundled with the subscription.                                                                                              |

Without a valid license SFTPGo runs in **limited mode** with the following restrictions:

- Concurrent transfers are capped at 2.
- Plugins are disabled (count = 0).
- Only the local filesystem is available as a storage backend.
- Event Manager actions PGP, ICAP, Identity Provider account check, and Event report are unavailable.

The commercial licensed tiers (used both on-premises and on cloud marketplaces) are **Starter**, **Premium**, and **Ultimate** — see `https://sftpgo.com/on-premises` for the feature matrix.

**Cloud marketplace offerings** (AWS, Azure, GCP) ship preconfigured: license activated, audit-logs enabled, in-memory transfer pipes on, data provider initialized for standalone use. Audit-logs is implemented as **two cooperating plugins** (`eventstore` for write/persist, `eventsearcher` for query) — both must be loaded for the audit-logs UI to function. The Starter marketplace tier allows **2 active plugins total**, which audit-logs already fills, so on Starter there is no spare slot for a third plugin without dropping audit-logs or moving up a tier. Premium and higher tiers have no plugin limit. See the "What is preconfigured on marketplace offerings" section of [installation.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md) for the full list.

**Marketplace OS images and updates.** AWS AMIs (x86_64 and arm64) are based on **Amazon Linux 2023** (RHEL-based, `dnf`). Azure and Google Cloud Linux VMs are based on **Debian 13** (`apt`). Both SFTPGo and the OS are kept current through the distribution's standard package manager:

```shell
sudo dnf update                          # AWS — Amazon Linux
sudo apt update && sudo apt upgrade      # Azure / GCP — Debian
```

The SFTPGo service is restarted automatically by the package's post-install scriptlet when the `sftpgo` package is upgraded — no manual `systemctl restart` is required.

Check the running SFTPGo version with `sftpgo --version`; the same version string is also logged at service startup (`starting SFTPGo v2.7.x ...`, visible via `journalctl -u sftpgo` or in the configured log file). Marketplace offerings hide the version in the WebAdmin by default, so on those instances the CLI and the startup logs are the canonical sources. Container-based marketplace listings have their own update procedure documented on each listing page: the AWS Container listing distributes images through the AWS marketplace registry and is updated by deploying the new image version published by the listing; the Azure AKS listing is a CNAB bundle and is upgraded through its CNAB mechanism. The generic [docker.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/docker.md) covers self-hosted containers from `registry.sftpgo.com` — it does not apply to the marketplace container listings, which use the marketplace's own registry / packaging.

**Marketplace pricing — what to tell customers.** All marketplace listings — both Enterprise and open-source-based — are billed entirely through the cloud provider; there is no separate fee from SFTPGo on top of the marketplace charge. Enterprise listings include an activated SFTPGo Enterprise license (nothing else to purchase or activate); open-source-based listings ship the SFTPGo Open Source edition (AGPLv3) and do not enable Enterprise features. To access Enterprise features, the customer switches to one of the Enterprise listings. Answer pricing questions in terms of what the customer pays for and what they get — do not speculate about how the marketplace charge is split between compute and software. Authoritative reference: the "Pricing and licensing" callout in [installation.md `## Commercial Marketplaces`](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md).

### Post-install sanity checklist

1. Confirm the service is running: `systemctl status sftpgo` (Linux) / `Get-Service sftpgo` (Windows).
2. Create the first admin (WebAdmin shows an install-code prompt on first access if one was configured; otherwise it opens a create-admin flow).
3. Configure the data provider — embedded SQLite is fine for dev / single-node; use PostgreSQL, MySQL/MariaDB, or CockroachDB for production / HA.
4. Put the license key in place: via WebAdmin (Server Manager → License) or by setting `SFTPGO_LICENSE_KEY=XXXX-XXXX-XXXX-XXXX`.

---

## 2. Where the config file lives

The binary looks for a file named `sftpgo.<ext>` inside the configured *config directory* (and secondary search paths). Supported extensions are those of [viper](https://github.com/spf13/viper) — **JSON**, **YAML**, **TOML**, **HCL**, **Java properties**, **envfile**. The default shipped file is `sftpgo.json`.

| Platform                 | Default config dir                  |
| ------------------------ | ----------------------------------- |
| Debian / RPM (systemd)   | `/etc/sftpgo`                       |
| Docker image             | `/var/lib/sftpgo` (override with `-c`) |
| Windows installer        | `C:\ProgramData\SFTPGo Enterprise`  |

The config directory can be overridden with the `-c <path>` flag on every `sftpgo` command (`serve`, `initprovider`, `service install`, …). On Linux the packaged systemd unit sources `/etc/sftpgo/sftpgo.env` as `EnvironmentFile` and loads every regular file under `/etc/sftpgo/env.d/` before starting. On Windows the service reads every regular file from `C:\ProgramData\SFTPGo Enterprise\env.d\` at startup. The `.env` suffix used in examples is only a convention — the loader accepts any file name.

**Recommendation**: do not edit the shipped `sftpgo.json`. Keep it at defaults and put all your changes in env files under `env.d/` — this way, package upgrades that ship a new default config do not conflict with your edits. The same applies on Windows: use `C:\ProgramData\SFTPGo Enterprise\env.d\` rather than rewriting `sftpgo.json`.

**File config vs WebAdmin UI.** Several areas are exposed in both the file config (env vars / `sftpgo.json`) and the WebAdmin UI — TLS certificates, ACME, SMTP, branding, OIDC, LDAP, Geo-IP, and more. **Prefer the UI when it is available**: it persists the configuration in the database (encrypted when appropriate), is replicated across HA nodes automatically, and avoids having to restart / reload the service for changes to take effect. Reserve file-based config for things the UI does not cover (bindings, transport-level options, hooks, memory pipes, licensing) or for headless / scripted deployments.

**Two Configurations sections are hidden behind opt-in env-var toggles** (the underlying features are built into SFTPGo — the toggle just exposes the form):

| WebAdmin section                       | Required env var                |
| -------------------------------------- | ------------------------------- |
| OIDC                                   | `SFTPGO_HOOK__ENABLE_OIDC_UI=1` |
| TLS certificates (default key pair)    | `SFTPGO_HOOK__ENABLE_TLS_UI=1`  |

These are env-var-only flags (no UI / API equivalent) and they take effect at process startup, so after setting one you must restart SFTPGo for the section to appear. On marketplace and SaaS offerings the relevant flags are already enabled. Both are documented in the public [env-vars.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/env-vars.md).

LDAP and Geo-IP are plugin-based integrations: a separate plugin binary plus a plugin slot under the license are required, so the WebAdmin Configurations entries for them are wired to plugin setup rather than to a self-contained toggle. Refer the user to the [LDAP plugin guide](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/plugins/ldap-auth.md) and the Geo-IP plugin docs for the full setup; the SaaS / marketplace offerings come with these wired up already.

A notable case: **OIDC**. The WebAdmin UI stores a single OIDC configuration in the database; the OIDC redirect URL is one URL registered with the IdP, so the OIDC flow is served by the listener whose hostname matches the configured `redirect_base_url` — typically the primary one. To configure OIDC per HTTPD binding (different IdPs on different listeners, e.g. an internal-only WebAdmin OIDC plus a customer-facing WebClient OIDC), use file or env-var config under `httpd.bindings[N].oidc` — the UI cannot express that.

---

## 3. Environment variable convention

This is the single rule an AI agent must get right — the binary silently ignores env vars whose names don't match the expected shape.

### 3.1 The naming rule

```text
SFTPGO_<SECTION>__<KEY>[__<SUBKEY>…]
```

- Prefix: `SFTPGO_` (one underscore).
- Separator between struct levels: **double** underscore `__`.
- Keys are uppercased versions of the JSON keys, e.g. `defender` → `DEFENDER`.

Examples:

| JSON path                       | Env var                                |
| ------------------------------- | -------------------------------------- |
| `common.defender.enabled = true`      | `SFTPGO_COMMON__DEFENDER__ENABLED=true`   |
| `webdavd.cors.enabled = true`         | `SFTPGO_WEBDAVD__CORS__ENABLED=true`      |
| `data_provider.driver = "postgresql"` | `SFTPGO_DATA_PROVIDER__DRIVER=postgresql` |
| `telemetry.enable_profiler = true`    | `SFTPGO_TELEMETRY__ENABLE_PROFILER=true`  |

### 3.2 Lists of scalars

Comma-separate the values:

```shell
SFTPGO_COMMON__ACTIONS__EXECUTE_ON=upload,download
SFTPGO_COMMON__EVENT_MANAGER__ENABLED_COMMANDS=/usr/bin/touch,/usr/bin/mkdir
```

### 3.3 Lists of structs — the main trap

Index each item by position, then walk into its fields. Bindings, command entries, plugins — all use this shape.

```shell
# sftpd.bindings[0].port = 22
SFTPGO_SFTPD__BINDINGS__0__PORT=22
SFTPGO_SFTPD__BINDINGS__0__ADDRESS=0.0.0.0

# A second SFTP binding on 2022
SFTPGO_SFTPD__BINDINGS__1__PORT=2022

# command.commands[0] = {path: "/usr/bin/ping", args: ["-c5","host"]}
SFTPGO_COMMAND__COMMANDS__0__PATH=/usr/bin/ping
SFTPGO_COMMAND__COMMANDS__0__ARGS=-c5,example.com
```

### 3.4 Secrets and special characters

Values containing `$`, spaces, or backslashes need to be quoted. The env-file parser follows shell rules:

```shell
# single quotes: everything literal
SFTPGO_DATA_PROVIDER__PASSWORD='my$secret\pwd'

# double quotes: $ expands, backslash escapes
SFTPGO_DATA_PROVIDER__PASSWORD="my\$secret\\pwd"
```

---

## 4. Where env vars are read from

In order of precedence:

1. **Process environment** — set by the init system, `docker run -e`, `kubectl set env`, or shell export. On Windows the service reads its environment from the Service Control Manager (so `sc`, `Set-Service`, or a registry `Environment` multi-string under `HKLM\SYSTEM\CurrentControlSet\Services\sftpgo` all feed this layer).
2. **Files in `<config-dir>/env.d/`** — every regular file in the directory is parsed at SFTPGo startup, regardless of extension (the `.env` suffix used below is only a convention for editor syntax highlighting — files with no extension, `.conf`, `.txt`, or anything else work the same way). Applies on **all platforms** (Linux, Windows, macOS). Values already set in the process environment **win** (env.d files do not override existing env).
3. **`EnvironmentFile=/etc/sftpgo/sftpgo.env`** *(Linux only)* — loaded by systemd *before* the binary starts, so it behaves like process env (takes precedence over env.d).

For Docker: pass env vars with `-e SFTPGO_SFTPD__BINDINGS__0__PORT=22` or mount a `.env` file.

For Kubernetes: use `env:` / `envFrom:` on the Deployment, or mount a Secret / ConfigMap as an env file into `env.d/`.

For package installs on Linux, **prefer putting changes in `env.d/`** rather than in `sftpgo.env`: multiple small files (one per concern) are easier to review, promote across environments, and remove when no longer needed. The same advice applies on Windows — `env.d` files work identically there and are the only cross-platform way to ship config changes.

Example layout (Linux):

```text
/etc/sftpgo/env.d/
├── 10-bindings.env          # SFTPGO_SFTPD__BINDINGS__…, SFTPGO_HTTPD__BINDINGS__…
├── 20-data-provider.env     # SFTPGO_DATA_PROVIDER__…
├── 30-telemetry.env         # SFTPGO_TELEMETRY__…
├── 40-acme.env              # SFTPGO_ACME__…
└── 90-plugins.env           # SFTPGO_PLUGINS__…
```

Equivalent on Windows:

```text
C:\ProgramData\SFTPGo Enterprise\env.d\
├── 10-bindings.env
├── 20-data-provider.env
├── 30-telemetry.env
├── 40-acme.env
└── 90-plugins.env
```

Reload after editing:

- Linux: `sudo systemctl restart sftpgo`
- Windows (PowerShell, elevated): `Restart-Service sftpgo`

---

## 5. Production recipes

Small, copy-pasteable env blocks for the most common asks. Each can go into its own `env.d/<name>.env`.

### 5.1 Bind SFTP on port 22

```shell
SFTPGO_SFTPD__BINDINGS__0__ADDRESS=
SFTPGO_SFTPD__BINDINGS__0__PORT=22
```

Port 22 is privileged on Linux; the `sftpgo` systemd unit already grants `CAP_NET_BIND_SERVICE`. On Windows the service runs under `LOCAL SYSTEM` by default and can bind low ports without special grants; if you changed the service account to a domain user, make sure it retains the right to bind privileged ports.

### 5.2 PostgreSQL data provider

```shell
SFTPGO_DATA_PROVIDER__DRIVER=postgresql
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST=db.internal
SFTPGO_DATA_PROVIDER__PORT=5432
SFTPGO_DATA_PROVIDER__USERNAME=sftpgo
SFTPGO_DATA_PROVIDER__PASSWORD='your-password'
SFTPGO_DATA_PROVIDER__SSLMODE=1
```

No manual schema step is needed. With the default `data_provider.update_mode = 0`, SFTPGo creates the schema on first startup and applies any pending migrations on every subsequent startup — just create the empty database and restart the service.

Run `sftpgo initprovider` only if you set `update_mode = 1` to disable automatic init/migration (for example, when the runtime DB credentials are intentionally restricted and have no DDL privileges, and you apply schema changes from an ops role under human control):

- Linux: `sftpgo initprovider -c /etc/sftpgo`
- Windows (elevated PowerShell): `sftpgo initprovider -c "C:\ProgramData\SFTPGo Enterprise"`

### 5.3 HTTPS on the WebAdmin + WebClient with a static cert

**Recommended: WebAdmin UI.** In recent Enterprise versions, TLS certificates (HTTPS, FTPS, WebDAV) are managed from **Server Manager → Configurations → TLS Certificates**. Paste or upload the cert + key and SFTPGo stores them encrypted in the database. The UI is preferred because the configuration persists across restarts and is replicated in HA clusters automatically — no filesystem paths, no reload command, no env vars.

For file-based / headless deployments that prefer on-disk certs:

```shell
SFTPGO_HTTPD__BINDINGS__0__PORT=8080
SFTPGO_HTTPD__BINDINGS__0__ENABLE_HTTPS=true
SFTPGO_HTTPD__BINDINGS__0__CERTIFICATE_FILE=/etc/sftpgo/tls/fullchain.pem
SFTPGO_HTTPD__BINDINGS__0__CERTIFICATE_KEY_FILE=/etc/sftpgo/tls/privkey.pem
```

On Windows use full paths under `ProgramData`:

```shell
SFTPGO_HTTPD__BINDINGS__0__CERTIFICATE_FILE=C:\ProgramData\SFTPGo Enterprise\tls\fullchain.pem
SFTPGO_HTTPD__BINDINGS__0__CERTIFICATE_KEY_FILE=C:\ProgramData\SFTPGo Enterprise\tls\privkey.pem
```

When you replace the cert files on disk (e.g. after obtaining a new certificate out of band), reload the running service so it re-reads them without a full restart:

- Linux: `sudo systemctl reload sftpgo` (sends `SIGHUP`).
- Windows (PowerShell, elevated): `sftpgo service reload` — sends a `paramchange` request.

### 5.4 HTTPS via built-in ACME (Let's Encrypt)

**Recommended: WebAdmin UI.** Navigate to **Server Manager → Configurations → ACME** and fill the form: email, domain, selected protocols (HTTPS / FTPS / WebDAV), HTTP-01 port. On save, SFTPGo obtains the certificate and stores it encrypted in the database; renewal is then automatic, managed by the running service. This is the preferred path — no CLI command, no file-based config, HA-replicated.

For headless / scripted deployments you can drive ACME from file-based config (`sftpgo.json` → `acme.*`, or env vars):

```shell
SFTPGO_ACME__EMAIL=ops@example.com
SFTPGO_ACME__KEY_TYPE=4096
SFTPGO_ACME__DOMAINS=sftp.example.com
SFTPGO_ACME__HTTP01_CHALLENGE__PORT=80
```

`key_type` accepts `2048`, `3072`, `4096`, `8192` (RSA) or `P256` / `P384` (ECDSA). With file-based config you must then **run the one-shot command** to register the ACME account and obtain the first certificate — this writes it to the database and activates automatic renewal from then on:

```shell
sftpgo acme run --database-protocols 7
```

`--database-protocols` is a bitmask: `1` HTTPS, `2` FTPS, `4` WebDAV (combinable; `7` = all). Use HTTP-01 when port 80 is reachable; use DNS-01 when it isn't.

### 5.5 Enable the Defender (intrusion detection)

```shell
SFTPGO_COMMON__DEFENDER__ENABLED=true
SFTPGO_COMMON__DEFENDER__DRIVER=provider
SFTPGO_COMMON__DEFENDER__BAN_TIME=60
SFTPGO_COMMON__DEFENDER__THRESHOLD=15
```

Use `driver=provider` in multi-node setups so the ban list is shared; `memory` is faster on single-node.

### 5.6 Expose metrics + enable the profiler (internal network only)

```shell
SFTPGO_TELEMETRY__BIND_ADDRESS=127.0.0.1
SFTPGO_TELEMETRY__BIND_PORT=10000
SFTPGO_TELEMETRY__ENABLE_PROFILER=true
SFTPGO_TELEMETRY__AUTH_USER_FILE=/etc/sftpgo/telemetry.htpasswd
```

Port `10000` is the convention used by the official Helm chart (hardcoded there, not user-configurable) and by all documentation examples — prefer it for consistency even on non-K8s deployments.

See `profiling.md` in the public docs for the `/debug/pprof` endpoints and `metrics.md` for Prometheus scraping.

### 5.7 HAProxy PROXY protocol

```shell
SFTPGO_COMMON__PROXY_PROTOCOL=2
SFTPGO_COMMON__PROXY_ALLOWED=10.0.0.0/8,192.168.0.0/16
```

Mode `2` is strict: connections from IPs **not** in `proxy_allowed` that carry a proxy header are rejected. Mode `1` is permissive (unlisted → header ignored).

### 5.8 License key

```shell
SFTPGO_LICENSE_KEY=XXXX-XXXX-XXXX-XXXX
```

### 5.9 Kubernetes — shape and pitfalls

Full shape is in [docker.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/docker.md) (`## Kubernetes` section). Chart: `oci://ghcr.io/sftpgo/helm-charts/sftpgo`.

The chart is **edition-agnostic** — its default `image.repository` is `ghcr.io/drakkan/sftpgo` (Open Source). For Enterprise, override `image.repository` (typically `registry.sftpgo.com/sftpgo/sftpgo`, with the `-plugins` / `-distroless` / `-distroless-plugins` variant you need) and add the license key as a Kubernetes `Secret` referenced via `envFrom`. Everything else in this section applies to both editions.

The SFTPGo-specific decisions an AI agent must get right:

- **Expose SFTP via Service `LoadBalancer` / `NodePort`, not Ingress.** SFTP, FTP, and WebDAV are plain TCP — a standard HTTP Ingress controller cannot forward them. The HTTP WebAdmin / WebClient *can* sit behind an Ingress. On clouds: AWS NLB, Azure `standard` LB with TCP, GCP TCP LB. An NGINX Ingress with [TCP services ConfigMap](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/) can also forward raw TCP.
- **Multi-replica needs external DB.** SQLite (bolt, default for distroless) only works single-replica. For `replicaCount > 1`, point at PostgreSQL / MySQL / CockroachDB via Secret-backed env vars. Remember the CockroachDB migration constraint (no `pg_advisory_lock` — serialize startups or use `update_mode=1`).
- **Persistence depends on backend mix.** Local user homes need a PVC; `ReadWriteOnce` is fine for single-replica, `ReadWriteMany` (EFS, Azure Files, Filestore) for multi-replica shared storage. Cloud-backend-only deployments can run with ephemeral storage if memory pipes are enabled.
- **Memory pipes mandatory on ephemeral / read-only roots.** Distroless pods, pods without a PVC, or pods running on memory-backed `emptyDir` must set `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1` or cloud uploads fail. Marketplace AKS / EKS offerings have this preconfigured.
- **Health probes.** The chart configures liveness/readiness automatically against `/healthz` on the telemetry port (`10000`, fixed). No manual probe config needed. The same port serves Prometheus metrics at `/metrics` and pprof at `/debug/pprof` — do not bind probes to the HTTPD port, which requires auth.
- **License key.** Deliver via Secret (`stringData.SFTPGO_LICENSE_KEY`) referenced by `envFrom`, never in the values.yaml committed to git.
- **Shared SSH host keys.** Without explicit configuration each replica generates its own host keys on first boot, which triggers client-side mismatch warnings as sessions land on different pods. Two ways to share one set:
  - **Paste the private keys in the WebAdmin** under Server Manager → Configurations → SFTP. Keys are stored encrypted in the data provider and loaded identically by every replica that shares the DB. This also removes host-key filesystem management on Linux and Windows installs and is the preferred approach for new deployments.
  - **Mount a Kubernetes Secret** containing the private keys as a volume and point SFTPGo at the files via `SFTPGO_SFTPD__HOST_KEYS` (comma-separated list of file paths). The chart exposes `volumes:` / `volumeMounts:` extension points for this — see the example below. Useful with external secret managers or when you keep keys out of the database.

Minimal cloud-ready values-fragment (distroless + S3 users + external Postgres):

```yaml
image:
  repository: registry.sftpgo.com/sftpgo/sftpgo
  tag: v2.7.x-distroless-plugins

replicaCount: 2

# Simplified key-value env (the chart also supports the full `envVars:` list form)
env:
  SFTPGO_HOOK__MEMORY_PIPES__ENABLED: "1"
  SFTPGO_DATA_PROVIDER__DRIVER: postgresql
  SFTPGO_DATA_PROVIDER__HOST: postgres.shared.svc
  SFTPGO_DATA_PROVIDER__NAME: sftpgo

# envFrom injects every key of the Secret as a container env var, so the keys
# must be named exactly as the SFTPGo env vars the server expects — here
# SFTPGO_LICENSE_KEY and SFTPGO_DATA_PROVIDER__PASSWORD.
envFrom:
  - secretRef:
      name: sftpgo-secrets

# Default service stays ClusterIP — used by the Ingress to reach httpd inside
# the cluster. For public SFTP, expose it through an additional LoadBalancer
# service (the chart's `services:` plural key).
services:
  public-sftp:
    type: LoadBalancer
    ports:
      sftp:
        port: 22

# Expose the WebAdmin / WebClient via Ingress
ui:
  ingress:
    enabled: true
    className: nginx
    hosts:
      - host: sftpgo.example.com
        paths:
          - path: /
            pathType: ImplementationSpecific

persistence:
  enabled: false    # S3-only, no local homes → no PVC needed
```

**Variant — host keys mounted from a Secret**. Create the Secret separately (e.g. `kubectl create secret generic sftpgo-host-keys --from-file=id_rsa=... --from-file=id_ecdsa=...`) and wire it into the pod via the chart's `volumes:` / `volumeMounts:` / `envVars:` extension points:

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

# Use the list form (not `env:`) when values need to reference files at a known
# path — here SFTPGO_SFTPD__HOST_KEYS points at the mounted files.
envVars:
  - name: SFTPGO_SFTPD__BINDINGS__0__PORT
    value: "2022"
  - name: SFTPGO_SFTPD__HOST_KEYS
    value: "/etc/sftpgo/ssh-keys/id_rsa,/etc/sftpgo/ssh-keys/id_ecdsa"
```

This produces the rendered pod spec:

```yaml
      containers:
        - env:
            - name: SFTPGO_SFTPD__BINDINGS__0__PORT
              value: "2022"
            - name: SFTPGO_SFTPD__HOST_KEYS
              value: "/etc/sftpgo/ssh-keys/id_rsa,/etc/sftpgo/ssh-keys/id_ecdsa"
          volumeMounts:
            - name: ssh-keys
              mountPath: /etc/sftpgo/ssh-keys
              readOnly: true
      volumes:
        - name: ssh-keys
          secret:
            secretName: sftpgo-host-keys
            optional: false
```

Alternative for new deployments — paste the key PEMs into **Server Manager → Configurations → SFTP** in the WebAdmin and skip the volume mount entirely.

### 5.10 Docker — production shape

A minimal `docker run` with persistent volumes, SFTP on port 22, a license key, and memory pipes enabled (safe default even for non-cloud deployments):

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

Host directories must be writable by UID `1000` (the image's default non-root user): `sudo chown -R 1000:1000 /srv/sftpgo-data /srv/sftpgo-config`. No actual Linux user with UID 1000 is required on the host.

`docker-compose.yml` equivalent with env.d overrides (one file per concern):

```yaml
services:
  sftpgo:
    image: registry.sftpgo.com/sftpgo/sftpgo:<tag>
    restart: unless-stopped
    ports:
      - "22:22"
      - "8080:8080"
    environment:
      SFTPGO_LICENSE_KEY: ${SFTPGO_LICENSE_KEY}
      SFTPGO_HOOK__MEMORY_PIPES__ENABLED: "1"
    volumes:
      - ./data:/srv/sftpgo
      - ./config:/var/lib/sftpgo
      - ./env.d:/var/lib/sftpgo/env.d:ro   # dropped env files read at startup
```

With `./env.d/10-bindings.env`:

```shell
SFTPGO_SFTPD__BINDINGS__0__PORT=22
SFTPGO_HTTPD__BINDINGS__0__PORT=8080
```

After editing `env.d/*.env`, reload with `docker compose restart sftpgo`.

For a **distroless** or **distroless-plugins** image on a cloud-only deployment, always set `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1` — the image has no writable scratch space beyond the mounted volumes.

### 5.11 Memory pipes for cloud backends

On cloud storage backends (S3, GCS, Azure Blob) SFTPGo needs a writable local directory for per-transfer temporary files used to stream data between the client and the backend. When the host has no usable local filesystem — read-only containers (distroless, scratch images), Kubernetes pods with ephemeral emptyDir memory-backed volumes, deployments where `temp_path` is on a full or non-writable partition — transfers fail before any bytes reach the backend.

Enable fully in-memory transfers so no temp file is created:

```shell
SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1
```

This is the standard configuration for cloud-only deployments and is the default on SFTPGo marketplace offerings. The trade-off is higher RAM per concurrent transfer (a buffer per upload); on hosts with limited memory and many concurrent uploads, file-backed pipes may still be preferable.

---

## 6. Pitfalls

- **Local home_dir not writable by the `sftpgo` system user.** Symptoms: user logs in successfully but gets "permission denied" on the first upload, or `MKDIR` fails. Cause: the APT / YUM packages create a dedicated `sftpgo` system account and run the service under it — a `home_dir` outside `/srv/sftpgo` (the packaged `users_base_dir`) that you created as `root` is unreadable/unwritable by `sftpgo`. Fix: make the path and its parents traversable for the `sftpgo` user, e.g.

  ```shell
  sudo mkdir -p /data/sftpgo/alice
  sudo chown -R sftpgo:sftpgo /data/sftpgo
  sudo chmod 755 /data            # parents need +x so sftpgo can traverse
  ```

  On Windows, grant the service account (default `LOCAL SYSTEM`) read/write on the chosen home-dir tree via the directory properties or `icacls`.
- **Avoid setting `common.temp_path`.** Leave it empty unless you have a specific reason. If set to a path on a different filesystem from the user home directories, atomic renames fall back to copy-then-delete — slower and non-atomic. The default (empty) is the right choice for the vast majority of deployments. The setting is on a deprecation path; new deployments should not adopt it.
- **Cloud backend uploads fail on a read-only / ephemeral host.** Symptoms: "unable to open pipe", "read-only file system", or "no space left" shortly after the client starts uploading to an S3 / GCS / Azure Blob user. Cause: SFTPGo tries to create a file-backed pipe in the configured temp dir and cannot. Fix: enable memory pipes — `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1` (see §5.11). Strongly recommended for distroless containers, read-only rootfs, and Kubernetes pods without a writable PVC; already the default on marketplace offerings. Memory pipes are an Enterprise feature; on the OSS build file-backed pipes are the only option, so make sure the temp dir is on writable storage with enough headroom.
- **Proxy mode `2` without populated allow-list.** Setting `proxy_protocol=2` rejects every connection until `proxy_allowed` contains your load balancer's IP / CIDR. Configure both in the same deploy step.
- **`allowlist_status=1` before populating the list.** Enabling it empty drops everything. Populate the allow list (via WebAdmin or REST API) first, then flip the flag.
- **Changing tagged-union fields without the sub-config.** Switching `data_provider.driver` from `sqlite` to `postgresql` requires setting `host`, `port`, `username`, `password`, `name`, `sslmode` in the same change — otherwise the server starts but cannot reach the database.
- **Running `initprovider` unnecessarily.** It is not a required first-boot step. With `update_mode = 0` (the default) SFTPGo initializes and migrates the schema on every startup. Only flip to `update_mode = 1` if the runtime credentials lack DDL privileges, and run `initprovider` from an ops role before each upgrade.
- **Concurrent migrations across multiple SFTPGo instances.** PostgreSQL serializes them via `pg_advisory_lock`; concurrent SFTPGo startups against the same PostgreSQL cluster are safe. MySQL/MariaDB and CockroachDB don't expose an equivalent advisory-lock primitive that SFTPGo can rely on, and MySQL/CockroachDB also can't wrap DDL in a transaction (so a forcibly-aborted migration can leave the schema in an intermediate state). For those engines, coordinate startup from the operator side — either start instances sequentially on first deploy and on every version bump, or set `update_mode = 1` everywhere and run `sftpgo initprovider` from a single one-shot job (CI step, Helm post-upgrade hook, …) before rolling out.
- **MySQL transaction isolation.** MySQL/MariaDB default to `REPEATABLE READ`, which uses gap locks on index ranges. Under high write concurrency (a wave of bulk user updates, sync jobs hitting tens of users in parallel) this surfaces as `Deadlock found when trying to get lock; try restarting transaction` in the SFTPGo logs. Set `transaction-isolation = READ-COMMITTED` in `my.cnf` and restart MySQL — that removes the gap-lock contention. PostgreSQL has no such issue at the default isolation level.
- **env.d files do not override the process environment.** If a value is already exported (systemd `EnvironmentFile`, container env), editing env.d has no effect. Move the setting to the authoritative source, or clear the process-level value first.
- **Package upgrade merging.** Upstream ships default `sftpgo.json`; `dpkg`/`rpm` may prompt about your local edits. Keeping your overrides in `env.d/` entirely avoids the conflict.
- **TLS file-based cert reload latency.** After replacing `certificate_file` / `certificate_key_file` on disk, the in-memory keys stay stale until the built-in 8-hour file watcher (size + mtime check) picks the new files up. To force an immediate reload use `systemctl reload sftpgo` (Linux) / `sftpgo service reload` (Windows — sends `paramchange`). Certs managed via the WebAdmin UI or refreshed by the built-in ACME client are reloaded immediately on save / renewal — no action needed.
- **List-of-struct indexes are dense.** `SFTPGO_SFTPD__BINDINGS__0__PORT` and `…__2__PORT` without a `__1__` leaves a gap — the binary will not assume index 1 exists. Use contiguous indices.
- **`disabled` on bindings.** To turn off a binding set its port to `0`; do not rely on a phantom `disabled` flag.
- **Log file rotation.** The binary rotates by size; if you use `logrotate` with `copytruncate` you'll occasionally lose a few lines. Prefer `postrotate` + `SIGHUP` via the SFTPGo systemd unit's own rotation, or disable external rotation. On Windows rotation is signalled with `sftpgo service rotatelogs`.
- **Windows service account.** The installer registers the service as `LOCAL SYSTEM`. If you change the account (via `sftpgo service uninstall` + `sftpgo service install --service-user …` or the Services UI), make sure the new account can read the config directory, the license key, and the home directories — and, if you bind privileged ports, retains the right to do so.
- **Windows path escaping in env files.** Use literal backslashes in single-quoted values (`'C:\path'`) or double-escaped (`"C:\\path"`) in double-quoted values. Forward slashes also work and sidestep the escaping question (`C:/ProgramData/SFTPGo Enterprise/tls/fullchain.pem`).
- **Secret leak in logs.** SFTPGo redacts secrets in its own startup log (`config file used: '…', config loaded: {…}`), but it cannot redact what systemd echoes. Do **not** use `systemd-cat`/`journalctl` `LogLevel=debug` on the unit in a way that dumps env files.

---

## 7. Operational recipes

### 7.1 Reset a lost administrator password

Run on the host where SFTPGo is installed (the command loads the data provider directly, no HTTP call):

```shell
sudo sftpgo resetpwd --admin <username>                       # Linux
"C:\Program Files\SFTPGo\sftpgo.exe" resetpwd --admin <username> -c "C:\ProgramData\SFTPGo Enterprise"   # Windows
```

The command prompts for the new password (plus confirmation) and also **disables 2FA on that admin** so the account is immediately usable — the admin can re-enable 2FA from the WebAdmin after signing in.

- Not supported for the `memory` data provider.
- **Stop the service first** when using an embedded provider (`bolt` or `sqlite`) — they are single-writer and the command can corrupt the DB if SFTPGo is running against it.
- Safe to run live when using a shared provider (PostgreSQL, MySQL, CockroachDB).
- Pass `-c <config-dir>` (or set `SFTPGO_CONFIG_DIR`) if the config directory isn't the current working directory.

### 7.2 Rotate logs on demand

- Linux: `sudo kill -USR1 $(pgrep -x sftpgo)` (or the systemd-unit PID). When an external `logrotate` is used, prefer `postrotate` with the same `SIGUSR1` signal over `copytruncate` to avoid losing lines.
- Windows: `sftpgo service rotatelogs`.

### 7.3 Reload TLS certificates after replacing files on disk

Only applicable when certs are managed as files (`certificate_file` / `certificate_key_file` on bindings). Certs stored in the database (WebAdmin UI or ACME) are reloaded automatically on save / renewal — no manual step.

- Linux: `sudo systemctl reload sftpgo` (sends `SIGHUP`).
- Windows: `sftpgo service reload` (sends a `paramchange` request).

Reload picks up cert / key files only; full config changes need a restart.

### 7.4 Dump / restore data

Use the REST API `POST /api/v2/dumpdata` and `POST /api/v2/loaddata` (or the WebAdmin Maintenance section) to produce a provider-independent JSON backup and restore it. The dump is portable across data-provider types (e.g. export from SQLite, import into PostgreSQL) and across SFTPGo versions on supported upgrade paths. See the REST API spec for the exact endpoints.

---

## 8. Quick reference — where to look next

| You need…                                                              | Fetch                                                                                                      |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| The full list of config parameters (every section, every field)        | [config-file.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/config-file.md)             |
| The env var convention authoritative doc + extra `SFTPGO_HOOK__*` vars | [env-vars.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/env-vars.md)                   |
| Install commands for your platform                                     | [installation.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md)           |
| Docker / K8s specifics                                                 | [docker.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/docker.md)                       |
| Data provider choice + tuning                                          | [data-provider.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/data-provider.md)         |
| First-time admin + first user walk-through                             | [initial-configuration.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/initial-configuration.md) |
| CLI flags and subcommands                                              | [cli.md](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/cli.md)                             |
