# SSH Agent: share a key with every shell, one shell, or one command

Run every command yourself. Nothing here starts, adds or removes a key automatically.
Replace `example_ed25519` with your key file name.

Three flows:

| Flow | Who can use the key | How it ends |
|---|---|---|
| 1. Shared | every shell (and tools/agents running as you) | you run `ssh-add -D` |
| 2. This shell only | only the shell you started | you close that shell |
| 3. Single use | only one command | the command finishes |

`~/.ssh/config` should say `AddKeysToAgent no`, so typing the passphrase during a push never stores the key by itself:

```
Host github.com gitlab.com
    User git
    IdentityFile ~/.ssh/example_ed25519
    IdentitiesOnly yes
    AddKeysToAgent no
```

---

# Part 1: Arch Linux

## Setup (once)

```bash
sudo pacman -S --needed openssh
```

The `ssh-agent` user service comes with the `openssh` package. Do **not** `enable` it (that would start it at every login); flow 1 starts and stops it by hand.

Point every new shell at that agent's fixed socket, unless the shell already has its own agent (flow 2):

```bash
echo '[ -z "$SSH_AUTH_SOCK" ] && export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"' >> ~/.bashrc
```

```bash
source ~/.bashrc
```

```bash
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/example_ed25519
```

- `[ -z "$SSH_AUTH_SOCK" ]`: true when the variable is empty, meaning this shell has no agent yet
- `&&`: run the next command only if the test was true
- `export SSH_AUTH_SOCK=...`: tells `ssh`, `ssh-add` and `git` where the agent's socket is (`/run/user/<uid>/ssh-agent.socket`)
- This line **does not start** an agent. It only points at the shared one.

## Flow 1: shared by every shell

1. Start the agent

```bash
systemctl --user start ssh-agent
```

2. Add the key (asks for the passphrase)

```bash
ssh-add ~/.ssh/example_ed25519
```

Or add it with an expiry (`30m`, `1h`, `2h`...):

```bash
ssh-add -t 1h ~/.ssh/example_ed25519
```

3. Push from any shell

```bash
git push origin main
```

4. Remove the key and stop the agent

```bash
ssh-add -D
```

```bash
systemctl --user stop ssh-agent
```

Check what is loaded at any time:

```bash
ssh-add -l
```

A shell opened without the `.bashrc` line can join the shared agent by hand:

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```

## Remove keys from the agent

These remove keys from the **agent only**. The key files stay on disk.

See what is loaded first (each line ends with the key's comment, and `-L` shows the full public keys):

```bash
ssh-add -l
```

**All keys**

```bash
ssh-add -D
```

**One key**: give the key file (the private key or its `.pub`)

```bash
ssh-add -d ~/.ssh/example_ed25519
```

```bash
ssh-add -d ~/.ssh/example_ed25519.pub
```

**Several keys**: list them in one command

```bash
ssh-add -d ~/.ssh/example_ed25519 ~/.ssh/example-work_ed25519 ~/.ssh/example-server_ed25519
```

**Check** that they are gone (`The agent has no identities.` when empty):

```bash
ssh-add -l
```

- `-d` needs the public part of the key. It reads it from the `.pub` file next to the key, or from the key file itself (ed25519 keys keep it there unencrypted), so it does not ask for the passphrase.
- If the key file is already deleted or moved, `-d` cannot find it. Use the `.pub` if you still have it, otherwise `-D`.
- `ssh-add -d` with no file removes the default keys only (`~/.ssh/id_ed25519`, `~/.ssh/id_rsa`, ...).

## Flow 2: this shell only

1. Start a new shell that owns its own agent

```bash
ssh-agent bash
```

2. Add the key

```bash
ssh-add ~/.ssh/example_ed25519
```

3. Push

```bash
git push origin main
```

4. Close the shell. The agent and the key go with it.

```bash
exit
```

## Flow 3: single use

```bash
ssh -i ~/.ssh/example_ed25519 -o IdentitiesOnly=yes -o IdentityAgent=none -T git@github.com
```

```bash
GIT_SSH_COMMAND='ssh -i ~/.ssh/example_ed25519 -o IdentitiesOnly=yes -o IdentityAgent=none' git push origin main
```

---

# Part 2: Windows (Git Bash + Windows OpenSSH)

On Windows, Git Bash ships its own `ssh` and `ssh-add`, which cannot talk to the Windows `ssh-agent` service. Flow 1 therefore uses the Windows programs by full path. Flow 2 uses the Git Bash ones.

## Setup (once)

**PowerShell (Admin)**

```powershell
winget install Microsoft.OpenSSH.Preview
```

Choose `Manual` (start it yourself each time) or `Automatic` (starts at boot):

```powershell
Set-Service ssh-agent -StartupType Manual
```

**Git Bash:** make git use the Windows `ssh.exe`, so it sees the Windows agent:

```bash
git config --global core.sshCommand '"C:/Program Files/OpenSSH/ssh.exe"'
```

```bash
git config --global --get core.sshCommand
```

If OpenSSH came with Windows instead of winget, use `C:/Windows/System32/OpenSSH/` in place of `C:/Program Files/OpenSSH/` everywhere below.

## Flow 1: shared by every shell

1. Start the agent (**PowerShell Admin**)

```powershell
Start-Service ssh-agent
```

2. Add the key (Git Bash, asks for the passphrase)

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" "$(cygpath -w ~/.ssh/example_ed25519)"
```

