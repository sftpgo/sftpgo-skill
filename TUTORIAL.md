# Using the SFTPGo skill: a practical tutorial

The SFTPGo skill teaches your AI assistant how SFTPGo works. With it, you can ask in plain words ("create a user that can only upload", "email me when a file arrives") and get the exact steps for your server.

This tutorial shows how to ask so that you get good answers. It covers the tasks people ask about most: managing users and groups, automating them with scripts, and setting up automatic actions with the Event Manager. It ends with common problems and how to ask about each one.

You don't need to know how the skill works inside. You only need to know how to ask.

## 1. Before you start

Install the skill as described in the [README](README.md). In Claude Code, type these two commands:

```text
/plugin marketplace add sftpgo/sftpgo-skill
/plugin install sftpgo-skill
```

After that, the skill loads by itself whenever you ask about SFTPGo. To load it on purpose, start your message with `/sftpgo-skill:sftpgo`.

Using ChatGPT, Gemini, Cursor or another tool? Follow the matching section of the README. The questions in this tutorial work the same way there.

## 2. Words you'll see

| Word | What it means |
| --- | --- |
| WebAdmin | The web page where administrators manage SFTPGo. |
| WebClient | The web page where your users upload and download files. |
| User | An account that connects to SFTPGo to transfer files. |
| Group | A set of settings shared by many users, so you change them in one place. |
| Virtual folder | A storage location (a disk folder, a bucket…) that you add to a user's files under a path of your choice, like /shared. |
| Quota | The maximum space (and number of files) a user may use. |
| REST API | The way scripts and other programs control SFTPGo, without the WebAdmin. |
| API key | A long password for scripts that use the REST API. |
| Event Manager | The part of SFTPGo that does things automatically when something happens, such as sending an email after an upload. |

## 3. Five habits for good answers

**1. Say which SFTPGo you run.** The right answer depends on it. There are three kinds:

- **Fully-managed SaaS:** we host SFTPGo for you at sftpgo.com. You only use the web pages.
- **Self-hosted Enterprise:** you run SFTPGo on your own server, with an SFTPGo license.
- **Open Source edition:** you run the free version, for example from GitHub or Docker Hub. It has fewer features.

Start your question with one line such as:

> I'm on SFTPGo Enterprise, self-hosted, version 2.8.

If you forget, the skill asks you first, with numbered options. Reply with the number. If you're not sure which one you have, pick "Not sure" and it helps you find out.

**2. Say how you want to do it.** By default you get steps in the WebAdmin. If you'd rather use a script, say so: "write a script", "give me the REST API call".

**3. Paste the exact error.** "Login doesn't work" can have ten causes. The exact message, or the line from the SFTPGo log, usually points to one. When you report a problem, the skill asks for it first.

**4. Keep one task per conversation.** When you've done a step, reply in the same conversation ("done, now I see this error: …"). The skill then gives you only the next steps.

