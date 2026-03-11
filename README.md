# git-secret-scanner


Block hardcoded secrets from reaching GitHub — automatically, on every `git commit`, across **every repo on your machine**.

---

## What It Does

Runs [Gitleaks](https://github.com/gitleaks/gitleaks) as a global Git hook. Before each commit, it scans your staged files for secrets. If anything is found, the commit is blocked.

Set it up once — it works everywhere, no per-project config needed.

---

## Quick Setup (Windows)

### 1. Install Gitleaks

```powershell
winget install gitleaks
```

> **Important:** After installation, add `gitleaks` to your system PATH if it isn't already.
>
> - Open **System Properties → Advanced → Environment Variables**
> - Under **System variables**, select **Path → Edit → New**
> - Add the folder path where `gitleaks.exe` was installed
> - Click **OK**, then close and reopen your terminal
>
> Without this, `gitleaks: command not found` will appear and hooks will not run.

Verify:

```bash
gitleaks version
```

---

### 2. Clone this repo

```bash
git clone https://github.com/your-username/git-secret-scanner.git
cd git-secret-scanner
```

---

### 3. Run setup

Open **Git Bash** inside the cloned folder and run:

```bash
# Create global hooks directory
mkdir -p ~/.git-hooks

# Copy hooks
cp pre-commit ~/.git-hooks/pre-commit
cp pre-push   ~/.git-hooks/pre-push

# Copy Gitleaks config
cp .gitleaks.toml ~/.gitleaks.toml

# Tell Git to use the global hooks for every repo
git config --global core.hooksPath ~/.git-hooks
```

That's it. Every repo on your machine is now protected.

---

### 4. Verify

```bash
gitleaks version
git config --global --get core.hooksPath   # should print ~/.git-hooks or the full path
ls ~/.git-hooks/                            # should show: pre-commit  pre-push
```

---

### 5. Test it

```bash
echo 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE' > secret-test.txt
git add secret-test.txt
git commit -m "test"
```

You should see:

```
🔍 Gitleaks: Scanning staged files for secrets...

╔══════════════════════════════════════════════╗
║   ❌  SECRET DETECTED — COMMIT BLOCKED       ║
╚══════════════════════════════════════════════╝
```

Clean up:

```bash
git restore --staged secret-test.txt
rm secret-test.txt
```

---

## What Gets Detected

| Category | Examples |
|---|---|
| AWS | Access Key IDs, Secret Keys, Session Tokens |
| Databases | PostgreSQL, MySQL, MongoDB, Redis, RabbitMQ connection strings and passwords |
| Cloud / SaaS | Supabase, Auth0, MinIO, Google API keys |
| Email / SMTP | Passwords, connection URLs, Gmail app passwords |
| JWT | Signing secrets and keys |
| Generic | Any variable named `PASSWORD`, `SECRET`, `API_KEY`, `TOKEN`, etc. |
| Bare variables | `password = "..."`, `secret = "..."` — even without a prefix |
| Inline DB URLs | `psycopg2.connect("postgres://user:pass@host/db")` — no variable name needed |
| Private keys | PEM headers (`-----BEGIN RSA PRIVATE KEY-----`) |
| Bearer tokens | Hardcoded `Authorization: Bearer ...` values |
| Entropy-based | High-randomness strings assigned to any variable — catches secrets with unusual names |

## What Is Ignored (False Positive Prevention)

All file types are scanned — no directories or extensions are excluded.

| Ignored | Reason |
|---|---|
| `example`, `placeholder`, `changeme`, `xxxxxx` values | Clearly fake placeholder values |
| `${VAR}`, `$VAR`, `%VAR%` references | Safe env var references, not actual secrets |

---

## Updating Rules

All detection rules live in `~/.gitleaks.toml`. Edit that file directly — changes take effect on the next commit.

To update to the latest version of this repo's rules:

```bash
cp .gitleaks.toml ~/.gitleaks.toml
```

---

## Troubleshooting

**Hooks not running (commit goes through silently)**

```bash
git config --global --get core.hooksPath
```

If empty, re-run the `git config --global core.hooksPath ~/.git-hooks` command.
Also check the hook files have **no file extension** — `pre-commit.txt` will never run.

---

**`gitleaks: command not found`**

```powershell
winget install gitleaks
```

Close and reopen Git Bash. If it still fails, find `gitleaks.exe` and add its folder to your system PATH via **System Properties → Environment Variables**.

---

**False positive (something flagged that isn't a secret)**

Add it to the `[allowlist]` in `~/.gitleaks.toml`:

```toml
[allowlist]
regexes = [
    '''your-non-secret-pattern'''
]
```

Or to ignore a whole file:

```toml
[allowlist]
paths = [
    '''path/to/file\.js'''
]
```

Save and it takes effect immediately.
