## Check if your local repository is behind or ahead the remote on GitHub
```bash
git fetch origin     
# Get latest changes from remote, without merging
git status           
# Check if your local branch is behind, ahead, or up-to-date
```
- if you see:
    ```bash
    Your branch is behind 'origin/main' by X commits 
    ```
    -> your local repo is behind
    - View the missing commits:
    ```bash
    git log HEAD..origin/main --oneline
    ```
    - Sync your local repo:
    ```bash
    git pull origin main
    ```
- if you see:
    ```bash
    Your branch is ahead of 'origin/main' by X commits.
    ```
    -> your local repo is ahead
    - Push your changes
    ```bash
    git push origin main
    ```

## How to pull if you have uncommited changes
### Option 1: Temporarily keep your local changes (stash)
```bash
git stash
git pull origin main
git stash pop
```
- This hides your local changes temporarily
- After pulling, your changes come back with git stash pop.
### Option 2:  Discard your changes and reset your working directory
1. Discard changes in tracked files (modified files)
```bash
git restore .
```
2. Delete untracked files or folders (new files not added to Git):
```bash
git clean -f -d
```
- f → force deletion (required)
- d → delete directories as well
3. Now you can safely pull:
```bash
git pull origin main
```