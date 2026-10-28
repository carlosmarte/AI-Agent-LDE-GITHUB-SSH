# GitHub SSH setup on Linux (agent instructions)

Follow these steps in order. Commands marked **USER RUNS** must be handed to the
user to type themselves, prefixed with `!` in Claude Code (for example
`! gh auth login`). Commands marked **AGENT RUNS** are safe for you to execute.

---

## Step 0 — Check prerequisites

### 0a. Identify the distro

**AGENT RUNS**

```bash
. /etc/os-release && echo "$ID $VERSION_ID"
command -v apt-get dnf yum pacman zypper apk brew 2>/dev/null
```

Use the result to pick the right install commands below. If Homebrew on Linux
(`brew`) is present, `brew install gh git` works too, but prefer the distro's
own package manager.

### 0b. Git

**AGENT RUNS**

```bash
command -v git && git --version
```

If `git` is missing, suggest the matching line and wait for confirmation:

```bash
# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y git

# Fedora / RHEL / CentOS Stream
sudo dnf install -y git

# Arch / Manjaro
sudo pacman -S --needed git

# openSUSE
sudo zypper install git

# Alpine
sudo apk add git openssh-client
```

### 0c. GitHub CLI

**AGENT RUNS**

```bash
command -v gh && gh --version
```

If `gh` is missing, suggest the matching block. Debian/Ubuntu repos often ship
an old `gh`, so use GitHub's own apt repo:

```bash
# Debian / Ubuntu (official GitHub CLI repo)
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -nv -O- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt-get update && sudo apt-get install -y gh

# Fedora / RHEL / CentOS Stream
sudo dnf install -y gh

# Arch / Manjaro
sudo pacman -S --needed github-cli

# openSUSE
sudo zypper install gh

# Alpine (community repo)
sudo apk add github-cli

# Homebrew on Linux
brew install gh
```

### 0d. OpenSSH client

**AGENT RUNS**

```bash
command -v ssh ssh-add ssh-agent
```

Usually present. If missing: `openssh-client` (Debian/Ubuntu), `openssh-clients`
(Fedora), `openssh` (Arch / Alpine).

Do not proceed to Step 1 until `git --version`, `gh --version`, and `ssh -V`
all succeed.

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

On a headless machine with no browser, `gh` prints a URL and one-time code the
user can open on another device. Alternatively they can choose
`Paste an authentication token` with a PAT that has `repo`, `read:org`, and
`admin:public_key` scopes.

The key lands at `~/.ssh/id_ed25519` by default.

### 1c. Point git at `gh`

**AGENT RUNS**

```bash
gh auth setup-git
```

---

## Step 2 — Load the key into the SSH agent

Linux has no single Keychain. Pick the path that matches the user's setup.

### 2a. Find the key

**AGENT RUNS**

```bash
ls ~/.ssh/ | grep -v '\.pub$' | grep -E '^id_'
```

Use the key name from this output below. The default is `id_ed25519`.

### 2b. Is an agent already running?

**AGENT RUNS**

```bash
echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK"
ssh-add -l 2>&1
```

- `Could not open a connection to your authentication agent` → no agent; go to 2c.
- `The agent has no identities` → agent is running; go to 2d.
- A fingerprint is listed → the key is already loaded; skip to Step 3.

### 2c. Start an agent (only if none is running)

Desktop sessions (GNOME, KDE) normally start one for you via gnome-keyring or
ksshaskpass. On a bare shell or server, start one for this session:

**USER RUNS**

```
! eval "$(ssh-agent -s)"
```

### 2d. Add the key

**USER RUNS** (it prompts for the passphrase)

```
! ssh-add ~/.ssh/id_ed25519
```

### 2e. Make it persist

Append to `~/.ssh/config` so the key is added to the agent automatically the
first time it is used in a session:

**AGENT RUNS** — only append if the `Host github.com` block is not already there:

```bash
grep -q 'Host github.com' ~/.ssh/config 2>/dev/null || cat >> ~/.ssh/config <<'CFG'

Host github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/id_ed25519
CFG
chmod 600 ~/.ssh/config
```

Adjust `IdentityFile` if the key from 2a has a different name.

For the agent itself to survive logout on a server, offer one of:

- **systemd user service** (most modern distros):
  ```bash
  systemctl --user enable --now ssh-agent.service 2>/dev/null \
    || echo "No ssh-agent user unit shipped; see the 'keychain' option"
  echo 'export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"' >> ~/.bashrc
  ```
  Arch ships this unit; Debian/Ubuntu may not.
- **`keychain` package**: `sudo apt-get install keychain` (or distro equivalent),
  then add `eval "$(keychain --eval --quiet id_ed25519)"` to `~/.bashrc` or `~/.zshrc`.

On a desktop with gnome-keyring or KDE Wallet, none of this is needed. The
passphrase is stored on first use.

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
