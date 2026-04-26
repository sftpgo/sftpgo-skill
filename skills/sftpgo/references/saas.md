# SFTPGo SaaS — what changes when the user is on the fully-managed service

SFTPGo is distributed as:

1. A self-hosted binary / Docker image / Kubernetes chart — the user owns the host.
2. A marketplace listing on AWS / Azure / GCP — the user owns the VM but the image is preconfigured. See the marketplace section of the public [installation guide](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md).
3. A **fully-managed SaaS** at `sftpgo.com` — we own the host. The customer only sees the WebAdmin, the WebClient, the SFTP endpoint, and the REST API. No shell, no config file, no environment variables, no Docker / systemd.

This file covers case 3 — the things an AI agent needs to know differently when the user is on the SaaS. Most of SFTPGo works the same as everywhere else; what changes is mostly **what to suggest** and **what to avoid**.

## 1. Detecting a SaaS user

Strong signals — treat as confirmed SaaS:

- Host is a dedicated per-instance subdomain we assigned at activation, or a customer-owned subdomain CNAME'd onto it.
- User configuration already uses `bucket=default`, `region=default` on S3 — these are the sentinel values for the integrated storage.
- User mentions "managed service", "SaaS", "fully managed", "sftpgo.com plan", "Small / Standard / Premium plan" (the SaaS tier names).
- User asks about a "custom domain" + CNAME pointing at the address we emailed them.

Weak signals — possible but verify:

- User mentions a custom domain that resolves to something they don't control (DNS CNAME).
- Questions framed around WebAdmin flows only, no talk of config files or env vars.
- Mentions of "recycle bin" already working out of the box.

**When in doubt, ask.** A single question — *"Are you using the fully-managed SaaS from sftpgo.com, or are you running SFTPGo on your own server?"* — avoids a long-winded wrong answer. Most of the skill content applies either way; the SaaS-specific adjustments are limited.

## 2. What is preconfigured on every SaaS plan

- **License** — activated on the subscription.
- **TLS** — Let's Encrypt certificate on the default domain (and on a customer's custom domain once the CNAME is in place).
- **Integrated S3 storage** — accessed by setting `bucket=default` and `region=default` on an S3 filesystem. These are sentinel values; credentials are managed server-side and rotated periodically without customer action.
- **`default` group** — a group named literally `default` is pre-created on every SaaS instance. It's wired to the integrated storage and uses `users/%username%/` as the key prefix. Assigning a user to the `default` group as **Primary** is all it takes to isolate that user in their own folder with no manual config.
- **Simplified user form for the default admin** — the default admin account that we ship has several advanced sections hidden on the Add User page. Additional admins the customer creates themselves see the full form.
- **Audit logs** — the audit plugin is active. Events are browsable from **Server Manager → Logs** and queryable via REST.
- **Brute-force protection** — Defender is on with sensible defaults.
- **Recycle bin** — deleted files move to a per-user `recycle/` folder; 3-day retention; files in the bin count against the user's quota.
- **Daily backups** — we manage them, so **the Backup event action is disabled** on SaaS.

## 3. SaaS conventions — things to remember when composing answers

### 3.1 Storage — always suggest the `default` group first

For 90 %+ of use cases on SaaS, the right answer to *"how do I add a user that can store files?"* is:

1. WebAdmin → Users → Add.
2. Fill username + password (or public key) + email.
3. Under **Groups**, add `default` as the Primary group.
4. Save.

The group already provides `s3config.bucket = "default"`, `s3config.region = "default"`, and `key_prefix = "users/%username%/"`. No need to write these fields on the user.

REST API equivalent (prefer this in automation answers):

```json
{
  "status": 1,
  "username": "alice",
  "email": "alice@example.com",
  "password": "choose-a-strong-one",
  "permissions": {"/": ["*"]},
  "groups": [{"name": "default", "type": 1}]
}
```

When the customer needs a different isolation pattern (for example, one shared folder per department), suggest creating an additional group based on the same S3 backend with a different key prefix — not editing the per-user filesystem directly.

### 3.2 Virtual folders

Customers often don't realize that key prefix ≠ virtual folder. Restricting a user to a sub-path of the integrated bucket is already handled by the group's key prefix — no virtual folder needed. Virtual folders are the right tool when the user needs to reach something **outside** their own prefix (a shared folder, a different bucket, another backend).

### 3.3 HIPAA, Geo-IP, LDAP, OIDC — tier-gated

- **Small plan and above**: Geo-IP filtering, HIPAA compliance mode.
- **Standard plan and above**: PGP encryption / decryption in the Event Manager, LDAP authentication, OpenID Connect.
- **Add-on (any plan)**: Document Editing & Collaboration (Collabora Online integration).

When the customer asks for one of these, confirm their plan can use it before walking them through the setup. If unclear, suggest emailing support.

### 3.4 Custom domain via CNAME

