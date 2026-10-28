# GitHub SSH setup on macOS (agent instructions)

Follow these steps in order. Commands marked **USER RUNS** must be handed to the
user to type themselves, prefixed with `!` in Claude Code (for example
`! gh auth login`). Commands marked **AGENT RUNS** are safe for you to execute.

---

## Step 0 — Check prerequisites

### 0a. Homebrew

**AGENT RUNS**

```bash
command -v brew && brew --version
```

If that fails, **check the standard install locations before declaring it
missing**. The agent's shell often runs without the user's login profile, so
`/opt/homebrew/bin` may simply not be on `PATH`:

```bash
ls /opt/homebrew/bin/brew /usr/local/bin/brew 2>/dev/null
```

- If one of those exists, Homebrew **is** installed. For the rest of this
  session prefix with the full path (`/opt/homebrew/bin/brew`), and tell the
  user to add this line to `~/.zprofile` so future shells find it:

  ```bash
  eval "$(/opt/homebrew/bin/brew shellenv)"
  ```

  On Intel Macs the path is `/usr/local/bin/brew` instead.

- If neither exists, tell the user:

> Homebrew is not installed. Install it with the official script, then re-run
> this check:
>
> ```bash
> /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
> ```
>
> On Apple Silicon, follow the post-install note to add
> `eval "$(/opt/homebrew/bin/brew shellenv)"` to `~/.zprofile`.

Stop and wait for the user to confirm before continuing.

### 0b. Git

**AGENT RUNS**

```bash
command -v git && git --version
```

If `git` is missing, suggest one of these and wait for confirmation:

```bash
# Preferred: Homebrew
brew install git

# Alternative: Apple Command Line Tools (also provides git)
xcode-select --install
```

### 0c. GitHub CLI

**AGENT RUNS**

```bash
command -v gh && gh --version
brew list --versions gh
```

As with `brew`, if `command -v gh` fails, check the Homebrew prefix directly
before concluding it is missing:

```bash
ls /opt/homebrew/bin/gh /usr/local/bin/gh 2>/dev/null
```

If it exists there, it is the same `PATH` problem from 0a. Use the full path
for this session and point the user at the `shellenv` fix above.

If `gh` is genuinely missing, or if `brew list` shows it was not installed via
Homebrew, suggest:

```bash
brew install gh
```

If `gh` exists but is not Homebrew-managed, do not uninstall it. Just note it
and continue.

Do not proceed to Step 1 until `git --version` and `gh --version` both succeed.

---

## Step 1 — Authenticate with `gh` over SSH

### 1a. Check the current state

**AGENT RUNS**

```bash
gh auth status 2>&1
ls -la ~/.ssh/*.pub 2>/dev/null
```

- If `gh auth status` reports a logged-in account **with `Git operations protocol: ssh`**, skip to Step 1c.
- If it reports `https`, continue with 1b; `gh auth login` will switch the protocol.

### 1b. Log in

**USER RUNS**

```
! gh auth login
```

Tell the user exactly what to pick at each prompt:

| Prompt | Answer |
|---|---|
| What account do you want to log into? | `GitHub.com` |
| What is your preferred protocol for Git operations? | `SSH` |
| Generate a new SSH key to add to your GitHub account? | `Yes` if `ls ~/.ssh/*.pub` showed nothing. Otherwise pick the existing key. |
| Enter a passphrase for your new SSH key | Recommend a passphrase. Step 2 stops it from being asked repeatedly. |
| Title for your SSH key | Something like `<machine-name> (gh)` |
| How would you like to authenticate? | `Login with a web browser` |

`gh` prints a one-time code, opens the browser, and uploads the public key to
the user's GitHub account. The key lands at `~/.ssh/id_ed25519` by default.

### 1c. Point git at `gh`

**AGENT RUNS**

```bash
gh auth setup-git
```

This configures git to use `gh` as its credential helper for any remaining HTTPS
remotes. SSH remotes do not need a credential helper, but running it is
harmless and keeps HTTPS clones working too.

---

## Step 2 — Load the key into the agent and the macOS Keychain

### 2a. Find the key

**AGENT RUNS**

```bash
ls ~/.ssh/ | grep -v '\.pub$' | grep -E '^id_'
```

Use the key name from this output below. The default is `id_ed25519`.

### 2b. Add it to the agent, storing the passphrase in Keychain

**USER RUNS** (it prompts for the passphrase)

```
! ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

`--apple-use-keychain` saves the passphrase in the login Keychain so it is
only typed once. On macOS older than 12 the flag is spelled `-K` instead.

### 2c. Make it persist across reboots

**AGENT RUNS** — only append if the `Host github.com` block is not already there:

```bash
grep -q 'Host github.com' ~/.ssh/config 2>/dev/null || cat >> ~/.ssh/config <<'CFG'

Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
CFG
chmod 600 ~/.ssh/config
```

Adjust `IdentityFile` if the key from 2a has a different name.

---

## Step 3 — Verify

**AGENT RUNS**

```bash
gh auth status
ssh-add -l
ssh -T git@github.com 2>&1
```

Expected results:

- `gh auth status` shows the account and `Git operations protocol: ssh`.
- `ssh-add -l` lists the key fingerprint.
- `ssh -T` prints `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.` An exit code of 1 here is normal.

If `ssh -T` asks to confirm the host fingerprint, tell the user to answer `yes`.
If it says `Permission denied (publickey)`, run `gh ssh-key list` to confirm the
key was uploaded and `ssh -vT git@github.com` to see which key is being offered.
