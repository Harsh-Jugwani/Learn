# 🚀 Git Cheat Sheet (Quick Revision Guide)

A concise and practical Git reference with real-world scenarios and clean structure.

---

## 📌 1. Repository Setup

```bash
git init
```

👉 Initialize a new Git repository.

---

## 🌿 2. Branching

```bash
git switch -c <branch-name>
```
👉 Create + switch to a new branch (recommended)

```bash
git switch <branch-name>
```
👉 Switch to an existing branch

```bash
git checkout -b <branch-name>
```
⚠️ Legacy method (avoid using)

---

## 📂 3. Staging Files

```bash
git add .
```
👉 Add all files to staging

```bash
git add <filename>
```
👉 Add specific file

---

## 🔍 4. Status Check

```bash
git status
```
👉 Shows:
- Current branch
- Modified files
- Staged files

---

## 💾 5. Commit Changes

```bash
git commit -m "your message"
```
👉 Save staged changes with message

---

## 🌐 6. Remote Repository

```bash
git remote add origin <repo-url>
```
👉 Connect local repo to GitHub

```bash
git push origin <branch-name>
```
👉 Push branch to remote

```bash
git pull origin <branch-name>
```
👉 Fetch + merge from remote

---

## 🔀 7. Merge

```bash
git merge <branch-name>
```
👉 Merge branch into current branch

```bash
git merge --abort
```
👉 Cancel merge if conflicts

---

## 🏷️ 8. Tagging

```bash
git tag <tag-name>
```
👉 Tag a commit (release versioning)

```bash
git push origin <tag-name>
```
👉 Push tag to remote

---

## 🔁 9. Reset (Undo Commits)

```bash
git reset --soft HEAD~n
```
👉 Undo commits (keep changes staged)

```bash
git reset --hard HEAD~n
```
⚠️ Deletes commits + changes permanently

```bash
git reset --soft <commit-hash>
```
👉 Go to specific commit

---

## 📜 10. Logs

```bash
git log --oneline
```
👉 Compact commit history

```bash
git log --graph --oneline --all
```
👉 Visual commit tree

---

## 📦 11. Stash (Temporary Save)

### Commands

```bash
git stash
git stash push -m "WIP: Feature X"
git stash list
git stash pop
git stash apply
git stash drop
git stash clear
```

### 🧠 Scenario
You're working on Feature A, but urgent bug comes in.

✔️ Use stash:
```bash
git stash push -m "Feature A progress"
```

✔️ Switch branch → fix bug → come back:
```bash
git stash pop
```

---

## 🔄 12. Rebase vs Merge

### Merge
```
A---B---C (main)
     \
      D---E (feature)
           \
            M (merge commit)
```

### Rebase
```
A---B---C---D'---E' (clean history)
```

```bash
git rebase main
```
👉 Rewrites history → cleaner commits

⚠️ Avoid rebasing shared branches

---

## 🔥 13. Important Missing Commands (Must Know)

### 📥 Clone Repo
```bash
git clone <repo-url>
```

### 🔄 Fetch Updates
```bash
git fetch
```
👉 Get updates without merging

### 🧹 Clean Untracked Files
```bash
git clean -fd
```

### 🧾 Show Differences
```bash
git diff
git diff --staged
```

### 🔍 Check Branches
```bash
git branch
git branch -a
```

### ❌ Delete Branch
```bash
git branch -d <branch-name>
```

### 🔁 Rename Branch
```bash
git branch -m <new-name>
```

### 🔗 Set Upstream Branch
```bash
git push -u origin <branch-name>
```

---

## 🧠 Real-World Workflow

### 🔹 Scenario: Feature Development
```bash
git clone <repo>
git switch -c feature/login
# Work on your feature
git add .
git commit -m "Added login feature"
git push -u origin feature/login
```

### 🔹 Scenario: Sync with Main
```bash
git switch main
git pull origin main
git switch feature/login
git rebase main
```

### 🔹 Scenario: Fix Mistake
```bash
git log --oneline
git reset --soft <commit-hash>
```

---

## ⚡ Pro Tips

✅ Use `switch` instead of `checkout`  
✅ Always pull before push  
✅ Use meaningful commit messages  
⚠️ Avoid `--hard` unless sure  
⚠️ Don't rebase shared branches

---

## 🎯 Quick Summary Table

| Task | Command |
|------|---------|
| Init repo | `git init` |
| Create branch | `git switch -c` |
| Add files | `git add .` |
| Commit | `git commit -m` |
| Push | `git push` |
| Pull | `git pull` |
| Undo | `git reset` |
| Temp save | `git stash` |
| Clean history | `git rebase` |

---

## 🧩 Mental Model

```
Working Directory → Staging Area → Commit → Remote
```

---

## 🚀 You're Ready!

Use this as a last-minute revision sheet before interviews or daily work.
