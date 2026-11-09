# SFTPGo SaaS — what changes when the user is on the fully-managed service

SFTPGo is distributed as:

1. A self-hosted binary / Docker image / Kubernetes chart — the user owns the host.
2. A marketplace listing on AWS / Azure / GCP — the user owns the VM but the image is preconfigured. See the marketplace section of the public [installation guide](https://raw.githubusercontent.com/sftpgo/docs/enterprise/docs/installation.md).
3. A **fully-managed SaaS** at `sftpgo.com` — we own the host. The customer only sees the WebAdmin, the WebClient, the SFTP endpoint, and the REST API. No shell, no config file, no environment variables, no Docker / systemd.

This file covers case 3 — the things an AI agent needs to know differently when the user is on the SaaS. Most of SFTPGo works the same as everywhere else; what changes is mostly **what to suggest** and **what to avoid**.

## 1. Detecting a SaaS user

Strong signals — treat as confirmed SaaS:

- Host is a dedicated per-instance subdomain we assigned at activation, of the form `<instance-id>.softwareng.cloud`, or a customer-owned subdomain CNAME'd onto it.
- User configuration already uses `bucket=default`, `region=default` on S3 — these are the sentinel values for the integrated storage.
- User mentions "managed service", "SaaS", "fully managed", "sftpgo.com plan", or a plan name: **Tiny, Small, Standard, Professional, Premium** (in increasing order).
- User mentions the `default` or `superusers` group, the `copy_to_recycle` / `recycle_retention` rules, or the credentials file received at activation.
- User asks about a "custom domain" + CNAME pointing at the address we emailed them.

Weak signals — possible but verify:

- User mentions a custom domain that resolves to something they don't control (DNS CNAME).
- Questions framed around WebAdmin flows only, no talk of config files or env vars.
- Mentions of "recycle bin" already working out of the box.

**When in doubt, ask.** The single numbered question in `SKILL.md` ("Step 0 — establish the environment") separates the SaaS from self-hosted Enterprise and Open Source in one reply and avoids a long-winded wrong answer. Most of the skill content applies either way; the SaaS-specific adjustments are limited.

## 2. What is preconfigured on every SaaS plan

- **Addresses** — WebAdmin at `https://<instance-id>.softwareng.cloud/web/admin`, WebClient at `https://<instance-id>.softwareng.cloud/web/client`, SFTP on port 22 of the same host. The first admin's username and password come in a credentials file sent at activation.
- **License** — activated on the subscription.
- **TLS** — Let's Encrypt certificate on the default domain (and on a customer's custom domain once the CNAME is in place), renewed automatically.
- **Protocols** — SFTP and HTTPS (WebAdmin, WebClient, REST API) are enabled. FTP/FTPS and WebDAV are disabled by default and enabled on request (§3.16).
- **Integrated S3 storage** — accessed by setting `bucket=default` and `region=default` on an S3 (Compatible) filesystem. These are sentinel values; credentials are managed server-side and rotated periodically without customer action.
- **`default` group** — pre-created and wired to the integrated storage with `users/%username%/` as the key prefix. As Primary group it gives each user a private folder: Alice sees `users/alice/` only.
- **`superusers` group** — pre-created on the same storage with an empty key prefix: its members see every user's files (each under `users/<username>`) and anything else in the storage. See §3.2.
- **Simplified user form for the default admin** — the default admin account that we ship has several advanced sections hidden on the Add User page. Additional admins the customer creates themselves see the full form.
- **Audit logs** — the audit plugins are active. Events are browsable from **Server Manager → Logs** and queryable via REST. Retention depends on the plan (§3.12).
- **Brute-force protection** — the Defender is always on, plus a connection rate limit (§3.8).
- **Recycle bin** — deleted or overwritten files are copied to a `recycle/` folder for 3 days, through two Event Manager rules (§3.6).
- **Daily backups** — we manage them, so **the Backup event action is disabled** on SaaS.

## 3. SaaS conventions — things to remember when composing answers

### 3.1 Adding a user — the `default` group does the work

