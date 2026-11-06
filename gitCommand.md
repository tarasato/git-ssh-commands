# Git Command SSH

- Add path `C:\Program Files\Git\`
- Now you can press `Win+R` and type `git-bash`

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

## Git SSH Key (Git Bash)

```bash
eval $(ssh-agent -s)
```

```bash
ssh-keygen -t ed25519 -C "youremail@example.com"
```

Enter file in which to save the key -> `C:\Git\.ssh\id_rsa`

```bash
ssh-add C:/Git/.ssh/id_rsa
```

```bash
ssh-add -l
```

To del ssh key from agent **THIS IS NOT DELETE FILE COMMAND**

```bash
ssh-add -D
```

```bash
cat C:/git/.ssh/id_rsa
```

```bash
ssh -T git@github.com
```

```bash
chmod 400 ~/.ssh/id_rsa
```

## ETC

### Custom Git Command

To check `.bashrc` file:

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