Default domain: a dedicated per-instance subdomain we assigned at activation (the address the customer received by email). The customer points a subdomain they own (e.g. `sftp.example.com`) via a DNS **CNAME** record targeting that assigned address. The IP behind it is static under normal operation, so an A record technically works; we recommend a CNAME because the IP can change in rare events such as a region migration or major infrastructure rework — we always notify the customer ahead of time when that happens. Apex / root domains don't support CNAME in standard DNS; the customer must use a subdomain, or a provider-specific CNAME-flattening / ALIAS record.

Once the CNAME resolves, the customer has **two ways** to get a Let's Encrypt cert for the custom domain:

1. **Email support** and we add the domain server-side via CLI; the cert is provisioned during that step.
2. **Self-service from the WebAdmin**: open **Server Manager → Configurations → Automatic Certificates (Let's Encrypt)**, add the custom domain to the list, save. This is one of the changes that need a short restart (§3.5) — we are notified automatically and apply the restart, which is when the cert is actually issued.

Either way, **keep the original assigned hostname in the cert list** — that name is how we monitor the instance, removing it would break our health checks.

### 3.5 Configuration changes that need a brief restart

- Turning on or changing the **OIDC** configuration.
- Turning on or changing the **LDAP** configuration.
- Enabling the global **Allow List** for the first time.
- Adding a custom domain in **Automatic Certificates (Let's Encrypt)** when the customer self-serves the cert (§3.4).

HIPAA mode is enabled by us on request rather than from the WebAdmin, so it does **not** appear on this list — see §3.3.

**We are notified automatically** when any of these settings are saved. No ticket required. During Italy business hours (09:00–20:00 CET / CEST) we verify the configuration is internally consistent and restart the service within a few minutes. Outside those hours the restart happens the next business morning; for urgent cases, email `support@sftpgo.com`.

Active SFTP / HTTPS / WebClient sessions are briefly disconnected during the restart, and any action running at that exact moment (an in-flight upload hook, a webhook, a scheduled rule that happened to fire) is interrupted; schedules themselves are not changed and will run again on their next trigger.

Tell the customer to save the config and wait — do **not** tell them to `systemctl restart sftpgo` or similar (they have no shell access).

## 4. What to avoid suggesting on SaaS

Any answer that assumes the customer owns the host is wrong. Specifically do **not** suggest:

- `systemctl restart sftpgo` / `Restart-Service sftpgo` / `docker restart` / `kubectl rollout`.
- Editing `sftpgo.json`, `/etc/sftpgo/env.d/*.env`, or any other config file.
- Setting environment variables (`SFTPGO_…`) — customers have no way to deliver them.
- `sftpgo resetpwd` / `sftpgo initprovider` / `sftpgo service reload` / any other CLI subcommand.
- `docker run …` / Helm values / `docker-compose.yml` recipes.
- Backup via the Event Manager **Backup** action — it's disabled on SaaS. If the customer asks how to back up their data, explain that we run automated daily backups, and if they want an **additional** copy they can set up an Event Manager rule that copies files (or a dump via the REST API) to their own storage.
- Changing `temp_path`, upload mode, or any other server-level setting that lives in the config file. These are set for the whole SaaS fleet.

## 5. What to suggest instead

| Customer need                                             | SaaS-appropriate answer                                                                                                                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Reset an admin password                                   | Sign in with a different admin and change the password from the user menu. If all admin credentials are lost, email `support@sftpgo.com`.                                |
| Add / remove users, groups, folders                       | WebAdmin or REST API — same shape as self-hosted.                                                                                                                        |
| Configure OIDC / LDAP                                     | WebAdmin → Server Manager → Configurations → OIDC / LDAP. Save and wait a few minutes for the automated restart.                                                         |
| Change brute-force thresholds                             | WebAdmin → Server Manager → Configurations → Defender.                                                                                                                   |
| Monitor connections / disconnect a user                   | WebAdmin → Active connections.                                                                                                                                           |
| See what happened on the server                           | WebAdmin → Server Manager → Logs (Filesystem / Provider / Other events). Export as CSV.                                                                                  |
| Back up data outside the SaaS                             | Use an Event Manager rule that copies files to another storage backend (S3 / GCS / Azure / SFTP) the customer owns. Or pull a JSON data-provider dump via the REST API.  |
| Send webhooks on upload                                   | Event Manager → Actions (HTTP) + Rules (Filesystem Events) — same as self-hosted.                                                                                        |
| Use a custom domain                                       | DNS CNAME to the dedicated address we emailed them at activation. TLS is automatic.                                                                                      |
| Turn on FTPS or WebDAV                                    | Email support; disabled by default.                                                                                                                                      |
| "I lost my SSE-C encryption key"                          | Files encrypted with that key are not recoverable. Recommend storing the key in a password manager / secret store before uploading data.                                 |

## 6. Marketplace offerings ≠ SaaS

Cloud marketplace listings (AWS / Azure / GCP Enterprise) are **not** the SaaS. The customer owns the VM or the managed K8s namespace, they have shell access, they can edit env files. Use `deployment.md` recipes for those. Clues: the customer mentions "AWS Marketplace", "AKS offering", "we run it in our own EKS cluster", "I can SSH to the VM". When in doubt, ask.