For 90 %+ of use cases on SaaS, the right answer to *"how do I add a user that can store files?"* is:

1. WebAdmin → **Users → Add**.
2. Fill in Username and Password (or a public key).
3. Leave the rest as it is. **Groups** already shows `default` as the Primary group, and **File system** shows **None**: that is correct, the storage comes from the group.
4. Optionally open **Profile** and enter the user's Email. It is needed for password reset (which also needs SMTP, §3.13).
5. Save.

What to give the new user: the username and password, plus either the WebClient address (simplest) or the SFTP settings for FileZilla / WinSCP / Cyberduck (protocol SFTP, host = the instance address or custom domain, port 22). The first SFTP connection asks to confirm the server's host key: that is normal, accept it.

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

### 3.2 The `superusers` group — access to all files

For an administrator who needs to see or recover every user's files: add the user as in §3.1 and pick `superusers` instead of `default` as Primary group. Recommend it only for the account owner or a trusted colleague. Files deleted or overwritten by a member of `superusers` go to the `recycle/` folder at the top of the storage, not to the recycle folder of the user who owns them.

### 3.3 New groups and folders on the integrated storage

The storage settings appear only when creating a new group or virtual folder. To stay on the integrated storage, choose **S3 (Compatible)** and enter `default` as both Bucket and Region; the real values are substituted server-side.

Each user sees only the files under the key prefix. For a different isolation pattern (for example one folder per department), create a new group on the same storage with another prefix, such as `departments/finance/%username%/`, instead of editing the per-user filesystem. An empty prefix gives access to the entire storage, as for `superusers`.

Users can also use their own storage instead (Amazon S3, Backblaze B2, Wasabi, Cloudflare R2 and other S3-compatible services, Google Cloud Storage, Azure Blob, another SFTP server): enter its real settings instead of `default` / `default`.

### 3.4 Virtual folders

A private folder per user needs nothing extra, and users can create sub-folders themselves. Key prefix ≠ virtual folder: restricting a user to a sub-path of the integrated bucket is already handled by the group's key prefix. A virtual folder (shown as "shared folder" in SaaS user-facing material) is the right tool when several users must work on the same files, or when a user needs to reach something **outside** their own prefix (a different bucket, another backend): create the folder once, then add it to the users or groups that need it, for example at `/shared`.

### 3.5 Encrypting files with a key (SSE-C)

Optional. The key is set in the storage settings of the `default` group, and the same key must be set on `superusers`, otherwise its members cannot open the users' files. Three rules to state plainly:

- Decide at the start. The key applies to files uploaded from then on; files uploaded earlier without a key become unreadable once the group has a key.
- Never change the key. Files uploaded with the old key can only be opened with the old key.
- Keep the key safe. If it is lost, nobody can recover the files, not even us.

If the user is unsure, recommend leaving it off.

### 3.6 Recycle bin

