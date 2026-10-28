# GitHub SSH setup on Windows (agent instructions)

Follow these steps in order. Commands are written for **PowerShell**. Commands
marked **USER RUNS** must be handed to the user to type themselves, prefixed
with `!` in Claude Code (for example `! gh auth login`). Commands marked
**AGENT RUNS** are safe for you to execute.

---

## Step 0 — Check prerequisites

### 0a. A package manager

Homebrew is not available on Windows. `winget` ships with Windows 10 (1809+) and
Windows 11 and is the default here.

**AGENT RUNS**

```powershell
Get-Command winget -ErrorAction SilentlyContinue
```

If `winget` is missing, tell the user to install "App Installer" from the
Microsoft Store, or offer Scoop / Chocolatey alternatives below.

### 0b. Git

**AGENT RUNS**

```powershell
Get-Command git -ErrorAction SilentlyContinue; git --version
```

If `git` is missing, suggest one of these and wait for confirmation:

```powershell
# winget (preferred)
winget install --id Git.Git -e --source winget

# Scoop
scoop install git

# Chocolatey
choco install git
```

Tell the user to open a **new** PowerShell window afterwards so `PATH` picks it up.

### 0c. GitHub CLI

**AGENT RUNS**

```powershell
Get-Command gh -ErrorAction SilentlyContinue; gh --version
```

If `gh` is missing, suggest:

```powershell
# winget (preferred)
winget install --id GitHub.cli

# Scoop
scoop install gh

# Chocolatey
choco install gh
```

Again, a new PowerShell window is needed after installing.

### 0d. OpenSSH client

Windows 10/11 ship the OpenSSH client as an optional feature. Check:

**AGENT RUNS**

```powershell
Get-Command ssh -ErrorAction SilentlyContinue
Get-Service ssh-agent -ErrorAction SilentlyContinue
```

If `ssh` or the `ssh-agent` service is missing, the user needs to enable the
feature (requires an elevated PowerShell):

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Do not proceed to Step 1 until `git --version`, `gh --version`, and `ssh -V`
all succeed.

---

## Step 1 — Authenticate with `gh` over SSH

### 1a. Check the current state

**AGENT RUNS**

```powershell
gh auth status 2>&1
Get-ChildItem "$env:USERPROFILE\.ssh\*.pub" -ErrorAction SilentlyContinue
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
| Generate a new SSH key to add to your GitHub account? | `Yes` if no `.pub` files were listed. Otherwise pick the existing key. |
| Enter a passphrase for your new SSH key | Recommend a passphrase. Step 2 stops it from being asked repeatedly. |
| Title for your SSH key | Something like `<machine-name> (gh)` |
| How would you like to authenticate? | `Login with a web browser` |

The key lands at `%USERPROFILE%\.ssh\id_ed25519` by default.

### 1c. Point git at `gh`

**AGENT RUNS**

```powershell
gh auth setup-git
```

---

## Step 2 — Load the key into the Windows SSH agent

### 2a. Start the agent service and make it automatic

**USER RUNS** (needs an elevated PowerShell the first time)

```
! Get-Service ssh-agent | Set-Service -StartupType Automatic; Start-Service ssh-agent
```

If this fails with an access error, tell the user to open PowerShell
"Run as administrator" and run the same line there once.

### 2b. Add the key

**USER RUNS** (it prompts for the passphrase)

```
! ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

The Windows agent service persists keys across reboots, so there is no
Keychain equivalent to configure. The passphrase is entered once.

### 2c. Make git use the Windows OpenSSH, not Git for Windows' bundled one

Git for Windows bundles its own `ssh.exe`, which does **not** talk to the
Windows `ssh-agent` service. Point git at the system OpenSSH so the key you just
added is actually used:

**AGENT RUNS**

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

Skip this if the user already has `core.sshCommand` set to something else on
purpose (check with `git config --global core.sshCommand`).

---

## Step 3 — Verify

**AGENT RUNS**

```powershell
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
If `git push` still prompts for a passphrase after `ssh-add -l` shows the key,
Step 2c was skipped or is pointing at the wrong `ssh.exe`.