- `"/c/Program Files/OpenSSH/ssh-add.exe"`: the Windows ssh-add. A plain `ssh-add` in Git Bash is the Git Bash one.
- `cygpath -w`: turns `/c/Users/...` into `C:\Users\...`, which Windows programs need.
- In PowerShell the same step is: `ssh-add $HOME\.ssh\example_ed25519`

3. Push from any Git Bash

```bash
git push origin main
```

4. Remove the key

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -D
```

Then stop the agent (**PowerShell Admin**):

```powershell
Stop-Service ssh-agent
```

Check what is loaded:

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -l
```

Test the login:

```bash
"/c/Program Files/OpenSSH/ssh.exe" -T git@github.com
```

> ⚠️ The Windows agent saves added keys (encrypted) in your profile and **keeps them after a reboot** until you run `-D`. `-t` (expiry) is not available on Windows.

## Remove keys from the agent

These remove keys from the **agent only**. The key files stay on disk. Same as Arch, but with the Windows `ssh-add.exe` and `cygpath -w` paths.

See what is loaded first:

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -l
```

**All keys**

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -D
```

**One key**

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -d "$(cygpath -w ~/.ssh/example_ed25519)"
```

**Several keys**: one `cygpath -w` per file

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -d "$(cygpath -w ~/.ssh/example_ed25519)" "$(cygpath -w ~/.ssh/example-work_ed25519)" "$(cygpath -w ~/.ssh/example-server_ed25519)"
```

**Check**:

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -l
```

The same in **PowerShell** (no `cygpath` needed):

```powershell
ssh-add -D
```

```powershell
ssh-add -d $HOME\.ssh\example_ed25519
```

```powershell
ssh-add -d $HOME\.ssh\example_ed25519 $HOME\.ssh\example-work_ed25519 $HOME\.ssh\example-server_ed25519
```

On Windows a key removed with `-d` / `-D` is also erased from the saved copy in your profile, so it does **not** come back after a reboot.

In flow 2 (`ssh-agent bash`) use the plain Git Bash `ssh-add -d ~/.ssh/...` / `ssh-add -D`, since that agent belongs to the shell.

## Flow 2: this shell only

1. Start a new Git Bash shell that owns its own agent

```bash
ssh-agent bash
```

2. Make git in this shell use the Git Bash `ssh`, which sees this agent

```bash
export GIT_SSH_COMMAND=ssh
```

3. Add the key

```bash
ssh-add ~/.ssh/example_ed25519
```

4. Push

```bash
git push origin main
```

5. Close the shell. The agent and the key go with it.

```bash
exit
```

## Flow 3: single use

```bash
"/c/Program Files/OpenSSH/ssh.exe" -i "$(cygpath -w ~/.ssh/example_ed25519)" -o IdentitiesOnly=yes -o IdentityAgent=none -T git@github.com
```

```bash
GIT_SSH_COMMAND='ssh -i ~/.ssh/example_ed25519 -o IdentitiesOnly=yes -o IdentityAgent=none' git push origin main
```

---

# Notes

## `ssh-agent bash` vs `eval $(ssh-agent -s)`

- `eval $(ssh-agent -s)` starts a **new background agent** with a random socket and puts its address into the current shell only. Other shells cannot see it. It **keeps running with the key in memory after the shell closes**, until you run `ssh-agent -k`.
- `ssh-agent bash` starts a shell that owns the agent. When the shell exits, the agent exits too.
- `export SSH_AUTH_SOCK=...` (flow 1 on Arch) starts nothing. It points the shell at the shared agent that is already running.

## `ssh-add` options

| Option | Meaning |
|---|---|
| `-l` | list loaded keys (fingerprints) |
| `-L` | list loaded public keys in full |
| `-d <key>` | remove one key |
| `-D` | remove all keys (does **not** delete the key file) |
| `-t <time>` | expire the key after a while (Linux) |
| `-c` | ask to confirm every use (Linux, needs `ssh-askpass`) |

## `ssh` options used in flow 3

| Option | Meaning |
|---|---|
| `-i <key>` | use this key file |
| `-o IdentitiesOnly=yes` | try only that key |
| `-o IdentityAgent=none` | do not use any agent, so nothing is kept |
| `-T` | no terminal, just test the login |
| `GIT_SSH_COMMAND=...` | the ssh command for this one git command only |
