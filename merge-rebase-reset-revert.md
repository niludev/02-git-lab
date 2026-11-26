# Git: merge, rebase, reset, revert

A complete guide with examples, scenarios, and comparisons.

---

## 0. A simple analogy

Imagine you are writing a book:

| Command | Analogy |
|---|---|
| `merge` | You staple two separate drafts together and attach a note explaining it |
| `rebase` | You take your pages and rewrite them as if you wrote them after the latest edit by others |
| `reset` | You **tear out** the last few pages |
| `revert` | You write a new page that says "ignore the change on page 5". Page 5 stays in the book |

---

## 1. Basic ideas you need first

| Idea | Meaning |
|---|---|
| **Commit** | A saved snapshot of your project |
| **Hash** | The commit's ID, like a fingerprint (for example `a1b2c3d`) |
| **Branch** | A label (pointer) that points to one commit |
| **HEAD** | "Where I am now" (usually the latest commit of the current branch) |
| **`HEAD~1`** | One commit before HEAD |
| **Parent** | The commit that came right before a commit |

### What is a hash made of?

A hash is made from **everything inside the commit**:

- the snapshot of the files
- the **parent** commit
- the author
- the message
- the time

If any of these changes, the hash changes. This is the key to understanding rebase.

You can see it yourself:

```bash
git cat-file -p <hash>
```

The output has a line `parent ...`. After a rebase, that line is different.

---

## 2. Shared starting point (for merge and rebase)

```
main:     A --- B --- E --- F
                 \
feature:          C --- D
```

Both branches started from B. After that, `main` got E and F, and `feature` got C and D.

---

## 3. `git merge`

### What it does

Combines two branches into one. It **does not touch any existing commit**.

```bash
git checkout main
git merge feature
```

Result:

```
main:     A --- B --- E --- F ---- M
                 \              /
feature:          C ---- D ----
```

M is a new **merge commit**. It has two parents (F and D).

### Facts

| Question | Answer |
|---|---|
| Do hashes change? | No. A, B, C, D, E, F keep the same hash |
| What is new? | One merge commit (M) with two parents |
| Exception | With fast-forward, no new commit is created |
| Does it rewrite history? | No, it only adds |
| Conflicts? | Solve once |
| Safe for pushed branches? | Yes |

### When to use it

- A feature is finished and you want it in `main`
- You work on a shared branch
- You want to keep the fact that "these commits were one feature" (`--no-ff`)

### Fast-forward vs 3-way

| Type | When | Result |
|---|---|---|
| **Fast-forward** | `main` did not change since you created your branch | The label just moves forward. No new commit |
| **3-way merge** | Both branches have new commits | A new merge commit is created |

Fast-forward example:

```
Before:   main: A --- B
                       \
          feature:      C --- D

After:    main, feature: A --- B --- C --- D
```

It is called "3-way" because Git compares three points: the common ancestor (B), the tip of `main` (F), and the tip of `feature` (D).

### `--no-ff`

Forces a merge commit even when fast-forward is possible. The branch stays visible in history.

```bash
git merge --no-ff feature
```

```
main:     A --- B -------- M
                 \        /
feature:          C --- D
```

### Useful merge commands

| Command | Meaning |
|---|---|
| `git merge feature` | Merge `feature` into the current branch |
| `git merge --no-ff feature` | Always create a merge commit |
| `git merge --abort` | Cancel a merge in progress |

---

## 4. `git rebase`

### What it does

Takes your commits and **rebuilds them after the latest commit of the target branch**.

```bash
git checkout feature
git rebase main
```

Result:

```
main:     A --- B --- E --- F
                              \
feature:                       C' --- D'
```

C' and D' are **new copies** of C and D. They have the same changes, but a new parent, so they have new hashes.

If you merge `feature` into `main` after this, it is a **fast-forward** because the line is straight:

```
main:  A --- B --- E --- F --- C' --- D'
```

### Facts

| Question | Answer |
|---|---|
| Do hashes change? | **Yes**, for the rebased commits (C and D become C' and D'). A, B, E, F stay the same |
| Why? | Their parent changed |
| What is new? | Copies of the commits, no merge commit |
| Does it rewrite history? | Yes |
| Conflicts? | Commit by commit, maybe several times |
| Safe for pushed branches? | **No** |

