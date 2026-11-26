# Git Safety: Local vs Remote, and Practicing in the Right Place

Two safety notes after practicing merge, rebase, and reset in a real repo.

---

## 1. Two places exist

| Place | Name | Where it is |
|---|---|---|
| **Local** | `main`, `feature` | On your computer |
| **Remote** | `origin/main` | On GitHub |

`origin/main` is only a **note on your computer** that says: "The last time I synced with GitHub, `main` was at this commit."

It is not live. It updates only when you run `git fetch` or `git pull`.

### Reading a real log

```
* f77b6f4 (HEAD -> main) main: green
* 7f73cf6 edit config
* a1bd516 (origin/main, origin/HEAD) add config
```

| Commit | Status |
|---|---|
| `a1bd516 add config` | On your computer **and** on GitHub |
| `7f73cf6 edit config` | **Local only** |
| `f77b6f4 main: green` | **Local only** |

So local `main` is **2 commits ahead** of `origin/main`. `git status` says the same:

```
Your branch is ahead of 'origin/main' by 2 commits.
```

---

## 2. Note 1: You practiced in your real repo

The lab said to create a new, throwaway folder (`~/conflict-lab`). Practicing happened inside the real DevOps repo instead. That repo has study notes, test files, and an `origin` on GitHub.

### Why does it matter?

| Reason | Explanation |
|---|---|
| The repo is your **portfolio** | Everything you push is visible on GitHub |
| Practice commits mix with real work | Commits like `edit config`, `main: green`, and `feature:blue` mean nothing to someone viewing your portfolio |
| Risk of mistakes | If you ran `reset` or `rebase` on a pushed commit, it could break shared history |

### Why was nothing broken this time?

`reset --hard` and `rebase` only touched **local, unpushed commits** (`7f73cf6`, `f77b6f4`, the merge commit, and the feature commits).

- Nobody else had copies of those commits
- GitHub was never touched
- Rewriting local history is safe

**Golden rule:** Rewriting history (`reset`, `rebase`) is only safe on commits you have **not pushed yet**.

---

## 3. Note 2: `main` is ahead of `origin/main`

### What is the danger?

If you run:

```bash
git push
```

the 2 practice commits (`edit config` and `main: green`) go to GitHub. Because you only moved forward (fast-forward), GitHub accepts them without a warning. After that, they are part of public history. Removing them needs a `revert` or a force push.

### What about the `feature` branch?

| Branch | Status |
|---|---|
| `feature` | Local only, not on GitHub |
| `git push` (no branch name) | Pushes only the current branch |
| `git push origin feature` | Sends `feature` to GitHub as well |

Do not push `feature` unless you want to. It is technically safe (it is a new branch, no force needed), but it only adds clutter to a portfolio repo.

---

## 4. How to check the state

| Command | What it shows |
|---|---|
| `git status` | "ahead of origin/main by 2 commits" |
| `git log origin/main..main --oneline` | **Only** the unpushed commits (the 2 commits) |
| `git branch -a` | Local and remote branches |

---

## 5. How to clean up (if you do not want the practice commits)

First, make sure the 2 commits are only practice and contain no real work:

```bash
git log origin/main..main --oneline
git show 7f73cf6
git show f77b6f4
```

Then:

| Goal | Command |
|---|---|
| Return `main` to the state of GitHub | `git switch main`, then `git reset --hard origin/main` |
| Delete the practice branch | `git branch -D feature` |

| Note | Explanation |
|---|---|
| `reset --hard origin/main` | **Throws away** the 2 local commits and their changes. Your files become the same as GitHub. If you want to keep anything, first run `git branch backup` |
| `branch -D` | Deletes the branch without checking whether it was merged. Fine for practice |

If you want to keep the commits, but not on `main`:

```bash
git branch lab-experiments      # keeps the current commits on a separate branch
git switch main
git reset --hard origin/main    # cleans main
```

---

## 6. Always practice in a throwaway repo

```bash
mkdir ~/git-lab && cd ~/git-lab
git init -b main
```

| Benefit | Explanation |
|---|---|
| No remote | `git push` cannot send anything anywhere |
| Safe to break | You can delete the whole folder when done |
| Your portfolio stays clean | Only real work reaches GitHub |

---

## 7. Quick summary

| Question | Answer |
|---|---|
| What is `origin/main`? | A local note of where `main` was on GitHub |
| What does "ahead" mean? | You have commits that are not pushed yet |
| Why was reset/rebase safe? | It only touched unpushed commits |
| Where is the danger? | One accidental `git push` sends practice commits to your portfolio |
| What should I do next time? | Practice in a separate folder like `~/git-lab` |

### Rules to remember

1. **Before `git push`**, run `git status` and `git log origin/main..main --oneline`. Check what you are about to send.
2. **Rewrite history** (`reset`, `rebase`) **only** on unpushed commits.
3. **Practice** in a throwaway repo, never in your portfolio repo.
4. **Before `reset --hard`**, create a backup branch if you are unsure: `git branch backup`.

### Cheat sheet

| Command | Meaning |
|---|---|
| `git status` | Shows if you are ahead or behind the remote |
| `git log origin/main..main --oneline` | Lists commits that are not pushed yet |
| `git branch -a` | Lists local and remote branches |
| `git branch backup` | Saves the current commit on a new branch (safety net) |
| `git reset --hard origin/main` | Makes local `main` identical to GitHub (discards local commits) |
| `git branch -D feature` | Force-deletes the local branch `feature` |
| `git fetch` | Updates `origin/main` without changing your files |