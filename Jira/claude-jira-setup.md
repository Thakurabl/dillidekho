# Claude + Jira (Direct REST API) — Setup Guide

A zero-MCP integration that lets Claude Code manage Jira tickets by shelling out to a small local CLI that calls Jira Cloud's REST API directly. Faster than MCP, fewer moving parts, and the only long-lived state is a shell env var.

---

## Table of Contents

1. [What this gives you](#what-this-gives-you)
2. [Prerequisites](#prerequisites)
3. [Setup instructions (for humans)](#setup-instructions-for-humans)
4. [Setup instructions (for Claude — agent mode)](#setup-instructions-for-claude--agent-mode)
5. [Source code: the `jira` CLI](#source-code-the-jira-cli)
6. [Source code: the project `CLAUDE.md`](#source-code-the-project-claudemd)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)
9. [Security notes](#security-notes)

---

## What this gives you

After setup, from any VS Code terminal running `claude`, you can say things like:

- "List my open Jira tickets grouped by status."
- "Create a Bug in PROJ titled 'Checkout crashes on empty cart', assign it to me."
- "Move PROJ-42 to In Review and comment with a link to the commit we just made."

Claude invokes a local `jira` command (plain Python, no dependencies) that hits the Jira Cloud REST API v3. No MCP server, no proxy, no OAuth dance — just HTTP Basic auth with an API token.

---

## Prerequisites

- **OS**: macOS, Linux, or Windows with WSL. (Native Windows works too; see Troubleshooting.)
- **Python**: 3.8+ (`python3 --version`).
- **VS Code** with a terminal.
- **Claude Code** installed: `npm install -g @anthropic-ai/claude-code`, then `claude` once to log in.
- **Jira Cloud** account on a site like `https://yourcompany.atlassian.net`.
- **Jira API token**: create one at https://id.atlassian.com/manage-profile/security/api-tokens.

---

## Setup instructions (for humans)

Follow these if you're doing it yourself.

### 1. Get your Jira API token

Visit https://id.atlassian.com/manage-profile/security/api-tokens → **Create API token** → name it "Claude CLI" → **copy the value** (you won't see it again).

### 2. Set environment variables

Append these to `~/.zshrc` (macOS default) or `~/.bashrc` (Linux / Git Bash):

```bash
export JIRA_BASE_URL="https://maxwize-in.atlassian.net"
export JIRA_EMAIL="abhishek@maxwize.in"
export JIRA_API_TOKEN="paste-the-token-here"
```

Reload: `source ~/.zshrc` (or open a new terminal).

### 3. Install the `jira` CLI

```bash
mkdir -p ~/.local/bin
# Copy the script from the "Source code" section below into:
#   ~/.local/bin/jira
chmod +x ~/.local/bin/jira
```

Make sure `~/.local/bin` is on your PATH. If `echo $PATH` doesn't include it, add to your rc file:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 4. Smoke test

```bash
jira list
```

You should see JSON with your open issues. If you get an HTTP error, jump to [Troubleshooting](#troubleshooting).

### 5. Drop `CLAUDE.md` into your project

Copy the contents of [Source code: the project CLAUDE.md](#source-code-the-project-claudemd) into a file named `CLAUDE.md` at your project root (or at `~/.claude/CLAUDE.md` for machine-wide availability). Claude Code auto-reads this file on startup.

### 6. Use it

In VS Code, open a terminal, `cd` into the project, run `claude`, and ask it to do Jira things in natural language.

---

## Setup instructions (for Claude — agent mode)

> **For the human reading this**: when you hand this file to Claude Code (e.g. `claude "please set up the Jira integration described in claude-jira-setup.md"`), Claude follows the instructions in this section to do steps 2, 3, and 5 above automatically. You still have to do steps 1 (create the token) and 4 (smoke test) yourself.

**Claude, when asked to perform this setup, do the following in order. Ask the user for confirmation before writing any file. Never echo the API token back to the terminal.**

1. **Check prerequisites.** Run `python3 --version` and `claude --version`. If either is missing, stop and tell the user.

2. **Ask the user for their Jira details.** You need three values: `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`. Tell the user to create the token at https://id.atlassian.com/manage-profile/security/api-tokens if they don't have one. Collect the values, but **do not write the token into any file**. Only write it into the user's shell rc file.

3. **Detect the shell rc file.** Check `$SHELL`. If zsh → `~/.zshrc`. If bash → `~/.bashrc`. Otherwise ask the user which one to use.

4. **Append env vars to the rc file**, only if they aren't already present. Use `grep -q 'JIRA_BASE_URL'` to check before appending. Append a block like:

   ```bash
   # Added by claude-jira-setup
   export JIRA_BASE_URL="<value>"
   export JIRA_EMAIL="<value>"
   export JIRA_API_TOKEN="<value>"
   export PATH="$HOME/.local/bin:$PATH"
   ```

5. **Create `~/.local/bin/jira`** with the exact contents from the [Source code: the `jira` CLI](#source-code-the-jira-cli) section of this document. Make it executable: `chmod +x ~/.local/bin/jira`.

6. **Create `CLAUDE.md`** in the current working directory with the contents from [Source code: the project `CLAUDE.md`](#source-code-the-project-claudemd). If one already exists, ask before overwriting — offer to append instead.

7. **Test.** Tell the user to open a new terminal (so the new env vars load) and run `jira list`. Do not run this yourself with the token you were just handed — the user should verify in their own shell.

8. **Report.** Summarize what you did, which files you touched, and the one manual step (opening a new terminal). Do not print the token.

---

## Source code: the `jira` CLI

Save as `~/.local/bin/jira` and `chmod +x` it. Pure Python stdlib — no `pip install` required.

```python
#!/usr/bin/env python3
"""
jira — thin Jira Cloud REST wrapper for Claude Code.

Commands:
  jira list [--jql "..."] [--max 20]
  jira get PROJ-123
  jira create --project PROJ --summary "..." [--type Task] [--description "..."]
  jira comment PROJ-123 "comment body"
  jira transition PROJ-123 "Done"
  jira assign PROJ-123 <accountId|email>
  jira search --jql "..." [--max 50]
"""
import argparse, base64, json, os, sys
from urllib import request, parse, error

try:
    BASE = os.environ["JIRA_BASE_URL"].rstrip("/")
    EMAIL = os.environ["JIRA_EMAIL"]
    TOKEN = os.environ["JIRA_API_TOKEN"]
except KeyError as e:
    sys.stderr.write(f"Missing env var {e}. Set JIRA_BASE_URL, JIRA_EMAIL, JIRA_API_TOKEN.\n")
    sys.exit(2)

AUTH = base64.b64encode(f"{EMAIL}:{TOKEN}".encode()).decode()
HEADERS = {
    "Authorization": f"Basic {AUTH}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}


def call(method, path, body=None, query=None):
    url = f"{BASE}{path}"
    if query:
        url += "?" + parse.urlencode({k: v for k, v in query.items() if v is not None})
    data = json.dumps(body).encode() if body is not None else None
    req = request.Request(url, data=data, method=method, headers=HEADERS)
    try:
        with request.urlopen(req) as r:
            raw = r.read()
            return json.loads(raw) if raw else {}
    except error.HTTPError as e:
        sys.stderr.write(f"HTTP {e.code} on {method} {path}: {e.read().decode()}\n")
        sys.exit(1)


def adf(text):
    """Jira Cloud uses Atlassian Document Format for descriptions/comments."""
    return {
        "type": "doc",
        "version": 1,
        "content": [
            {"type": "paragraph", "content": [{"type": "text", "text": text}]}
        ],
    }


def cmd_list(a):
    jql = a.jql or "assignee = currentUser() AND statusCategory != Done ORDER BY updated DESC"
    data = call(
        "GET",
        "/rest/api/3/search/jql",
        query={"jql": jql, "fields": "summary,status,assignee", "maxResults": a.max},
    )
    out = [
        {
            "key": i["key"],
            "summary": i["fields"]["summary"],
            "status": i["fields"]["status"]["name"],
            "assignee": (i["fields"].get("assignee") or {}).get("displayName"),
            "url": f"{BASE}/browse/{i['key']}",
        }
        for i in data.get("issues", [])
    ]
    print(json.dumps(out, indent=2))


def cmd_get(a):
    d = call("GET", f"/rest/api/3/issue/{a.key}")
    print(
        json.dumps(
            {
                "key": d["key"],
                "summary": d["fields"]["summary"],
                "status": d["fields"]["status"]["name"],
                "type": d["fields"]["issuetype"]["name"],
                "assignee": (d["fields"].get("assignee") or {}).get("displayName"),
                "reporter": (d["fields"].get("reporter") or {}).get("displayName"),
                "priority": (d["fields"].get("priority") or {}).get("name"),
                "url": f"{BASE}/browse/{d['key']}",
            },
            indent=2,
        )
    )


def cmd_create(a):
    fields = {
        "project": {"key": a.project},
        "summary": a.summary,
        "issuetype": {"name": a.type},
    }
    if a.description:
        fields["description"] = adf(a.description)
    d = call("POST", "/rest/api/3/issue", {"fields": fields})
    print(json.dumps({"key": d["key"], "url": f"{BASE}/browse/{d['key']}"}, indent=2))


def cmd_comment(a):
    call("POST", f"/rest/api/3/issue/{a.key}/comment", {"body": adf(a.body)})
    print(json.dumps({"ok": True, "key": a.key}))


def cmd_transition(a):
    ts = call("GET", f"/rest/api/3/issue/{a.key}/transitions")["transitions"]
    m = next((t for t in ts if t["name"].lower() == a.name.lower()), None)
    if not m:
        sys.stderr.write(
            f"No transition {a.name!r}. Available: {[t['name'] for t in ts]}\n"
        )
        sys.exit(2)
    call(
        "POST",
        f"/rest/api/3/issue/{a.key}/transitions",
        {"transition": {"id": m["id"]}},
    )
    print(json.dumps({"ok": True, "key": a.key, "to": m["name"]}))


def cmd_assign(a):
    account_id = a.user
    if "@" in a.user:
        users = call("GET", "/rest/api/3/user/search", query={"query": a.user})
        if not users:
            sys.stderr.write(f"No user matches {a.user}\n")
            sys.exit(2)
        account_id = users[0]["accountId"]
    call("PUT", f"/rest/api/3/issue/{a.key}/assignee", {"accountId": account_id})
    print(json.dumps({"ok": True, "key": a.key, "assignee": account_id}))


def cmd_search(a):
    d = call(
        "GET",
        "/rest/api/3/search/jql",
        query={"jql": a.jql, "fields": "summary,status", "maxResults": a.max},
    )
    print(json.dumps(d.get("issues", []), indent=2))


def main():
    p = argparse.ArgumentParser(prog="jira")
    s = p.add_subparsers(dest="cmd", required=True)

    x = s.add_parser("list")
    x.add_argument("--jql")
    x.add_argument("--max", type=int, default=20)
    x.set_defaults(fn=cmd_list)

    x = s.add_parser("get")
    x.add_argument("key")
    x.set_defaults(fn=cmd_get)

    x = s.add_parser("create")
    x.add_argument("--project", required=True)
    x.add_argument("--summary", required=True)
    x.add_argument("--type", default="Task")
    x.add_argument("--description")
    x.set_defaults(fn=cmd_create)

    x = s.add_parser("comment")
    x.add_argument("key")
    x.add_argument("body")
    x.set_defaults(fn=cmd_comment)

    x = s.add_parser("transition")
    x.add_argument("key")
    x.add_argument("name")
    x.set_defaults(fn=cmd_transition)

    x = s.add_parser("assign")
    x.add_argument("key")
    x.add_argument("user")
    x.set_defaults(fn=cmd_assign)

    x = s.add_parser("search")
    x.add_argument("--jql", required=True)
    x.add_argument("--max", type=int, default=50)
    x.set_defaults(fn=cmd_search)

    args = p.parse_args()
    args.fn(args)


if __name__ == "__main__":
    main()
```

---

## Source code: the project `CLAUDE.md`

Save this as `CLAUDE.md` at the root of any project where you want Claude Code to manage Jira. Edit the `Conventions` block for your team.

```markdown
# Jira access

Use the `jira` CLI on PATH for all Jira operations. Do not suggest or use MCP for Jira.

Auth is handled via env vars (JIRA_BASE_URL, JIRA_EMAIL, JIRA_API_TOKEN). Never echo the token.

## Available commands

- `jira list [--jql "..."]` — default lists my open issues
- `jira get KEY` — detailed view of one issue
- `jira create --project KEY --summary "..." [--type Task|Bug|Story] [--description "..."]`
- `jira comment KEY "body"`
- `jira transition KEY "<status name>"`
- `jira assign KEY <email|accountId>`
- `jira search --jql "..."` — raw JQL

All commands return JSON on stdout. Parse it; don't regex it.

## Conventions (edit per team)

- Default project key: `PROJ`
- Default issue type: `Task`. Use `Bug` only for defects with a clear reproduction.
- When creating a ticket, include a short description and link the relevant file/commit if known.
- After creating or transitioning, print the issue URL back to me.
- For bulk queries, use `jira search --jql` once rather than looping `jira get`.
```

---

## Verification

In a **new** terminal (so env vars are loaded):

```bash
jira list
jira create --project PROJ --summary "Setup verification ticket" --type Task --description "Delete me."
```

Then inside `claude`:

> Find the ticket I just created titled "Setup verification ticket" and close it with a comment saying "verified".

Claude should run `jira search`, `jira comment`, and `jira transition` in sequence. If it does, you're good.

---

## Troubleshooting

**`HTTP 401` on every call.** Email or token is wrong. Remember Jira Cloud wants `email:token`, not `username:password`. Regenerate the token and retry.

**`HTTP 404` on `/rest/api/3/search/jql`.** Your site may be very old — try `/rest/api/3/search` instead. Swap the path in `cmd_list` and `cmd_search`.

**`jira: command not found`.** `~/.local/bin` isn't on PATH. Check with `echo $PATH` and fix your rc file, then open a new terminal.

**Native Windows (not WSL).** Put the script at `%USERPROFILE%\bin\jira.py` and create a wrapper `jira.cmd` that calls `python "%USERPROFILE%\bin\jira.py" %*`. Set env vars via **System Properties → Environment Variables**.

**IP allowlisting.** If your org restricts Atlassian Cloud to specific IPs, make sure you're on the VPN when you call the script.

**ADF errors on description/comment.** The helper only handles plain text. For bold, lists, or @mentions, extend `adf()` — the Atlassian Document Format reference is at https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/.

---

## Security notes

- The API token has your full Jira permissions. Treat it like a password.
- Never commit `CLAUDE.md` or any file with the token in it. The token lives only in the shell rc file, which is in your home directory and not tracked by repo VCS.
- If you leave the team or suspect exposure, revoke the token at https://id.atlassian.com/manage-profile/security/api-tokens.
- If your Jira admin supports scoped API tokens, use one scoped to just the projects you need.
- Jira ticket content (descriptions, comments) becomes input Claude reads. If external reporters can file tickets, treat their content as untrusted — prompt injection via ticket bodies is a real risk.