- When a file is deleted or overwritten, the previous version is copied to the user's `recycle/` folder with a unique code added to its name. It stays there 3 days (72 hours), then is removed for good.
- To restore a file, the user moves it out of `recycle/` from the WebClient or an SFTP program.
- Files in `recycle/` count towards the storage included in the plan, like any other file.
- It is made of two Event Manager rules: `copy_to_recycle` saves the copies, `recycle_retention` removes old ones once a day at 01:00 UTC (the time can be changed in that rule's schedule).
- To change the retention: **Event Manager → Actions → `recycle_retention`**, change the number next to `/recycle` (in hours: 72 = 3 days, 168 = 7 days, 720 = 30 days) and save.
- To turn the recycle bin off: **Event Manager → Rules → `copy_to_recycle`**, set Status to Inactive and save.
- Recommend not changing anything else in these two rules; for other needs, write to technical support (§3.16).

### 3.7 Passwords and sign-in

- **First login** — the admin should change the password (user icon top right → Change password) and enable two-factor authentication from the same menu **before** adding users: the instance is reachable from the internet from day one.
- **Password strength** — the default strength is 70: it rejects obvious passwords and accepts a few unrelated words. To relax or tighten it for one user: open the user → **Profile → Password strength**; a lower value accepts simpler passwords, 0 accepts any. Classic rules (minimum length, uppercase, digits, symbols) can be set there instead.
- **Two-factor authentication** — admins enable it from their profile, users from the WebClient.
- **Public keys** — the user creates a key pair (`ssh-keygen` or PuTTYgen) and sends the public key; the admin pastes it in the user's **Public keys** section. Supported: Ed25519, ECDSA, RSA 2048 bits or more. A key starting with `---- BEGIN SSH2 PUBLIC KEY ----` must first be converted with `ssh-keygen -i -f key.pub`.
- **LDAP / Active Directory and OpenID Connect** (Standard plan and above) — **Server Manager → Configurations → LDAP** or **→ OIDC**. Saving needs a short restart (§3.15). OpenID Connect works for the WebAdmin and the WebClient only: users who also need SFTP must sign in to the WebClient once and set a password or a public key.

### 3.8 Security: blocked addresses and IP lists

- **Automatic blocking** — always on. An address that fails to log in too many times, or keeps opening connections without logging in, is blocked: 30 minutes the first time, longer if it repeats. Blocked addresses are listed in **IP Manager → Auto Block List**; deleting an entry unblocks it right away. This is the usual cause of "my user can't log in any more" / "connection refused".
- **Rate limit** — each address can open at most 16 new SFTP or FTP connections per second. Only automated systems hit it; exempt them with the Rate limiters safe list.
- **IP Manager → IP Lists**:
  - **Trusted List** — never blocked or limited (e.g. the office). The one used most often.
  - **Defender List** — addresses always blocked, or never blocked, by the attack protection.
  - **Rate limiters safe list** — exempt from the connection limit.
  - **Allow List** — when enabled, only listed addresses can connect. Enabled on request; it must be filled in first, otherwise nobody can connect. Enabling it the first time needs a short restart (§3.15).
- Per-user or per-group address restrictions are in the **ACLs** section of the user or group.
- **Geo-IP** (Small plan and above) — **Server Manager → Configurations → Geo-IP**, allow or deny by two-letter country code. Trusted List addresses are never blocked.
- If the default Defender or rate-limit thresholds don't fit, write to technical support (§3.16).

### 3.9 Plan-gated features

- **Small plan and above**: Geo-IP filtering, HIPAA compliance mode.
- **Standard plan and above**: LDAP authentication, OpenID Connect, and in the Event Manager the **Event report** and **IDP account check** actions and PGP encryption / decryption.
- **Add-on (any plan)**: Document Editing & Collaboration (Collabora Online integration). If fewer editor seats were bought than there are users, disable the editor for some users or groups in their **ACLs** section.

When the user asks for one of these, confirm their plan can use it before walking them through the setup.

### 3.10 HIPAA mode (Small plan and above)

Only for organizations handling US healthcare data. Not enabled by default: the user writes to `support@sftpgo.com`, we sign a BAA and switch the instance to HIPAA mode. In HIPAA mode:

- the instance and the data are in the United States;
- web sessions expire after 10 minutes of inactivity instead of 20;
- on the integrated storage the encryption key (§3.5) is mandatory: set it on `default` and `superusers` before adding users; SFTPGo refuses to save users, groups or folders on the integrated storage without it;
- the recycle bin keeps files for 30 days;
- audit logs are kept at least 6 years, encrypted and copied to a second US location, with a separate read-only installation to download them as CSV.

Also recommend allowing connections from the United States only (Geo-IP). With their own storage, the user must make sure it is HIPAA compliant too, including a BAA with the storage provider.

### 3.11 Custom domain via CNAME

Default domain: the dedicated `<instance-id>.softwareng.cloud` address we assigned at activation. The user points a subdomain they own (e.g. `sftp.example.com`) at it with a DNS **CNAME** record (Name = the part before the domain, e.g. `sftp`; Value = the assigned address). Always recommend a CNAME, not an A record: the IP addresses behind the assigned name can change. Apex / root domains don't support CNAME at most DNS providers; use a subdomain. The record can take from a few minutes to a few hours to become active.

Once the CNAME resolves, there are **two ways** to get a Let's Encrypt certificate for the custom domain:

1. **Ask technical support** (§3.16) and we add the domain server-side.
2. **Self-service from the WebAdmin**: **Server Manager → Configurations → Automatic Certificates (Let's Encrypt)**, add the domain in **Domain** after the address already there, separated by a space, leave the other fields as they are and save. This needs a short restart (§3.15), which is when the certificate is issued.

Either way, **keep the original assigned hostname in the list** — we use it to monitor the instance; removing it breaks our health checks.

Branding (name, logo and icon of the WebAdmin and WebClient, a link on the login page such as a privacy policy) is in **Server Manager → Configurations → Branding**.

### 3.12 Audit logs and active connections

- **Server Manager → Logs**:
  - **Filesystem events** — uploads, downloads, renames, deletions; filter and export as CSV.
  - **Provider events** — admin changes to users, groups, folders, shares, admins.
  - **Other events** — logins of users and admins, successful and failed.
- Retention by plan: Tiny 7 days, Small 14, Standard 30, Professional 60, Premium 90 (6 years in HIPAA mode).
- **Active connections** shows who is connected now; the X button closes a connection.

### 3.13 Email (SMTP)

SFTPGo sends emails only after the user enters the settings of their mail provider in **Server Manager → Configurations → SMTP**; the **Test** button checks that emails arrive. Needed for password reset, for shares protected by email, and for any Event Manager action that sends email. "My user did not receive the password reset email" almost always means SMTP is not configured.

### 3.14 Event Manager and REST API on SaaS

- The Event Manager works as on self-hosted Enterprise, with the limits of §3.9 and the Backup action disabled. Leave the two recycle rules (§3.6) in place.
- For webhooks, suggest testing the endpoint with a service such as webhook.site first.
- Everything in the WebAdmin is also available through the REST API; a Terraform provider exists too. For scripts, API keys are more convenient than tokens: they don't expire and are sent in the `X-SFTPGO-API-KEY` header.
- End users can automate their own transfers: a user token from `/api/v2/user/token`, then the `/api/v2/user/files` endpoints.

### 3.15 Configuration changes that need a brief restart

- Turning on or changing the **OIDC** configuration.
- Turning on or changing the **LDAP** configuration.
- Enabling the global **Allow List** for the first time.
- Adding a custom domain in **Automatic Certificates (Let's Encrypt)** when the user self-serves the certificate (§3.11).

HIPAA mode is enabled by us on request rather than from the WebAdmin, so it does **not** appear on this list — see §3.10.

**We are notified automatically** when any of these settings are saved; the page saying the service must be restarted needs no action from the user. During Italian business hours (09:00–20:00 Italian time) we check the configuration and restart the service within a few minutes; otherwise the next business morning. If it is needed sooner, write to technical support (§3.16).

Active SFTP / HTTPS / WebClient sessions are briefly disconnected during the restart, and any action running at that exact moment (an in-flight upload hook, a webhook, a scheduled rule that happened to fire) is interrupted; users simply reconnect and retry. Schedules are not changed and run again on their next trigger.

Tell the user to save the config and wait — do **not** tell them to `systemctl restart sftpgo` or similar (they have no shell access).

### 3.16 Where to get help

- **Technical support** (something does not work, a setting to change, a feature to enable such as FTP/FTPS or WebDAV): `ticket@softwareng.eu`. Each email opens a request that gets a reply.
- **Commercial and general questions** (plans, billing, quotes, HIPAA): `support@sftpgo.com`.
- For technical requests, suggest including: the instance address, the username involved, what they were doing and when, and a screenshot of the error.

## 4. Common SaaS questions — short answers

| Question | Answer |
| --- | --- |
| A user can't log in any more / gets "connection refused". | The address is probably blocked after failed logins (§3.8). Unblock it in **IP Manager → Auto Block List**, then check the password (case matters). |
| A password is rejected as too weak. | Use a longer password (three or four unrelated words work well), or relax **Profile → Password strength** for that user (§3.7). |
| A file was deleted or overwritten by mistake. | A copy is in the user's `recycle/` folder for 3 days; move it back (§3.6). |
| A user did not receive the password reset email. | Configure SMTP (§3.13). |
| A user sees an empty folder, or not a colleague's files. | By design, each user has a private folder. To share between users, use a shared (virtual) folder (§3.4); to share with outsiders, share links from the WebClient. |
| A saved setting says the service must be restarted. | Nothing to do: we are notified and restart it (§3.15). |

## 5. What to avoid suggesting on SaaS

Any answer that assumes the customer owns the host is wrong. Specifically do **not** suggest:

- `systemctl restart sftpgo` / `Restart-Service sftpgo` / `docker restart` / `kubectl rollout`.
- Editing `sftpgo.json`, `/etc/sftpgo/env.d/*.env`, or any other config file.
- Setting environment variables (`SFTPGO_...`) — there is no way to deliver them.
- `sftpgo resetpwd` / `sftpgo initprovider` / `sftpgo service reload` / any other CLI subcommand.
- `docker run ...` / Helm values / `docker-compose.yml` recipes.
- Backup via the Event Manager **Backup** action — it's disabled on SaaS. If the user asks how to back up their data, explain that we run automated daily backups, and if they want an **additional** copy they can set up an Event Manager rule that copies files (or a dump via the REST API) to their own storage.
- Changing the upload mode or any other server-level setting that lives in the config file. These are set for the whole SaaS fleet.
- Removing or rewriting the `copy_to_recycle` / `recycle_retention` rules beyond the retention hours and the on/off switch (§3.6).

## 6. What to suggest instead

| Need                                                      | SaaS-appropriate answer                                                                                                                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Reset an admin password                                   | Sign in with a different admin and change the password from the user menu. If all admin credentials are lost, write to technical support (§3.16).                       |
| Add / remove users, groups, folders                       | WebAdmin or REST API — same shape as self-hosted; new users keep `default` as Primary group (§3.1).                                                                      |
| Configure OIDC / LDAP                                     | WebAdmin → Server Manager → Configurations → OIDC / LDAP (Standard plan and above). Save and wait for the automated restart.                                             |
| Unblock an address / change brute-force behaviour         | Unblock in IP Manager → Auto Block List; never block an address with the Trusted List. For different thresholds, write to technical support.                            |
| Monitor connections / disconnect a user                   | WebAdmin → Active connections, X button.                                                                                                                                 |
| See what happened on the server                           | WebAdmin → Server Manager → Logs (Filesystem / Provider / Other events). Export as CSV. Retention depends on the plan (§3.12).                                           |
| Back up data outside the SaaS                             | Use an Event Manager rule that copies files to another storage backend (S3 / GCS / Azure / SFTP) the user owns. Or pull a JSON data-provider dump via the REST API.      |
| Send webhooks on upload                                   | Event Manager → Actions (HTTP) + Rules (Filesystem Events) — same as self-hosted. Test with webhook.site first.                                                          |
| Use a custom domain                                       | DNS CNAME to the dedicated address we emailed them at activation, then the certificate (§3.11).                                                                          |
| Turn on FTP/FTPS or WebDAV                                | Write to technical support; disabled by default.                                                                                                                          |
| "I lost my SSE-C encryption key"                          | Files encrypted with that key are not recoverable. Recommend storing the key in a password manager / secret store before uploading data (§3.5).                          |

## 7. Marketplace offerings ≠ SaaS

Cloud marketplace listings (AWS / Azure / GCP Enterprise) are **not** the SaaS. The customer owns the VM or the managed K8s namespace, they have shell access, they can edit env files. Use `deployment.md` recipes for those. Clues: the customer mentions "AWS Marketplace", "AKS offering", "we run it in our own EKS cluster", "I can SSH to the VM". When in doubt, ask.
