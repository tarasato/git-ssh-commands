# Git Command SSH

Windows = Git Bash · Arch = Arch Linux · no label = same on both

## Install

**Windows**

- Add path `C:\Program Files\Git\`
- Now you can press `Win+R` and type `git-bash`

```powershell
winget install --id GitHub.cli
```

```powershell
winget install --id GLab.GLab
```

**Arch**

```bash
sudo pacman -S --needed git openssh github-cli glab
```

## Git Login Command

```bash
git config --global user.name "yourUsername"
```

```bash
git config --global user.email "youremail@example.com"
```

## Git init

```bash
git init
```

```bash
git add .
```

```bash
git commit -m "first commit"
```

```bash
git branch -M main
```

```bash
git remote add origin https://github.com/examples/examples.git
```

```bash
git push -u origin main
```

**Or:**

```bash
git remote add origin https://github.com/examples/examples.git
```

```bash
git branch -M main
```

```bash
git push -u origin main
```

## Save HTTPS Login

**Windows** — Git Credential Manager is included (`credential.helper = manager`), first `git push` opens the login window. Nothing to set.

**Arch** — no Git Credential Manager, choose one:

GitHub only — let `gh` give git its token (run after `gh auth login`):

```bash
gh auth setup-git
```

Any host (GitHub + GitLab) — save in the desktop keyring (GNOME Keyring / KWallet), needs `libsecret`:

```bash
sudo pacman -S --needed libsecret
```

```bash
git config --global credential.helper /usr/lib/git-core/git-credential-libsecret
```

No keyring (server / TTY) — keep in memory for 1 hour:

```bash
git config --global credential.helper 'cache --timeout=3600'
```

`store` also works but saves the password/token as **plain text** in `~/.git-credentials`:

```bash
git config --global credential.helper store
```

## Git SSH Key

```bash
eval $(ssh-agent -s)
```

```bash
ssh-keygen -t ed25519 -C "youremail@example.com"
```

Enter file in which to save the key:
- **Windows** -> `C:\Git\.ssh\id_rsa`
- **Arch** -> press `Enter` (default `~/.ssh/id_ed25519`)

Set permissions — **Arch** (ssh refuses a private key others can read: `UNPROTECTED PRIVATE KEY FILE`):

```bash
chmod 700 ~/.ssh
```

```bash
chmod 600 ~/.ssh/id_ed25519
```

Add key to agent:

**Windows**

```bash
ssh-add C:/Git/.ssh/id_rsa
```

**Arch**

```bash
ssh-add ~/.ssh/id_ed25519
```

```bash
ssh-add -l
```

To del ssh key from agent **THIS IS NOT DELETE FILE COMMAND**

```bash
ssh-add -D
```

Show / copy key, then paste it in GitHub/GitLab settings → SSH Keys:

**Windows**

```bash
cat C:/git/.ssh/id_rsa
```

**Arch** (Wayland / X11)

```bash
wl-copy < ~/.ssh/id_ed25519.pub
```

```bash
xclip -selection clipboard < ~/.ssh/id_ed25519.pub
```

```bash
ssh -T git@github.com
```

```bash
ssh -T git@gitlab.com
```

**Windows**

```bash
chmod 400 ~/.ssh/id_rsa
```

## Check Login

### HTTPS

Check which credential helper git uses (Windows default = `manager`):

```bash
git config --show-origin --get-all credential.helper
```

**Windows** — list GitHub accounts saved in Git Credential Manager:

```bash
git credential-manager github list
```

**Windows** — list saved git credentials in Windows Credential Manager (`//list` because Git Bash converts `/list` to a path):

```bash
cmdkey //list | grep -i git
```

Check that the saved GitHub token still works (shows your login, does not print the token). If nothing is saved, `git credential fill` asks you to log in instead:

```bash
TOKEN=$(printf "protocol=https\nhost=github.com\n\n" | git credential fill | sed -n 's/^password=//p')
```

```bash
curl -s -H "Authorization: Bearer $TOKEN" https://api.github.com/user | grep '"login"'
```

```bash
unset TOKEN
```

Test access to a **private** repo without any login prompt (works for GitHub and GitLab — a public repo succeeds even when logged out):

