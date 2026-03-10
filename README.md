# git-secret-scanner

Global Git hooks that automatically scan every `git commit` and `git push` for hardcoded secrets using [Gitleaks](https://github.com/gitleaks/gitleaks).

Secrets are blocked **before** they ever reach GitHub — across every repository on your machine, with no per-project setup needed.

---

## How It Works

| Hook | Trigger | What it does |
|---|---|---|
| `pre-commit` | Every `git commit` | Scans only staged files; blocks the commit if a secret is found |
| `pre-push` | Every `git push` | Full repo scan; blocks the push if a secret is found |

---

## Files in This Repo

| File | Place it at (Linux / macOS) | Place it at (Windows) |
|---|---|---|
| `.gitleaks.toml` | `~/.gitleaks.toml` | `C:\Users\<user>\.gitleaks.toml` |
| `pre-commit` | `~/.git-hooks/pre-commit` | `C:\Users\<user>\.git-hooks\pre-commit` |
| `pre-push` | `~/.git-hooks/pre-push` | `C:\Users\<user>\.git-hooks\pre-push` |

---

## Linux / macOS Setup

### Prerequisites

- **Gitleaks** — install via your package manager:
  ```bash
  # Fedora / RHEL
  sudo dnf install gitleaks

  # macOS
  brew install gitleaks
  ```
- **Python 3** — used by the pre-commit hook to parse the JSON report (usually pre-installed)

### Step 1 — Create global hooks directory

```bash
mkdir -p ~/.git-hooks
```

### Step 2 — Tell Git to use it globally

```bash
git config --global core.hooksPath ~/.git-hooks
```

Verify:

```bash
git config --global --get core.hooksPath
```

### Step 3 — Copy the hook scripts

```bash
cp pre-commit ~/.git-hooks/pre-commit
cp pre-push   ~/.git-hooks/pre-push
```

### Step 4 — Make them executable

```bash
chmod +x ~/.git-hooks/pre-commit
chmod +x ~/.git-hooks/pre-push
```

### Step 5 — Copy the Gitleaks config

```bash
cp .gitleaks.toml ~/.gitleaks.toml
```

### Step 6 — Verify

```bash
gitleaks version
git config --global --get core.hooksPath
```

---

## Windows Setup

Git for Windows ships with **Git Bash**, which runs the same bash scripts as Linux. No changes to the hook files are needed.

### Prerequisites

- **Git for Windows** (includes Git Bash): https://git-scm.com/download/win
- **Gitleaks** — choose one install method:
  ```powershell
  # Option A — winget (recommended)
  winget install gitleaks

  # Option B — Chocolatey
  choco install gitleaks
  ```
  Option C — Manual: download `gitleaks.exe` from [GitHub Releases](https://github.com/gitleaks/gitleaks/releases) and place it in a folder on your `PATH` (e.g. `C:\Windows\System32` or `C:\tools\`).

- **Python 3**: https://python.org — required for JSON parsing in the pre-commit hook. Make sure `python3` is on your `PATH` (tick "Add Python to PATH" during install).

Verify both are working in Git Bash:

```bash
gitleaks version
python3 --version
```

### Step 1 — Create global hooks directory

Run in **PowerShell** or **Git Bash**:

```powershell
# PowerShell
mkdir $HOME\.git-hooks
```

```bash
# Git Bash
mkdir -p ~/.git-hooks
```

### Step 2 — Tell Git to use it globally

```bash
git config --global core.hooksPath "$HOME/.git-hooks"
```

Verify:

```bash
git config --global --get core.hooksPath
```

### Step 3 — Copy the hook scripts

In **Git Bash** (from this repo's directory):

```bash
cp pre-commit ~/.git-hooks/pre-commit
cp pre-push   ~/.git-hooks/pre-push
```

> **Important:** The files must have **no file extension**. If Windows saves them as `pre-commit.txt` the hooks will not run.

### Step 4 — Copy the Gitleaks config

```bash
# Git Bash
cp .gitleaks.toml ~/.gitleaks.toml
```

```powershell
# PowerShell
Copy-Item .gitleaks.toml "$HOME\.gitleaks.toml"
```

### Step 5 — Verify

```bash
gitleaks version
git config --global --get core.hooksPath
```

---

## Linux vs Windows — Quick Comparison

| Item | Linux / macOS | Windows |
|---|---|---|
| Install gitleaks | `sudo dnf install gitleaks` | `winget install gitleaks` |
| Hooks directory | `~/.git-hooks/` | `C:\Users\<user>\.git-hooks\` |
| Hook scripts | bash (`#!/bin/bash`) | Same bash scripts via Git Bash |
| Config file | `~/.gitleaks.toml` | `C:\Users\<user>\.gitleaks.toml` |
| Make executable | `chmod +x pre-commit` | Not needed in Git Bash |
| git config command | Same on both platforms | Same on both platforms |

---

## What Gets Detected

The `.gitleaks.toml` config covers:

- **AWS** — Access Keys, Secret Keys, Session Tokens, Account IDs
- **AWS RDS** — connection strings, passwords, usernames
- **PostgreSQL** — connection URLs, passwords
- **MongoDB** — connection URIs, passwords
- **Redis** — passwords, auth tokens, connection URLs
- **RabbitMQ** — passwords, AMQP connection URLs
- **MinIO** — secret keys, access keys
- **Supabase** — service role keys, database passwords
- **Auth0** — client secrets, client IDs, management tokens
- **SMTP / Mailer** — passwords, connection URLs, Gmail app passwords
- **Google / GCP** — API keys
- **JWT** — signing secrets
- **Generic fallbacks** — any `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN` variable
- **Entropy-based detection** — high-randomness strings even without a recognizable key name
- **JSON env files** — `.env.json` style password/secret/connection fields

### Global Allowlist

The following are automatically skipped to reduce false positives:

- Test directories (`tests/`, `test/`)
- Documentation files (`.md`, `.example`, `.sample`, `.template`, `.dist`)
- Placeholder values (`example`, `placeholder`, `your_key_here`, `changeme`, `xxxxxx`, `000000`)
- Environment variable references (`${VAR}`, `$VAR`, `%VAR%`)

---

## Pre-commit Hook Output

When a secret is detected, the commit is blocked with a clear report:

```
🔍 Gitleaks: Scanning staged files for secrets...

Secrets found:

  ✗  .env  →  line 3  →  aws-access-key-id

╔══════════════════════════════════════════════╗
║   ❌  SECRET DETECTED — COMMIT BLOCKED       ║
╚══════════════════════════════════════════════╝

  Fix:  Remove the secret from the file(s) above.
  Then: git add <file> && git commit again.
```

---

## Troubleshooting

**Hooks are not running**

```bash
git config --global --get core.hooksPath
# Should return: /home/<user>/.git-hooks  (or Windows equivalent)
```

**`gitleaks: command not found`**

Make sure gitleaks is installed and on your `PATH`:

```bash
which gitleaks
gitleaks version
```

**`python3: command not found` (Windows)**

Install Python 3 from https://python.org and tick "Add Python to PATH" during setup. Then restart Git Bash.

**Hook files not executable (Linux / macOS)**

```bash
chmod +x ~/.git-hooks/pre-commit ~/.git-hooks/pre-push
```

**Hook file has wrong name (Windows)**

Open `C:\Users\<user>\.git-hooks\` in File Explorer. Make sure the files are named exactly `pre-commit` and `pre-push` with no extension. Enable "File name extensions" in Explorer's View menu to check.