### When to use it

- You want to update your personal branch with the latest `main` (and you have not pushed it)
- You want a clean, straight history

### Conflicts in rebase

It goes **commit by commit**:

| Step | What happens | What you do |
|---|---|---|
| `git rebase main` | Conflict while applying C. Git stops | Fix the file |
| | | `git add file` |
| | | `git rebase --continue` |
| | Conflict while applying D | Fix, `add`, `--continue` |
| | Done | |

If you get lost: `git rebase --abort` returns everything to before the rebase.

### Useful rebase commands

| Command | Meaning |
|---|---|
| `git rebase main` | Replay my commits on top of `main` |
| `git rebase --continue` | Continue after fixing a conflict |
| `git rebase --abort` | Cancel the rebase |

### Duplicate commits are skipped

If a commit's change already exists in the target branch (same diff), rebase skips it:

```
warning: skipped previously applied commit 89559f3
```

Git says: "this work was already done, I will not repeat it". Merge does not do this.

---

## 5. `git reset`

### What it does

Moves the branch label **back to an older commit**. The commits after it are cut off from the branch.

```
Before:    A --- B --- C   <- main
```

```bash
git reset --hard HEAD~1
```

```
After:     A --- B         <- main     (C is no longer on any branch)
```

### Facts

| Question | Answer |
|---|---|
| Do hashes change? | No. A and B are the same. C also keeps its hash, but no branch points to it |
| What is new? | Nothing |
| Does it rewrite history? | Yes, commits leave the branch |
| Can I undo it? | If committed: yes, with `git reflog`. If not committed: no |
| Safe for pushed branches? | **No** |

### The three modes

Say you go from C back to B:

| Command | Commit C | Staging area | Files in your working directory |
|---|---|---|---|
| `reset --soft HEAD~1` | Removed | **Stays** (ready to commit) | Stay |
| `reset --mixed HEAD~1` (default) | Removed | Cleared | **Stay** |
| `reset --hard HEAD~1` | Removed | Cleared | **Deleted** |

Real example: you just committed, but the message was wrong, or you forgot a file.

| You want | Use |
|---|---|
| Remove the commit, keep the code, commit again | `--soft` |
| Remove the commit and staging, keep the code | `--mixed` |
| Throw everything away | `--hard` |

### When to use it

- A **local** commit that you have not pushed was a mistake
- You want to cancel a bad merge or rebase

### How to recover from a mistake

```bash
git reflog
# a1b2c3d HEAD@{0}: reset: moving to HEAD~1
# e4f5g6h HEAD@{1}: commit: add login     <- the lost commit

git reset --hard e4f5g6h     # go back to it
```

| Situation | Recoverable? |
|---|---|
| It was committed, then `reset --hard` | Yes, with `reflog` |
| It was not committed (only changed files) | No, gone forever |

**Always run `git status` before `--hard`.**

---

## 6. `git revert`

### What it does

Creates a **new commit** that cancels the effect of an old commit. The old commit stays untouched.

```
Before:    A --- B --- C
```

```bash
git revert C
```

```
After:     A --- B --- C --- R
```

R does exactly the opposite of C. If C added a line, R removes that line.

### Facts

| Question | Answer |
|---|---|
| Do hashes change? | No. C keeps its hash |
| What is new? | One new commit (R) |
| Does it rewrite history? | No, it only adds |
| Conflicts? | Possible (if the code changed after C) |
| Safe for pushed branches? | **Yes** |

### Reverting a merge commit

```bash
git revert -m 1 M
```

| Part | Meaning |
|---|---|
| `M` | The hash of the merge commit |
| `-m 1` | A merge commit has two parents. `-m 1` means: "Use parent 1 (the main line, `main`) as the base, and cancel the changes that came from parent 2 (`feature`)" |

Example:

```
main:     A --- B -------- M
                 \        /
feature:          C --- D
```

```bash
git revert -m 1 M
```

```
main:     A --- B -------- M --- R
```

R removes the changes of C and D from `main`.

