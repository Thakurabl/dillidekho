# Renewing Your Jira API Token (1-Year Expiry)

A step-by-step guide for setting your Atlassian API token to expire in 1 year (the maximum allowed) instead of the default 30 days, and updating it everywhere it's used.

> **Background**: Atlassian API tokens have a configurable expiry. If you don't pick one explicitly, Atlassian uses a short default. Picking **1 year** (365 days) is the longest option available and means you only renew once per year instead of every month.

---

## Part A — Create the new token (5 minutes)

### Step 1. Open the Atlassian token manager

Go to: **https://id.atlassian.com/manage-profile/security/api-tokens**

Sign in with your Atlassian account if prompted (the same email you use for Jira, e.g. `you@maxwize.in`).

### Step 2. Revoke the old token (optional, do this *after* Part B works)

You'll see a list of existing tokens. Don't delete the old one yet — keep it active until the new one is verified working everywhere. Come back and click **Revoke** on the old token once Part B is complete.

### Step 3. Create a new token

1. Click **Create API token**.
2. **Name**: give it something descriptive like `Claude CLI – 2026` (so future-you knows when it was issued and what uses it).
3. **Expires on**: this is the key field. Click the dropdown and select **1 year** (or pick a date roughly 365 days in the future — Atlassian caps this at 1 year maximum).
4. Click **Create**.

### Step 4. Copy the token immediately

A dialog shows the token **once**. You will not be able to see it again.

- Click **Copy**.
- Paste it somewhere temporary (a password manager entry, or a sticky note you'll shred).
- **Do not** paste it into Slack, email, a Git-tracked file, or a chat with anyone.

### Step 5. Note the expiry date

Write down the new expiry date (roughly today + 1 year). Set a calendar reminder for **11 months from today** — that gives you a month of buffer to renew before things break.

---

## Part B — Update the token everywhere it's used

The token is stored in environment variables on each machine where Jira tooling runs. Update it in every location, then verify.

### Step 6. Update your shell rc file (Mac/Linux/WSL)

Open the rc file your shell uses:

- **zsh** (Mac default): `~/.zshrc`
- **bash** (Linux/Git Bash): `~/.bashrc`

Find the line that looks like:

```bash
export JIRA_API_TOKEN="<old token>"
```

Replace `<old token>` with the new one. Save the file.

Reload the shell:

```bash
source ~/.zshrc    # or ~/.bashrc
```

### Step 7. Update environment variables on Windows (if applicable)

If you're on native Windows (not WSL):

1. Press **Win**, type `Environment Variables`, open **Edit the system environment variables**.
2. Click **Environment Variables…**
3. Under **User variables**, find `JIRA_API_TOKEN`, click **Edit**, paste the new token, click **OK**.
4. Click **OK** on all dialogs.
5. **Close and reopen any terminal windows** — running terminals keep the old value.

### Step 8. Update any other places the token lives

Check these common spots and update if found:

- `.env` files in any project directory (`grep -r JIRA_API_TOKEN ~/projects` if you're not sure)
- Password manager entries
- CI/CD secrets (GitHub Actions, GitLab CI, etc.) — though for a personal token this is unusual
- Other developer machines you use

---

## Part C — Verify

### Step 9. Smoke test from a new terminal

Open a **new** terminal (so the new env var is loaded), then run:

```bash
jira list
```

You should see JSON output with your open Jira issues. If you get `HTTP 401`, the token wasn't updated correctly — re-check Step 6 or Step 7.

### Step 10. Verify Claude Code can use it

In a project directory, run `claude`, then ask:

> List my open Jira tickets.

If Claude returns a list, the new token is wired up end-to-end.

### Step 11. Revoke the old token

Go back to https://id.atlassian.com/manage-profile/security/api-tokens and click **Revoke** on the old token. This prevents leaks of the old value from being usable.

---

## Reminders for next year

- Calendar reminder for **11 months from today** to renew before expiry.
- Token expiry hits at midnight UTC on the expiry date — don't wait until the last day.
- If the token expires unexpectedly, every `jira` command returns `HTTP 401`. The fix is just this same process again.

---

## Troubleshooting

**`HTTP 401` after updating.** The shell didn't reload. Close the terminal and open a fresh one. Verify with `echo $JIRA_API_TOKEN | head -c 8` — the first 8 characters should match the new token.

**`Create API token` dialog doesn't show an expiry dropdown.** You may be on an older Atlassian account view. Try the direct link again, or click **Create API token (classic)** if offered. As of 2025, all Atlassian Cloud accounts have the expiry selector.

**Maximum expiry isn't 1 year.** Atlassian capped tokens at 1 year (365 days). If you see a shorter maximum, your org admin may have enforced a stricter policy — check with IT.

**Lost the token before saving it.** You cannot recover it. Revoke it and create a new one.

---

## Security notes

- The API token grants full access to Jira as your user. Treat it like a password.
- Never paste it into Slack, email, Git, or any file that could be committed.
- If you suspect the token leaked, revoke it immediately at the token manager URL above and create a fresh one.