```bash
GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never git ls-remote https://gitlab.com/USERNAME/PROJECT.git HEAD
```

### SSH (with ssh-agent)

Start ssh-agent (needed once per terminal — `Could not open a connection to your authentication agent` means it is not running):

```bash
eval $(ssh-agent -s)
```

Add your private key to the agent (change the path to your key):

```bash
ssh-add ~/.ssh/id_ed25519
```

Check keys loaded in the agent:

```bash
ssh-add -l
```

Test GitHub → `Hi USERNAME! You've successfully authenticated...` (exit code 1 is normal, GitHub has no shell). First time it asks to trust the host key → type `yes`:

```bash
ssh -T git@github.com
```

Test GitLab → `Welcome to GitLab, @USERNAME!`:

```bash
ssh -T git@gitlab.com
```

`Permission denied (publickey)` = the host does not know this key → add the `.pub` file in GitHub/GitLab settings → SSH Keys:

```bash
cat ~/.ssh/id_ed25519.pub
```

### GitHub CLI / GitLab CLI

```bash
gh auth status
```

```bash
glab auth status
```

## GitHub CLI (gh) / GitLab CLI (glab)

Install → see **Install** above. Commands below are the same on Windows and Arch.

### Login

```bash
gh auth login
```

```bash
glab auth login
```

```bash
gh auth status
```

```bash
glab auth status
```

```bash
gh auth logout
```

```bash
glab auth logout
```

### Create Repo

New repo on the host only (`--public` for public, glab also `--internal`; glab `-s` = do not touch local git):

```bash
gh repo create PROJECT --private
```

```bash
glab repo create PROJECT --private -s
```

From current local repo → create on host + add remote (gh also pushes; glab uses remote `gitlab` to keep your existing `origin`):

```bash
gh repo create PROJECT --private --source=. --remote=origin --push
```

```bash
glab repo create PROJECT --private --remoteName gitlab
```

### Rename Repo

`glab repo update` has no rename option → use the API (`USERNAME%2FPROJECT` = `USERNAME/PROJECT` URL-encoded; `name` = display name, `path` = URL):

```bash
gh repo rename NEW-PROJECT -R USERNAME/PROJECT --yes
```

```bash
glab api -X PUT projects/USERNAME%2FPROJECT -f name=NEW-PROJECT -f path=NEW-PROJECT
```

Update local remote to the new URL (old URL redirects, but update it anyway):

```bash
git remote set-url origin https://github.com/USERNAME/NEW-PROJECT.git
```

```bash
git remote set-url gitlab https://gitlab.com/USERNAME/NEW-PROJECT.git
```

### Change Visibility

`public` / `private` / `internal` (GitHub internal = organization only; no visibility option in `glab repo update` → use the API):

```bash
gh repo edit USERNAME/PROJECT --visibility public --accept-visibility-change-consequences
```

```bash
glab api -X PUT projects/USERNAME%2FPROJECT -f visibility=public
```

```bash
gh repo view USERNAME/PROJECT --json visibility
```

```bash
glab repo view USERNAME/PROJECT
```

## ETC

### Custom Git Command

To check `.bashrc` file (**Arch** with zsh → use `~/.zshrc` instead):

```bash
ls -a ~
```

```bash
nano ~/.bashrc
```

```bash
alias cls='clear'
```

```bash
source ~/.bashrc
```

### Make Script Executable

So people who clone on Linux/macOS can run `./install.sh` directly (no `Permission denied`). Git stores it as file mode `100755` (executable) instead of `100644` (normal).

**Windows** — `chmod +x` has no effect (Git for Windows uses `core.fileMode=false`), so set the mode in git index:

```bash
git update-index --chmod=+x install.sh
```

**Arch** — git reads the execute bit from the file:

```bash
chmod +x install.sh
```

```bash
git add install.sh
```

Check (`100755` = executable):

```bash
git ls-files -s install.sh
```

Then commit and push as usual.

### Push to GitHub + GitLab at once

```bash
git remote add origin https://github.com/USERNAME/PROJECT.git
```

```bash
git remote set-url --add --push origin https://github.com/USERNAME/PROJECT.git
```

```bash
git remote set-url --add --push origin https://gitlab.com/USERNAME/PROJECT.git
```

```bash
git push origin main
```