| Question | Answer |
|---|---|
| Is the feature code removed from `main`? | Yes |
| Is the `feature` branch deleted? | No. You can fix it and merge again |
| Is history deleted? | No. R is only a new commit that does the opposite |

This is where `--no-ff` helps. Without it (fast-forward), there is no M, and you must revert each commit one by one (`git revert D C`).

### When to use it

- A commit on a shared or pushed branch has a bug
- A whole merged feature is broken

---

## 7. Full comparison

| | `merge` | `rebase` | `reset` | `revert` |
|---|---|---|---|---|
| What it does | Combines two branches | Rebuilds commits on a new base | Moves the label back | Creates an opposite commit |
| Hash of old commits | Same | **Changed** (new copies) | Same | Same |
| New commit | Merge commit (none with fast-forward) | Copies of commits | None | One opposite commit |
| Rewrites history? | No | Yes | Yes | No |
| Graph shape | Two lines that meet | One straight line | Commits leave the branch | Straight line with one extra commit |
| Conflicts | Once | Commit by commit | None | Possible |
| On pushed commits | Safe | Not safe | Not safe | Safe |
| How to cancel it | `merge --abort` | `rebase --abort` | `reflog` | Revert the revert |

---

## 8. Which one should I use?

| Situation | Use |
|---|---|
| Feature is done, I want it in `main` | `merge` (or `--no-ff`) |
| Update my personal branch with the latest `main` (not pushed) | `rebase` |
| A local commit was a mistake | `reset` |
| Cancel a bad merge or rebase | `reset --hard` (or `--abort` while it is running) |
| A pushed commit has a bug | `revert` |
| A whole merged feature is broken | `revert -m 1` |
| Shared branch or `main` | `merge` and `revert`, never `rebase` and `reset` |

---

## 9. Golden rule: do not rewrite pushed history

> **Never rebase or reset commits that you already pushed and others may use.**

### Why? A scenario with rebase

| Step | You | Your teammate |
|---|---|---|
| 1 | Push `feature` (C, D with old hashes) | Runs `git pull`. Now has old C and D and works on them (makes commit G) |
| 2 | Rebase, which creates C' and D' (new hashes) | Still has old C and D |
| 3 | `git push --force` (history changed) | Old C and D are deleted on the server |
| 4 | | When they pull, Git sees C' and D' and also their own C and D. It tries to merge both: **duplicate commits and fake conflicts** |

The problem: Git thinks C and C' are two different changes, even though they are the same work.

The same problem happens with `reset` on a pushed commit. Use `revert` instead.

---

## 10. Two key questions

### Q1. What is the difference between the graph of merge and rebase?

```
merge:                          rebase:
*   Merge branch 'feature'      * feature: red
|\                              * main: green
| * feature: red                * add config
* | main: green
|/
* add config
```

| | merge | rebase |
|---|---|---|
| Lines | **Two lines** that meet at the merge commit | **One** straight line |
| Meaning | "This work was done in parallel" | "It looks like the work was done one after another" |

### Q2. Why did the hash of the commit `feature: red` change after rebase?

A hash is like a **fingerprint**. It is made from the whole content of the commit: the file snapshot, the parent, the author, the message and the time. If one of these changes, the fingerprint changes.

| | Parent | Hash |
|---|---|---|
| `feature: red` before rebase | `add config` | for example `a1b2c3d` |
| `feature: red` after rebase | `main: green` | a **new** hash |

The parent changed, so the hash changed. Rebase does **not modify** the old commit (commits never change). It creates a **new commit** and moves the `feature` label to it.

Also, if you fixed a conflict during the rebase, the content of the commit changed too. That changes the hash as well.

### A nice detail to remember

| Command | Hashes |
|---|---|
| `reset` | The old commit still exists with the **same hash**. If you bring it back with `reflog`, it is exactly the same |
| `rebase` | The old commit still exists, but **a new version with a different hash** is also created |

---

## 11. Understanding a real graph

How to read `git log --oneline --graph --all`:

