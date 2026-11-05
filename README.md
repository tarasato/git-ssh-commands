# Git Login Command
<br>
add path C:\Program Files\Git\
<br>
now you can win+r and type -> `git-bash`
<br>
git config --global user.name "yourUsername"
<br>
git config --global user.email "youremail@example.com"
<br>

# Git init
git init
<br>
git add .
<br>
git commit -m "first commit"
<br>
git branch -M main
<br>
gh repo create exmanples --private
<br>
gh repo rename new-exmanples-name
<br>
gh repo edit --visibility private
<br>
git remote add origin https://github.com/examples/examples.git
<br>
git remote set-url --add --push origin https://github.com/examples/examples.git
<br>
git remote set-url --add --push origin git@gitlab.com:examples/examples.git
<br>
git push -u origin main
<br>
<br>
git remote add origin https://github.com/examples/examples.git
<br>
git branch -M main
<br>
git push -u origin main

# Git SSH key (Git Bash)
<br>
eval $(ssh-agent -s)
<br>
ssh-keygen -t ed25519 -C "youremail@example.com"
<br>
ssh-add E:/Git/.ssh/id_rsa
<br>
to del ssh key from agent THIS IS NOT DELETE FILE COMMAND
<br>
ssh-add -D
<br>
test connecting to github
<br>
ssh -T git@github.com
<br>
gh auth login

# ETC
<br>
# Custom Git Command
<br>
to check .bashrc file
<br>
ls -a ~
<br>
nano ~/.bashrc
<br>
alias cls='clear'
<br>
source ~/.bashrc
<br>

## To back to specific version
# 1. Grab the commit hash of specific Commit
`git log --oneline -n`

# 2. Overwrite all your project files with how they looked at specific Commit
`git restore --source <commit-specific-hash>`

# 3. Commit and push normally (no force-push needed!)
`git commit -m "reset code back to specific commit hash state"`
<br>
`git push origin <branch-name>`
<br>
## For GitLab
please use this command first before first commit <br>
git pull origin main --allow-unrelated-histories --no-rebase
