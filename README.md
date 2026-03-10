# git-secret-scanner

Automatically block hardcoded secrets from ever reaching GitHub — on every `git commit` and `git push`, across **every repository** on your machine.

This works by installing two global Git hooks that run [Gitleaks](https://github.com/gitleaks/gitleaks) in the background. Once set up, you never need to configure anything per-project. It just works everywhere.

---

## Why This Exists

Accidentally committing secrets (API keys, database passwords, tokens) is one of the most common and damaging developer mistakes. This setup catches them **locally** — before they leave your machine — so you never have to deal with secret rotation, GitHub alerts, or security incidents.

---

## How It Works

Git supports **hooks** — scripts that run automatically at specific points in your workflow. This repo configures two of them globally:

| Hook | When it runs | What it does |
|---|---|---|
| `pre-commit` | Before every `git commit` | Extracts staged files into a temp folder and scans them with Gitleaks. If a secret is found, the commit is blocked and you are shown exactly which file and line caused it. |
| `pre-push` | Before every `git push` | Runs a full Gitleaks scan on the entire repository. If a secret is found anywhere, the push is blocked. |

Setting hooks **globally** (via `core.hooksPath`) means every Git repo on your machine is protected automatically — no need to add hooks to each project individually.

---

## Files in This Repo

| File | What it is | Where to place it on Windows |
|---|---|---|
| `.gitleaks.toml` | Custom ruleset that tells Gitleaks what to look for | `C:\Users\<user>\.gitleaks.toml` |
| `pre-commit` | Bash script — the pre-commit hook | `C:\Users\<user>\.git-hooks\pre-commit` |
| `pre-push` | Bash script — the pre-push hook | `C:\Users\<user>\.git-hooks\pre-push` |

---

## Windows Setup — Step by Step

### Prerequisites

Before you begin, install the following tools. Each one plays a specific role.

---

#### 1. Git for Windows

Git for Windows bundles **Git Bash** — a bash shell for Windows. The hook scripts are written in bash, so Git Bash is what actually runs them when you commit or push.

Download: https://git-scm.com/download/win

During install, leave all defaults as-is. Git Bash will be installed alongside Git automatically.

After install, verify Git is working:

```bash
git --version
```

---

#### 2. Gitleaks

Gitleaks is the tool that does the actual secret scanning. It reads your files, applies the rules from `.gitleaks.toml`, and reports any matches.

Install via **winget** (recommended — built into Windows 10/11):

```powershell
winget install gitleaks
```

Or via **Chocolatey** if you have it:

```powershell
choco install gitleaks
```

Or **manually**: download `gitleaks.exe` from [GitHub Releases](https://github.com/gitleaks/gitleaks/releases), then place it in `C:\Windows\System32\` or any folder that is on your system `PATH`.

Verify it works (open Git Bash or PowerShell):

```bash
gitleaks version
```

You should see a version number like `v8.x.x`. If you get "command not found", gitleaks is not on your PATH — revisit the install step.

---

### Step 1 — Create the global hooks directory

Git hooks need to live in a dedicated folder. You are creating this folder at a fixed location in your home directory so Git knows where to find it.

Open **PowerShell** and run:

```powershell
mkdir $HOME\.git-hooks
```

This creates `C:\Users\<your-username>\.git-hooks\`.

---

### Step 2 — Tell Git to use that folder for all repos

By default, Git looks for hooks inside each repo's `.git/hooks/` folder. This command overrides that globally — telling Git to look in `~/.git-hooks/` instead, for every repo on your machine.

```bash
git config --global core.hooksPath "$HOME/.git-hooks"
```

Confirm it was saved correctly:

```bash
git config --global --get core.hooksPath
```

Expected output:

```
C:/Users/<your-username>/.git-hooks
```

---

### Step 3 — Copy the hook scripts

Open **Git Bash** inside this repo's folder (right-click the folder → "Open Git Bash here") and run:

```bash
cp pre-commit ~/.git-hooks/pre-commit
cp pre-push   ~/.git-hooks/pre-push
```

> **Critical — no file extensions:** Windows sometimes adds `.txt` or other extensions to files. The hook files **must** be named exactly `pre-commit` and `pre-push` with no extension at all, otherwise Git will not recognise them and they will silently never run.
>
> To check: open `C:\Users\<user>\.git-hooks\` in File Explorer, go to **View → Show → File name extensions**, and confirm there is no extension on the files.

---

### Step 4 — Copy the Gitleaks config

The `.gitleaks.toml` file contains all the detection rules — what patterns to look for, what to ignore, and how severe each match is. Without it, Gitleaks falls back to its built-in defaults only.

In **PowerShell**:

```powershell
Copy-Item .gitleaks.toml "$HOME\.gitleaks.toml"
```

Or in **Git Bash**:

```bash
cp .gitleaks.toml ~/.gitleaks.toml
```

This places the config at `C:\Users\<user>\.gitleaks.toml`, which is where the hook scripts expect to find it.

---

### Step 5 — Verify the full setup

Run these checks to confirm everything is in place:

```bash
# Gitleaks is installed and accessible
gitleaks version

# Git is pointing to the right hooks folder
git config --global --get core.hooksPath

# Hook files exist in the right place
ls ~/.git-hooks/
```

Expected output for the last command:

```
pre-commit  pre-push
```

---

### Step 6 — Test it

The easiest way to confirm the hooks are working is to stage a file that contains a fake secret and try to commit it.

```bash
# Create a test file with a fake AWS key
echo 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE' > test-secret.txt

# Stage it
git add test-secret.txt

# Try to commit — this should be blocked
git commit -m "test"
```

You should see output like this:

```
🔍 Gitleaks: Scanning staged files for secrets...

Secrets found:

  ✗  test-secret.txt  →  line 1  →  aws-access-key-id

╔══════════════════════════════════════════════╗
║   ❌  SECRET DETECTED — COMMIT BLOCKED       ║
╚══════════════════════════════════════════════╝

  Fix:  Remove the secret from the file(s) above.
  Then: git add <file> && git commit again.
```

If you see this, the setup is working correctly. Clean up the test file:

```bash
git restore --staged test-secret.txt
rm test-secret.txt
```

---

## What Gets Detected

The `.gitleaks.toml` ruleset covers a wide range of secrets across common services:

| Category | What is detected |
|---|---|
| AWS | Access Key IDs (`AKIA…`), Secret Access Keys, Session Tokens, Account IDs |
| AWS RDS | Connection strings with embedded credentials, passwords, usernames |
| PostgreSQL | Connection URLs with credentials, passwords |
| MongoDB | Connection URIs with credentials, passwords |
| Redis | Passwords, auth tokens, connection URLs |
| RabbitMQ | Passwords, AMQP connection URLs |
| MinIO | Secret keys, access keys |
| Supabase | Service role keys, anon keys (JWT format), database passwords |
| Auth0 | Client secrets, client IDs, management API tokens |
| SMTP / Email | Passwords, connection URLs, Gmail app passwords |
| Google / GCP | API keys (`AIza…`) |
| JWT | Signing secrets |
| Generic | Any variable named `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`, `ACCESS_KEY`, etc. |
| Entropy-based | High-randomness strings assigned to any variable — catches secrets with non-standard names |
| JSON env files | Password, secret, token, and connection fields in `.env.json` style files |

### What is automatically ignored (allowlist)

To keep false positives low, the following are skipped automatically:

- Files in `tests/` or `test/` directories
- `.md`, `.example`, `.sample`, `.template`, `.dist` files
- Values that look like placeholders: `example`, `placeholder`, `your_key_here`, `changeme`, `xxxxxx`, `000000`
- Environment variable references: `${VAR}`, `$VAR`, `%VAR%` — these are safe references, not actual secrets

---

## Troubleshooting

**Hooks are not running at all (commit goes through without any scan output)**

The `core.hooksPath` setting is probably missing or wrong. Check it:

```bash
git config --global --get core.hooksPath
```

It must return a path. If it returns nothing, re-run Step 2. If it returns a path, confirm the hook files actually exist there with `ls ~/.git-hooks/`.

Also check the hook files have no extension — a file named `pre-commit.txt` will never run.

---

**`gitleaks: command not found`**

Gitleaks is not on your system PATH. Try:

```powershell
winget install gitleaks
```

Then close and reopen Git Bash. If it still doesn't work, find where `gitleaks.exe` was installed and add that folder to your system PATH via **System Properties → Environment Variables**.

---


**Hook files not being recognised (have wrong name)**

Open `C:\Users\<user>\.git-hooks\` in File Explorer. Go to **View → Show → File name extensions**. The files must be named `pre-commit` and `pre-push` — no `.txt`, no `.sh`, no anything. If they have an extension, rename them to remove it.

---

**Gitleaks is flagging something that is not a real secret (false positive)**

Add the value or file path to the `[allowlist]` section in `~/.gitleaks.toml`. For example, to ignore a specific file:

```toml
[allowlist]
paths = [
    '''path/to/your/file\.js'''
]
```

Or to ignore a specific string pattern:

```toml
[allowlist]
regexes = [
    '''your-non-secret-pattern'''
]
```

Then save the file — the change takes effect immediately on the next commit.
