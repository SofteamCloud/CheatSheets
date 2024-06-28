# Git Tooling
## Set up your git Prevents you from committing passwords
To Prevents you from committing passwords and other sensitive information to a git repository.
acting before something bad happen 😉  
`brew install git-secrets`

Add a configuration template if you want to add hooks to all repositories you initialize or clone in the future.  
`git secrets --register-aws --global`   
Add hooks to all your local repositories.  
`git secrets --install ~/.git-templates/git-secrets`   
`git config --global init.templateDir ~/.git-templates/git-secrets`

### Install it on existing repo
To add the hook in existing repo   
Go to you're prefer git folder

```sh
for d in $(ls -d ./* | sort -r) ; do
  echo " install in folder $d";
  cd $d ;
   git secrets --install;
  cd -
done
```

### check 
Check  history of a repo 
`git secrets --scan-history` 

You’re done ✅ 

## Enhance your git usage with lazygit
A simple terminal UI for git commands, written in Go
[see more here](https://github.com/jesseduffield/lazygit)

## Auto-Stash on Pull
Enjoy advantages like auto-stashing:

`git config rebase.autoStash true`

This means if you are working on something, you don’t need to stash them before pulling; git will do it for you. 
It will stash the changes, do a pull (rebase), and then apply the stashed changes into the result.