```
*   2b174e0 (HEAD -> main) Merge branch 'feature'
|\
| * 89559f3 (feature) feature:blue
| * 9b382de Add test file for feature branch
* | f77b6f4 main: green
* | 7f73cf6 edit config
|/
* 8635e6e Add a blank line commit 3
```

| Symbol | Meaning |
|---|---|
| `\|\` | One place where **two lines split** (two parents) |
| Left column | Work on `main` |
| Right column | Work on `feature` |
| `\|/` | Lower down, the two lines **meet again** (the point where they split) |
| `*` with two lines above it | A merge commit |

After `git rebase main` on `feature`:

```
* 5e30290 (HEAD -> feature) Add test file for feature branch
* f77b6f4 (main) main: green
* 7f73cf6 edit config
```

| What happened | Why |
|---|---|
| One straight line | Rebase rebuilt the `feature` commits after `main` |
| `9b382de` became `5e30290` | Its parent changed, so its hash changed |
| `89559f3` disappeared | Its change already existed in `main`, so Git skipped it (`skipped previously applied commit`) |

---

## 12. Practice: all four commands (10 minutes)

Use a **new, throwaway folder**. Do not practice in your real project repo.

```bash
mkdir ~/git-lab && cd ~/git-lab
git init -b main
echo "color: blue" > config.txt
git add . && git commit -m "add config"
```

### Merge with conflict

```bash
git switch -c feature
echo "color: red" > config.txt
git commit -am "feature: red"
git switch main
echo "color: green" > config.txt
git commit -am "main: green"

git merge feature                 # CONFLICT
cat config.txt                    # see the markers
# edit the file, keep one final line, remove <<<<, ====, >>>>
git add config.txt
git commit --no-edit
git log --oneline --graph --all   # two lines that meet
```

### Reset (cancel the merge)

```bash
git reset --hard HEAD~1
git log --oneline --all           # write down the hash of "feature: red"
```

### Rebase with conflict

```bash
git switch feature
git rebase main                   # CONFLICT
git status                        # "rebase in progress"
# fix the file
git add config.txt
git rebase --continue
git log --oneline --graph --all   # one line, new hash for "feature: red"
```

### Revert

```bash
git switch main
git merge feature                 # fast-forward
git log --oneline
git revert HEAD                   # creates a new commit that cancels the last one
git log --oneline                 # old commit is still there, plus a revert commit
```

### Questions to answer in your own words

1. What is the difference between the graph shape of merge and rebase?
2. Why did the hash of `feature: red` change after rebase?
3. Why does `reset` keep the same hashes, but `rebase` does not?
4. Why is `revert` safer than `reset` on a pushed branch?

---

## 13. Quick summary

| Command | One sentence |
|---|---|
| `merge` | Join two histories with one extra commit. Nothing old changes |
| `rebase` | Take my commits and rebuild them on top of the latest work. Hashes change |
| `reset` | Move the branch label back. Commits are cut off but keep their hash |
| `revert` | Add a new commit that cancels an old one. Nothing old changes |

### Three rules to remember

1. **Already pushed?** Use only `merge` and `revert`.
2. **Local and a mistake?** `reset` or `rebase` is fine.
3. **Before `reset --hard`**, always run `git status`. Uncommitted changes cannot be recovered.

### Cheat sheet

| Command | Meaning |
|---|---|
| `git merge feature` | Merge `feature` into the current branch |
| `git merge --no-ff feature` | Merge and always create a merge commit |
| `git merge --abort` | Cancel a merge |
| `git rebase main` | Replay my commits on top of `main` |
| `git rebase --continue` | Continue after fixing a conflict |
| `git rebase --abort` | Cancel a rebase |
| `git reset --soft HEAD~1` | Remove last commit, keep staging and files |
| `git reset --mixed HEAD~1` | Remove last commit and staging, keep files |
| `git reset --hard HEAD~1` | Remove last commit and all changes |
| `git reflog` | See where HEAD has been (to recover lost commits) |
| `git revert <hash>` | Create a commit that cancels `<hash>` |
| `git revert -m 1 <merge-hash>` | Cancel a merge commit |
| `git log --oneline --graph --all` | See the shape of all branches |
| `git cat-file -p <hash>` | See what is inside a commit (including its parent) |