# jira-upload-skill

> 🌐 **Language:** [繁體中文](README.md) · English

A [Claude Code](https://claude.com/claude-code) skill that teaches Claude how to upload files to **Jira Cloud** issues via the REST API — with secure API token handling (the token never appears on the command line as `-u email:TOKEN`).

Works for both audiences:

- **Claude Code users** — install as a skill; Claude handles the upload conversationally.
- **Anyone else** — [`jira-upload-skill/SKILL.md`](jira-upload-skill/SKILL.md) doubles as a complete standalone `curl` how-to guide.

---

## What it does

Once installed, you can say to Claude:

> Upload `~/Downloads/report.pdf` to `PROJ-123`

Claude will:

1. Ask for any missing pieces (site, login email, issue key, file path)
2. Check whether `~/.netrc` is already configured for your Jira site, and walk you through setup if not
3. Run the curl upload
4. Confirm the attachment landed by parsing the JSON response

The skill enforces secure token handling — your Atlassian API token is sourced from `~/.netrc` (chmod 600) or an environment variable, **never** inlined on the command line.

---

## Install (Claude Code)

```bash
# Clone or download this repo
git clone https://github.com/<your-username>/jira-upload-skill.git
cd jira-upload-skill

# Copy the skill folder into Claude Code's skills directory
mkdir -p ~/.claude/skills
cp -r jira-upload-skill ~/.claude/skills/
```

Restart Claude Code (or start a new session). The skill is auto-discovered by name — just ask Claude to upload something to Jira and the skill will fire.

Verify the install:

```bash
ls ~/.claude/skills/jira-upload-skill/SKILL.md
```

---

## One-time prerequisites

You need **two** things before first use:

### 1. Atlassian API token

1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Click **Create API token**, give it a label (e.g. `claude-jira-upload`)
3. Copy the token immediately — it's only shown once

### 2. `~/.netrc` entry

```bash
cat >> ~/.netrc <<'EOF'
machine YOUR_SITE.atlassian.net
  login YOUR_EMAIL@example.com
  password YOUR_ATLASSIAN_API_TOKEN
EOF
chmod 600 ~/.netrc
```

After this, every `curl --netrc` call to that host is authenticated automatically.

---

## Use without Claude Code

The skill doubles as a standalone curl reference. The most-used one-liner:

```bash
curl -sS -X POST \
  -H "X-Atlassian-Token: no-check" \
  --netrc \
  -F "file=@/path/to/file.pdf" \
  "https://YOUR_SITE.atlassian.net/rest/api/3/issue/PROJ-123/attachments"
```

Full guide (multi-file upload, env-var auth, error troubleshooting, download/delete): see [`jira-upload-skill/SKILL.md`](jira-upload-skill/SKILL.md).

---

## Scope

- ✅ **Jira Cloud** (`*.atlassian.net`) via REST API v3
- ❌ Jira Server / Data Center — endpoints and auth differ; this skill won't work for those.
- ❌ Confluence attachments — different endpoint family.

---

## Security notes

- Treat the API token like a password — never commit it, paste it into chat, or pass it inline on the command line.
- If a token is ever exposed, revoke it immediately at the [token management page](https://id.atlassian.com/manage-profile/security/api-tokens) and issue a new one.
- Use separate tokens for different purposes (CLI vs CI) so revocation is scoped.
- Rotate tokens every 6 months as a baseline.

---

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. If you hit a Jira edge case (custom CSRF settings, attachment size quirks, regional endpoints) that breaks the recipe, please open an issue.
