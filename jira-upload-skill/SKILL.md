---
name: jira-upload-skill
description: Use when the user wants to upload, attach, or add a file to a Jira Cloud issue (e.g. "幫我把 X 附到 ABC-123", "上傳檔案到 Jira", "attach this file to the ticket", "add screenshot to a Jira issue"). Applies to Atlassian Jira Cloud (*.atlassian.net) via the REST API multipart endpoint /rest/api/3/issue/{issueKey}/attachments. Skip for Jira Server / Data Center (different endpoint and auth).
---

# Jira Cloud Attachment Upload

## Overview

Upload one or more files to a Jira Cloud issue by POSTing a multipart form to `/rest/api/3/issue/{issueKey}/attachments`, authenticated with **Email + Atlassian API Token** Basic Auth, with the mandatory `X-Atlassian-Token: no-check` header.

**Core principle:** Never inline the API token on the command line. Use `~/.netrc` (chmod 600) or an environment variable so the token does not leak into shell history or the process list.

## When to Use

Use this skill whenever the user says any of:
- "上傳檔案到 Jira ticket"、"把附件加到 ABC-123"、"幫我 attach this file to the ticket"
- "add a screenshot/log/PDF to ERP-54"
- "attach this report to the Jira issue"
- "上傳 [filename] 到 Jira"

**Skip this skill for:**
- Jira Server / Jira Data Center — different endpoint (`/rest/api/2/...`) and OAuth/cookie auth.
- Confluence attachments — different endpoint family.
- Atlassian MCP tool use — there is no first-class attachment upload tool in the public Atlassian MCP at time of writing; falling back to curl is the right move.

## Required Inputs (ALWAYS ask user if missing)

Before running anything, you MUST have all four of these. If any are missing, ask the user:

| Input | Example | Notes |
|---|---|---|
| Jira site host | `acme.atlassian.net` | Without `https://`. Look at the URL when the user opens a Jira ticket. |
| Login email | `alice@acme.com` | The Atlassian account login email, **not** display name. |
| Issue key | `PROJ-123` | Format is `{PROJECT_KEY}-{number}`. Visible in the ticket URL. |
| File path(s) | `/abs/path/to/file.pdf` | Absolute path. Can be multiple. |

API token is a **separate** concern handled in the next section — do not ask the user to paste the raw token into the chat; route it through `.netrc` or env var.

## Prerequisite: API Token (one-time setup)

If the user has not yet created an API token, point them to:

1. Open https://id.atlassian.com/manage-profile/security/api-tokens
2. Click **Create API token**, give it a label like `claude-jira-upload`
3. **Copy it immediately** — the value is only shown once
4. Format looks like `ATATT3xFf...=12A9F90D` (80+ chars)

**Security:** Treat the token like a password. Never paste it into chat, commits, shared docs, or `-u email:TOKEN` on the command line (it leaks into `history` and `ps`).

## Recommended: Configure ~/.netrc (do this once)

The skill's preferred auth pattern. Check if it already exists:

```bash
grep -l "atlassian.net" ~/.netrc 2>/dev/null && echo "netrc already configured"
```

If not, instruct the user to create it (replacing the placeholders with their own values):

```bash
# Append to ~/.netrc — REPLACE the placeholder values
cat >> ~/.netrc <<'EOF'
machine YOUR_SITE.atlassian.net
  login YOUR_EMAIL@example.com
  password YOUR_ATLASSIAN_API_TOKEN
EOF
chmod 600 ~/.netrc
```

After this, all `curl --netrc` calls to that host are authenticated automatically.

## Core Upload Command

Once `.netrc` is set up, the upload is a single curl. Substitute `SITE`, `ISSUE`, and `FILE`:

```bash
SITE='YOUR_SITE.atlassian.net'
ISSUE='PROJ-123'
FILE='/absolute/path/to/file.pdf'

curl -sS -X POST \
  -H "X-Atlassian-Token: no-check" \
  --netrc \
  -F "file=@${FILE}" \
  "https://${SITE}/rest/api/3/issue/${ISSUE}/attachments"
```

**Why each flag is mandatory:**

