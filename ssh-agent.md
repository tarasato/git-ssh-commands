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

## Additional

### Show loaded keys: `-l` vs `-L`

Fingerprint (short ID of each key):

```bash
ssh-add -l
```

```
256 SHA256:Xa1bC2dE3fG4hI5jK6lM7nO8pQ9rS0tU1vW2xY3zA4b user@example.com (ED25519)
```

- `256`: key size in bits (always 256 for ed25519)
- `SHA256:...`: fingerprint = SHA-256 hash of the public key. GitHub / GitLab show the same format under Settings -> SSH Keys
- `user@example.com`: the key's **comment** (set with `-C` when the key was made), not the file name
- `(ED25519)`: key type

Full public keys (same line as in the `.pub` file):

```bash
ssh-add -L
```

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... user@example.com
```

Use `-L` when you need to paste the public key somewhere (GitHub, GitLab, a server's `authorized_keys`) and the `.pub` file is not at hand. Neither option ever shows the private key.

Same output from the files:

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

```bash
cat ~/.ssh/example_ed25519.pub
```

### Which file is a loaded key?

The agent keeps the key and its comment only, not the file name. Match fingerprints instead.

Fingerprint of every `.pub` file:

```bash
for f in ~/.ssh/*.pub; do echo "$(ssh-keygen -lf "$f" | cut -d' ' -f2)  $f"; done
```

Only the files whose key is loaded in the agent:

```bash
for f in ~/.ssh/*.pub; do fp=$(ssh-keygen -lf "$f" | cut -d' ' -f2); ssh-add -l | grep -q "$fp" && echo "loaded: $f"; done
```

- `for f in ~/.ssh/*.pub; do ...; done`: run once per `.pub` file, name in `$f`
- `ssh-keygen -lf "$f"`: fingerprint of that file (`-l` fingerprint, `-f` file)
- `cut -d' ' -f2`: keep the 2nd space-separated word (`SHA256:...`)
- `grep -q "$fp"`: true if that fingerprint is in the agent's list (`-q` = print nothing)
- `&& echo`: print only when it matched

### Change a key's comment

The comment is the label at the end of the public key (`... example_ed25519` or `... user@example.com`). `ssh-add -l` shows it instead of the file name. It is only a label: changing it does not change the key or its fingerprint, and every place that already trusts the key keeps working.

1. See the current comment

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

2. Change it (asks for the passphrase if the key has one)

```bash
ssh-keygen -c -C "example_ed25519" -f ~/.ssh/example_ed25519
```

- `-c`: change the comment
- `-C "..."`: the new comment
- `-f`: the **private** key file. The `.pub` next to it is rewritten too.

3. Check the result (the last word must be the new comment)

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

4. If the key is loaded in the agent, remove and add it again, or `ssh-add -l` keeps the old comment

```bash
ssh-add -d ~/.ssh/example_ed25519
```

```bash
ssh-add ~/.ssh/example_ed25519
```

5. Optional: GitHub / GitLab / a server's `authorized_keys` keep showing the old comment. Login still works. To show the new one, delete the key there and add the new `.pub` again.

Set the comment when creating a new key instead:

```bash
ssh-keygen -t ed25519 -C "example_ed25519" -f ~/.ssh/example_ed25519
```

The comment travels with every public key you hand out, so a file name is better than a real e-mail address.

### File permissions

```bash
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/example_ed25519
```

| Number | Owner | Group | Others | Result |
|---|---|---|---|---|
| `700` | `rwx` | `---` | `---` | only you can open / list / change the folder |
| `600` | `rw-` | `---` | `---` | only you can read / edit the file |

Each digit = read `4` + write `2` + execute `1` (for a folder, `x` = may enter it). Check with `ls -la ~/.ssh`. ssh refuses a private key that others can read (`UNPROTECTED PRIVATE KEY FILE!`) and a `config` that others can write (`Bad owner or permissions`). `.pub` files may stay `644`.


### Copy folder from remote
```bash
scp -i ~/.ssh/id_ed25519 -r user@0.0.0.0:~/path/to/folder ~/path/to/destination
```
Or
```bash
mkdir -p /path/to/destination && ssh -i ~/.ssh/id_ed25519 user@0.0.0.0 "tar czf - -C /path/to/folder" | tar xzf - -C /path/to/destination
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

## Additional

### Show loaded keys: `-l` vs `-L`

Same as Arch (see Part 1 -> Additional for what each field means), with the Windows `ssh-add.exe`:

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -l
```

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -L
```

PowerShell:

```powershell
ssh-add -l
```

```powershell
ssh-add -L
```

### Which file is a loaded key?

Git Bash (`ssh-keygen` from Git Bash is fine here, it only reads the files):

```bash
for f in ~/.ssh/*.pub; do echo "$(ssh-keygen -lf "$f" | cut -d' ' -f2)  $f"; done
```

Only the loaded ones (asks the Windows agent):

```bash
for f in ~/.ssh/*.pub; do fp=$(ssh-keygen -lf "$f" | cut -d' ' -f2); "/c/Program Files/OpenSSH/ssh-add.exe" -l | grep -q "$fp" && echo "loaded: $f"; done
```

PowerShell:

```powershell
Get-ChildItem $HOME\.ssh\*.pub | ForEach-Object { "{0}  {1}" -f ((ssh-keygen -lf $_.FullName) -split ' ')[1], $_.Name }
```

### Change a key's comment

Same steps as Arch (see Part 1 -> Additional). `ssh-keygen` from Git Bash is fine for the file; only the agent steps need the Windows `ssh-add.exe`.

1. See the current comment

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

2. Change it

```bash
ssh-keygen -c -C "example_ed25519" -f ~/.ssh/example_ed25519
```

3. Check the result

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

4. If the key is loaded in the Windows agent, remove and add it again so the agent stores the new comment

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" -d "$(cygpath -w ~/.ssh/example_ed25519)"
```

```bash
"/c/Program Files/OpenSSH/ssh-add.exe" "$(cygpath -w ~/.ssh/example_ed25519)"
```

The same in **PowerShell**:

```powershell
ssh-keygen -c -C "example_ed25519" -f $HOME\.ssh\example_ed25519
```

```powershell
ssh-add -d $HOME\.ssh\example_ed25519
```

```powershell
ssh-add $HOME\.ssh\example_ed25519
```

5. Optional: GitHub / GitLab / `administrators_authorized_keys` keep the old comment until you replace the public key there.

### File permissions

`chmod` in Git Bash has almost no effect: Windows uses ACLs. Keys under `C:\Users\<you>\.ssh` are normally readable only by you, SYSTEM and Administrators. To lock a key down to your account only (PowerShell):

```powershell
icacls $HOME\.ssh\example_ed25519 /inheritance:r /grant:r "${env:USERNAME}:F"
```

```powershell
icacls $HOME\.ssh\example_ed25519
```

- `/inheritance:r`: drop permissions inherited from the folder
- `/grant:r "<user>:F"`: give your account Full control, replacing what it had

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