**5. Check before you change your live server.** You can ask "which docs page says this?" to see the source. Try anything new on a test user first. See [section 7](#7-what-to-double-check).

## 4. Managing users and groups

### 4.1 Everyday tasks in the WebAdmin

Ask in plain words. A few examples:

| You ask | You get |
| --- | --- |
| "Create a user that can upload but not download." | The steps to add the user, and which permissions to select in the **ACLs** section of the user page. |
| "Give every user a 5 GB quota and a 100 MB limit per uploaded file." | The steps to set it once on a group (see below), in the **Disk quota and bandwidth limits** section. |
| "Which settings does user alice really get from her groups?" | Where to find **Effective configuration**, in the Actions menu of the users list. It shows the user with all group settings applied. |
| "Add a shared folder that only the finance users can see." | A virtual folder, plus a group for the finance users that adds the folder to them. |

**Why the answers often use groups.** When something should apply to many users, it's easier to set it once on a group. Each user can belong to groups of three kinds:

| Group kind | What the user gets from it |
| --- | --- |
| Primary (one per user) | The base settings: storage, quota, speed limits, upload size limit. The group's value is used only when the user's own value is 0 or empty. So a user can still have a personal value. |
| Secondary (as many as you like) | Extra folders and permissions, added on top. |
| Membership (as many as you like) | Nothing. It only labels users, for example so that an Event Manager rule applies to them. |

Users get the group settings when they log in. Someone who is already connected keeps the old settings until they reconnect.

Example conversation:

> **You:** I'm on Enterprise self-hosted. I want every user to have a 5 GB quota, except two power users with 50 GB.
>
> **Skill:** Put the 5 GB on the primary group and leave the users at 0, so they take the group's value.
> 1. Go to **Groups** and open the group your users have as primary group. In **Disk quota and bandwidth limits**, set Quota size to 5GB. Save.
> 2. Check that each user's own Quota size is 0.
> 3. Open the two power users. In **Disk quota and bandwidth limits**, set Quota size to 50GB. A value on the user wins over the group.
>
> To check a user, open **Effective configuration** from the Actions menu in the users list.

### 4.2 Automating users and groups with scripts

If you create many users, or create them from another system (a CRM, an HR tool, a spreadsheet), a script does it for you. You don't need to write it: describe what you want and the skill writes it.

**Step 1: create an API key.** The script needs it to log in to SFTPGo. In the WebAdmin go to **Server Manager > API Keys** and create a key with the **Admin** scope. Copy it right away, because it is shown only once. Keep it secret, like a password.

**Step 2: describe the job.** Say what the input looks like, which groups to use, and what to do when something goes wrong:

> I'm on Enterprise self-hosted. Write a bash script that reads users.csv (columns: username, email, department) and creates each user with the primary group "partners" and the secondary group named like the department. Use an API key from the SFTPGO_API_KEY environment variable. Skip users that already exist and print a summary at the end.

You get a complete script. You don't need to understand every line, but this is the part that creates one user:

```bash
curl -s -X POST "$SFTPGO_URL/api/v2/users" \
  -H "X-SFTPGO-API-KEY: $SFTPGO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "email": "alice@example.com",
    "status": 1,
    "groups": [
      {"name": "partners", "type": 1},
      {"name": "finance", "type": 2}
    ],
    "permissions": {"/": ["*"]}
  }'
```

In the groups list, `type` 1 means primary group and 2 means secondary group. The groups must already exist: create them first, or ask the script to create them.

**Other jobs a script can do:**

- "Create the groups partners, finance and legal, each with its own shared folder."
- "List all users with their used space and their quota, as a CSV file."
- "Move every user of the group old-partners to the group partners."
- "Disable the users in this list." (then paste the list)
- "Change the password of user backup-bot and print the new one."

**Good to know before you run a script:**

- Run it on one or two test users first, then on everyone.
- A script that creates users fails for users that already exist. Ask it to skip them, as in the example above.
- A script that changes users should change only what you asked. Say it clearly: "change only the quota, keep everything else".

## 5. Automatic actions with the Event Manager

The Event Manager does things for you when something happens. You set it up in two parts:

- **Actions:** what to do. For example, send an email, call a web address, delete old files.
- **Rules:** when to do it. A rule has a trigger (a file is uploaded, every night at 2:00, a user logs in…), optional filters (only some users, only some folders) and the actions to run.

You find them under **Event Manager > Actions** and **Event Manager > Rules** in the WebAdmin.

Most of what follows needs the Enterprise edition. On the Open Source edition, the skill tells you what isn't available and suggests the closest alternative.

### 5.1 Things people often automate

| You ask | What the skill sets up |
| --- | --- |
| "Email me when someone uploads a file." | An email action, and a rule that runs it after each upload. |
| "Post a Slack (or Teams) message when a partner uploads to /incoming." | An action that sends the chat message, and a rule for uploads in that folder only. |
| "Every night, delete files older than 30 days in /tmp." | A rule that runs every night and deletes the old files. |
| "Send me a daily summary of uploads per user." | A rule that runs every day and emails a report. |
| "Disable users that haven't logged in for 90 days." | A rule that runs every day and disables inactive users. |
| "When someone logs in for the first time with our company login (single sign-on), create their account and put them in the right groups." | A rule that creates the account at the first login, with groups based on the person's role. |

### 5.2 Example: email on upload

> **You:** I'm on Enterprise self-hosted. How do I get an email when a user uploads a file?
>
> **Skill:** You need a mail server set up first, then an action that sends the email and a rule that runs it on upload.
> 1. Go to **Server Manager > Configurations > SMTP** and enter your mail server details. Save.
> 2. Go to **Event Manager > Actions** and click **Add**. Pick the Email type, enter the recipient, and use this as the subject: `{{.Name}} uploaded {{.VirtualPath}}`
> 3. Go to **Event Manager > Rules** and click **Add**. Pick **Filesystem events**, tick **upload**, and add the action from step 2.
>
> That's it. The next upload sends the email. Want a longer email body with the file size and the user's IP?

The parts in double braces are placeholders. When the email is sent, SFTPGo replaces `{{.Name}}` with the username and `{{.VirtualPath}}` with the path of the uploaded file. So the subject becomes, for example, "alice uploaded /reports/march.pdf".

### 5.3 Tips

- **Say what should start it and what the message must contain.** "When a file over 1 GB is uploaded to /reports, post the file name and size to Teams" gets you a finished setup. "Notify me" gets you a question back.
- **Ask for a preview.** "What will the email look like when alice uploads report.pdf (3 MB)?" shows you the result before you save anything.
- **Some placeholders are empty for some triggers.** A rule that runs every night has no uploaded file, so the file path placeholder is empty there. If a message arrives with empty parts, paste what you wrote and ask why.

## 6. Common problems and how to ask

Add your setup and version to each question.

| Problem | What to ask | Good to know |
| --- | --- | --- |
| Users get "permission denied" when they log in via SFTP. | "SFTP login fails with permission denied. Here is the line from the SFTPGo log: …" | The SFTPGo log says much more than the message in the user's program. Paste the log line. |
| Uploads from the WebClient fail. | "WebClient uploads fail with this error: … We run SFTPGo behind nginx." | Say if another web server (like nginx) sits in front of SFTPGo. Its own size limit often blocks big uploads. |
| You want a size limit for every uploaded file. | "Set a 100 MB per-file upload limit for all users, with a few exceptions." | Users who already have their own limit keep it. |
| The used space shown for a user looks wrong. | "The used space for user bob doesn't match the files on disk. Why, and how do I fix it?" | If files were changed outside SFTPGo, SFTPGo has to recount them (a quota scan). |
| A group change doesn't reach connected users. | "I changed a group setting. How do I apply it to users who are connected now?" | Connected users keep the old settings until they reconnect. |
| You want two servers sharing the work. | "We want two SFTPGo servers behind a load balancer, with files on local disks." | Both servers need the same database, the same files and the same SSH keys. Some features need a specific license. |
| You want a second server that takes over if the first one fails. | "How do I set up an active/standby pair?" | The spare server should stay off until it's needed, because a running server counts as an active license. |
| You want to be told about uploads. | See [section 5.2](#52-example-email-on-upload). | A mail server must be set up first. |

## 7. What to double-check

The skill knows SFTPGo well, but it can't see your server. Check these points before you act on an answer:

- **Details the documentation doesn't cover.** If the answer matters, try it on a test user, or ask "which docs page says this?".
- **Where things are on screen.** Menus and section names can change between versions. If you can't find something, tell the skill your version and what you see.
- **Your version and edition.** An answer for the latest Enterprise release can be different on an older version or on the Open Source edition.
- **Anything that deletes or disconnects.** Cleanup rules, disconnecting users and scripts that change many users act on real data. Try them on a test user or a test folder first.

The skill reads the current documentation at [docs.sftpgo.com/enterprise](https://docs.sftpgo.com/enterprise/) when it answers. If the skill and the documentation ever disagree, trust the documentation.

## 8. Cheat sheet

```text
I'm on SFTPGo Enterprise self-hosted, version 2.8.        <- always start like this

How do I … in the WebAdmin?                               <- steps in the web page
Write a script that …                                     <- automate many changes
Give me the REST API call to …                            <- one call for your own program
Email me / notify me when …                               <- automatic actions
… fails with this error: <paste the log line>             <- fix a problem
Which docs page says this?                                <- check the source
```