| Flag | Reason |
|---|---|
| `-X POST` | Endpoint is POST-only. |
| `-H "X-Atlassian-Token: no-check"` | **Required.** Bypasses Jira CSRF check; missing this returns 403. |
| `--netrc` | Sources `email:token` from `~/.netrc` so it never appears on the command line. |
| `-F "file=@PATH"` | Multipart form field; Jira requires the field name to be exactly `file`. The `@` makes curl read the file contents. |
| `-sS` | Silent mode but still prints errors. |

### Successful response

HTTP 200 with a JSON array. Each element describes one uploaded attachment:

```json
[
  {
    "id": "12054",
    "filename": "file.pdf",
    "size": 5258,
    "mimeType": "application/pdf",
    "content": "https://YOUR_SITE.atlassian.net/rest/api/3/attachment/content/12054",
    "created": "2026-05-16T23:22:20.942+0800"
  }
]
```

Confirm success by checking `filename` matches what the user uploaded.

## Multi-file Upload

Pass multiple `-F file=@...` in the same request:

```bash
curl -sS -X POST \
  -H "X-Atlassian-Token: no-check" \
  --netrc \
  -F "file=@/path/to/spec.md" \
  -F "file=@/path/to/diagram.png" \
  -F "file=@/path/to/data.csv" \
  "https://${SITE}/rest/api/3/issue/${ISSUE}/attachments"
```

Response is a JSON array with one element per file.

## Alternative: Environment Variable Auth (CI-friendly)

When `.netrc` is not available (CI, throwaway shell), use env vars instead — still avoids inlining the token:

```bash
export ATLASSIAN_EMAIL='alice@acme.com'
export ATLASSIAN_TOKEN='ATATT3xFf...=12A9F90D'

curl -sS -X POST \
  -H "X-Atlassian-Token: no-check" \
  -u "${ATLASSIAN_EMAIL}:${ATLASSIAN_TOKEN}" \
  -F "file=@${FILE}" \
  "https://${SITE}/rest/api/3/issue/${ISSUE}/attachments"
```

`-u "EMAIL:TOKEN"` is still safer than literal inlining because the token only appears in the env, not in the command string saved to history. **Never** paste a literal token into `-u` directly.

## Verify Upload via API

```bash
curl -sS --netrc \
  "https://${SITE}/rest/api/3/issue/${ISSUE}?fields=attachment" \
  | python3 -m json.tool
```

## Download / Delete an Attachment

```bash
# Download (id is from the upload response or issue query)
curl -sS --netrc -L -o downloaded.bin \
  "https://${SITE}/rest/api/3/attachment/content/12054"

# Delete
curl -sS -X DELETE --netrc \
  "https://${SITE}/rest/api/3/attachment/12054"
```

## Common Errors

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong email/token or token revoked | Verify email is the Atlassian login email (not display name); regenerate the token. |
| `403 Forbidden` + message contains `XSRF` | Missing `X-Atlassian-Token: no-check` header | Add the header. |
| `404 Issue Does Not Exist` | Wrong issue key OR no read permission | Verify the user can open the ticket in the browser. |
| `413 Request Entity Too Large` | File exceeds Jira's per-file cap (default 10 MB; admin-configurable) | Compress, split, or ask admin to raise the limit. |
| curl prints nothing AND no attachment appears | `-sS` swallowed the error or network blocked | Re-run with `-v` to see the full HTTP exchange. |

## Security Checklist

- [ ] Token stored in `~/.netrc` (chmod 600) or an env var — never in a script or commit.
- [ ] Never `-u email:TOKEN` with a literal token on the command line.
- [ ] If the token ever appears in chat, a shared doc, a screenshot, or a commit — revoke it on the Atlassian token management page immediately and issue a new one.
- [ ] Use separate tokens for different purposes (one for CLI, one for CI) so revocation is scoped.
- [ ] Rotate tokens periodically (every 6 months is a sane default).

## Workflow When Invoked

When the user asks to upload a file:

1. Confirm the four required inputs (site, email, issue key, file path). Ask for any missing.
2. Check whether `~/.netrc` already has an entry for the site (`grep -l atlassian.net ~/.netrc`). If not, walk the user through setting it up.
3. Run the core curl command.
4. Parse the JSON response and confirm `filename` matches.
5. If non-200, look up the symptom in the Common Errors table and apply the fix.

## References

- Jira Cloud REST API — Issue attachments: https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-attachments/
- Atlassian API token management: https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/
