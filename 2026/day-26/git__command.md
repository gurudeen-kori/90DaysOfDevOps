1. git init ( make the git repository of this folder)
2. git status [ check the status ]
3. git add <filename> [add file in git staged]
4. git commit -m "message"              [for tracking file or tracked stage ]
5. git log      [history of commits ]
6. git rm <filename>
7. git restore          [<restore deleted file >]
8. git branch   [ show branches ]
9. git branch -d        [ delete branche ]
10. git checkout -b     [create new branches with copy all file of current branch]
11. git checkout        [switch existing branch]
12. git switch  [for swiche branche]
git config --global user.name "gurudeen_kori"
git config --global user.email "gurudeenkori2003@gmail.com"                                    

git config --list  # View configuration

git init    # Initialize a repository

git status  # Check repository status

git add index.html        # Stage changes

git add index.html            # Stage all changes

git commit -m "initial commite "  # Commit staged changes

git diff              # Changes not staged

git diff --staged     # Changes staged for commit

git branch            # List branches

git branch <name>     # Create a new branch

git checkout <name>   # Switch to branch

git switch <name>     # Alternative to checkout

git checkout -b <name>  # Create and switch in one command

git clone <repo-url>  # Copy repository locally

git fork <repo>       # Fork repository on GitHub (web)

git push origin master   # Push changes to remote

git pull origin dev   # Pull & merge remote changes

git fetch origin            # Fetch changes without merging

git reset --soft 7e2c8e5    # Undo commit, keep changes staged

git reset --mixed 7e2c8e5   # Undo commit, unstage changes

git reset --hard 7e2c8e5  # Undo commit, discard changes

gh commands 

gh auth login
gh auth logout
gh auth status
gh repo create
gh repo clone <owner>/<repo>  Clone Repository

View Repository
gh repo view <owner>/<repo>

List Repositories
gh repo list <username>

Fork Repository
gh repo fork <owner>/<repo>

Delete Repository
gh repo delete <owner>/<repo>

Issue Commands
Create Issue
gh issue create

List Issues
gh issue list
View Issue

gh issue view <number>

Close Issue
gh issue close <number>

Reopen Issue
gh issue reopen <number>

Pull Request Commands
Create Pull Request

gh pr create
List Pull Requests

gh pr list
View Pull Request
gh pr view <number>

Checkout Pull Request
gh pr checkout <number>

Merge Pull Request
gh pr merge <number>

Close Pull Request
gh pr close <number>

GitHub Actions Commands

List Workflow Runs
gh run list
