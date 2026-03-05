##  Scenario 1 – Wrong Branch Commit

Situation:
1. You are supposed to work on feature/login.
2. By mistake, you commit directly to main.
3. The commit has not been pushed yet
 Solution
 cd Cloud-Maven-Repo
 git status
 git add .
 git commit -m "update"
 git pull origin main --rebase
 git push origin main

## Scenario 2
Situation
 1. Developer A pushed code to main
 2. You pulled the changes
 3. You made a wrong commit (bad config change)
 4. You pushed it to remote
 5. Team says the change is wrong
 Solution
  git log --oneline
  git revert a12b45c
  git push origin main
  git reset --hard HEAD~1  

 ##  Scenario 3   
 Situation
  You are working on branch feature/payment
  Developer A changed the same file in main
  You try to merge main into your branch
  A merge conflict occurs

  Solution
  git checkout feature/payment
  git merge main
  git status
  git diff
# resolve conflict manually
  git add payment.js
  git commit
  git push origin feature/payment
