# Git Merge, Fast-Forward, 3-Way Merge, --no-ff, and Rebase

Simple notes with examples, scenarios, and the questions I had while learning.

---

## 1. What is `git merge`?

**Merge = combine two branches into one.**

Think of a Word file. You make a copy to try some changes. When the changes are good, you bring them back into the original file. That is a merge.

### How it works

```
main:     A --- B --------- M     <- after merge
                \          /
feature:         C --- D --
```

| Step | What you do |
|---|---|
| 1 | Create a new branch from `main` (called `feature`) |
| 2 | Work on it and make commits (C and D) |
| 3 | Go back to `main` and bring the changes of `feature` into it |

### Command

```bash
git checkout main        # go to the branch that should RECEIVE the changes
git merge feature        # bring feature into it
```

**Rule:** Always stand on the **target** branch (for example `main`) and type the name of the **source** branch.

---

## 2. A branch is just a label

A branch is **a pointer (label) to one commit**. When you make a new commit, the label moves to the new commit.

This idea makes everything below easier.

---

## 3. Two types of merge

| Type | When it happens | Result |
|---|---|---|
| **Fast-forward** | `main` did not change since you created your branch | The label just moves forward. No new commit. |
| **3-way merge** | Both branches have new commits | A new **merge commit** is created. |

### Scenario 1: Fast-forward

You created `feature` and worked on it. Nobody touched `main` during that time.

```
Before merge:

main:     A --- B
                 \
feature:          C --- D
```

`main` is at B. `feature` started from B and is 2 commits ahead. `main` has nothing new that `feature` does not have.

So the path from `main` to `feature` is **one straight line**. Git does not need to combine anything. It only moves the `main` label to D:

```
After merge:

main, feature:   A --- B --- C --- D
```

| Question | Answer |
|---|---|
| Was a new commit created? | No |
| What does the history look like? | One straight line |
| Why was it easy? | `main` had no changes that could clash with yours |

**Real-life example:** You copied a project, worked on the copy, and the original never changed. Now you put the copy in place of the original.

### Scenario 2: 3-way merge

You created `feature` and worked on it. But your teammate **also added commits to `main`** in the same time.

```
Before merge:

main:     A --- B --- E --- F
                 \
feature:          C --- D
```

Now two lines grew from B:
- `main` went to F (your teammate's work)
- `feature` went to D (your work)

Git cannot just move a label. If it moved `main` to D, the work in E and F would be lost. So Git must **combine both works**.

It is called "3-way" because Git looks at three points:

| Point | Role |
|---|---|
| **B** | The common ancestor (where the branches split) |
| **F** | The latest state of `main` |
| **D** | The latest state of `feature` |

Git checks what changed from B to F, what changed from B to D, and applies both. The result is a new commit called the **merge commit** (M). It has two parents:

```
After merge:

main:     A --- B --- E --- F ---- M
                 \              /
feature:          C ---- D ----
```

| Question | Answer |
|---|---|
| Was a new commit created? | Yes (M) |
| What does the history look like? | Two lines that meet at M |
| Can there be a conflict? | Yes, if both changed the same line |

### Real example with files

Your project has `login.py` and `payment.py`.

| Case | What happened on `main`? | What did you do? | Merge type |
|---|---|---|---|
| 1 | Nothing | Changed `login.py` | Fast-forward |
| 2 | Teammate changed `payment.py` | Changed `login.py` | 3-way (no conflict, different files) |
| 3 | Teammate changed line 10 of `login.py` | Also changed line 10 of `login.py` | 3-way **with conflict** |

---

## 4. What is a conflict?

A conflict happens when **both branches changed the same line of the same file**. Git does not know which one is correct.

```
<<<<<<< HEAD
version from main
=======
version from feature
>>>>>>> feature
```

### How to fix it

| Step | Action |
|---|---|
| 1 | Open the file and decide what should stay |
| 2 | Delete the markers `<<<<`, `====`, `>>>>` |
| 3 | `git add <file>` |
| 4 | `git commit` |

### Useful commands

| Command | What it does |
|---|---|
| `git merge --abort` | Cancel the merge and go back to before it |
| `git log --graph --oneline` | See the shape of branches and merges |

---

## 5. Practice: Fast-forward and 3-way (2 minutes)

```bash
mkdir test && cd test && git init
echo "v1" > a.txt && git add . && git commit -m "A"

# --- Fast-forward ---
git checkout -b feature
echo "v2" >> a.txt && git commit -am "C"
git checkout main
git merge feature            # it says: Fast-forward
git log --oneline --graph    # straight line
```

```bash
# --- 3-way ---
git checkout -b feature2
echo "x" > b.txt && git add . && git commit -m "D"
git checkout main
echo "y" > c.txt && git add . && git commit -m "F"   # main changed too
git merge feature2           # editor opens for the merge commit message
git log --oneline --graph    # two lines that meet
```

---

## 6. `git merge --no-ff`

`--no-ff` means: **"Do not fast-forward. Always create a merge commit."**

**Scenario:** `main` did not change, so a normal merge would fast-forward.

```
Before:      main: A --- B
                          \
             feature:      C --- D
```

| Command | Result |
|---|---|
| `git merge feature` | Straight line: `A - B - C - D`. You can no longer see that C and D were on a feature branch. |
| `git merge --no-ff feature` | Creates an extra commit (M). The branch stays visible in history. |

```
After --no-ff:

main:     A --- B -------- M
                 \        /
feature:          C --- D
```

### Why do teams use it?

- History shows "these 2 commits belong to one feature".
- Teams often want this for Pull Requests.
- It makes **undoing a whole feature** easy (see next section).

---

## 7. What does "revert M" mean?

> Correction: earlier I wrote "the whole feature comes back". That was confusing. The right meaning is: **the whole feature is removed from `main`.**

**Scenario:** You merged `feature` into `main`, and then found out it has a bug.

### With `--no-ff` (you have a merge commit M)

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

R is a **new commit** that undoes the changes of C and D on `main`. With one command, the whole feature leaves `main`.

| Question | Answer |
|---|---|
| Is the feature code removed from `main`? | Yes |
| Is the `feature` branch deleted? | No. It is still there. You can fix it and merge again. |
| Is history deleted? | No. R is only a new commit that does the opposite. |

### Without `--no-ff` (fast-forward)

```
main:  A --- B --- C --- D
```

There is no M. You must revert C and D one by one (`git revert D C`). If the feature had 20 commits, you need 20 reverts. **This is where `--no-ff` helps.**

---

## 8. `git rebase`

**Same goal as merge:** bring the changes of one branch into another.
**Different method:** instead of combining, Git **takes your commits and replays them again, after the latest commit of the other branch.**

**Scenario:** Same as the 3-way case. `main` moved forward (E and F).

```
Before:

main:     A --- B --- E --- F
                 \
feature:          C --- D
```

### With merge

```bash
git checkout feature
git merge main
```

```
feature:  A --- B --- E --- F
                 \           \
                  C --- D --- M
```

One extra commit (M). History has two lines.

### With rebase

```bash
git checkout feature
git rebase main
```

```
main:     A --- B --- E --- F
                              \
feature:                       C' --- D'
```

Git takes C and D and puts **new copies (C' and D')** after F. It is as if you had started your work from F.

After that, merging into `main` is a straight line, so it is a **fast-forward**:

```bash
git checkout main
git merge feature     # fast-forward
```

```
main:  A --- B --- E --- F --- C' --- D'
```

### Merge vs Rebase

| | `merge` | `rebase` |
|---|---|---|
| History | Branches are visible | One straight line |
| New commit | Creates a merge commit | None, but commits are **copied** (hash changes) |
| Changes history? | No, safe | Yes |
| Readability | Busier | Cleaner |
| Conflicts | Solve once | You may solve for every commit |

---

## 9. What do C' and D' mean?

A commit is not only "changes". Every commit has an **address (hash)** made from two things: **its own changes + its parent commit**.

| | Parent | Hash |
|---|---|---|
| C (before rebase) | B | `abc123` |
| C' (after rebase) | F | `xyz789` |

The changes (code) are **the same**, but the parent is different. So Git creates a **new commit with a new hash**. The `'` sign only means "a copy of C with a new parent".

Your code is not lost. It is just applied again on top of a new base.

---

## 10. Golden rule of rebase

> **Never rebase commits that you already pushed and other people are using.**

### Why? A scenario

| Step | You | Your teammate |
|---|---|---|
| 1 | Push `feature` (C, D with old hashes) | Runs `git pull`. Now has old C and D and works on them (makes commit G). |
| 2 | Rebase, which creates C' and D' (new hashes) | Still has old C and D |
| 3 | `git push --force` (history changed) | Old C and D are deleted on the server! |
| 4 | | When they pull, Git sees C' and D' and also their own C and D. It tries to merge both: **duplicate commits and fake conflicts**. |

The problem: Git thinks C and C' are two different changes, even though they are the same.

### Which one should I use?

| Situation | Use |
|---|---|
| My own branch, not pushed yet | `rebase` |
| Update my feature with the latest `main` before a PR | `rebase` |
| Shared branch or `main` | `merge` |
| I want to keep branch history visible | `merge --no-ff` |

---

## 11. Conflicts during rebase

It is like merge, with one big difference: it goes **commit by commit**.

**Scenario:** You have 2 commits (C and D), and both conflict with `main`.

| Step | What Git says | What you do |
|---|---|---|
| `git rebase main` | "Conflict while applying C. I stopped." | Fix the file |
| | | `git add file` |
| | | `git rebase --continue` |
| | "Now applying D. Conflict again." | Fix, `add`, `--continue` |
| | Done | |

If you get lost: `git rebase --abort` and everything goes back to how it was before the rebase.

| | merge | rebase |
|---|---|---|
| Conflicts | Solve **once** | May solve **for each commit** |

So if you have 20 commits and many conflicts, merge is easier.

---

## 12. Why do we rebase at all?

One main reason: **a clean, straight history.**

After weeks of work, with many "bring main into feature" merges:

```
Very busy:
*   Merge main into feature
|\
| * fix typo
* |  Merge main into feature
|\|
| * add login
```

With rebase:

```
Clean:
* add login
* fix typo
* ...
```

| Reason | Explanation |
|---|---|
| Readability | `git log` reads like a story |
| No useless merge commits | No "Merge main into feature" ten times |
| Finding bugs | `git bisect` is easier on a straight line |

**If rebase feels hard, it is fine to use merge.** Merge is safer and fully enough while learning.

---

## 13. Practice: Rebase (2 minutes)

```bash
mkdir t2 && cd t2 && git init
echo "1" > a.txt && git add . && git commit -m "A"

git checkout -b feature
echo "f" > f.txt && git add . && git commit -m "C"

git checkout main
echo "m" > m.txt && git add . && git commit -m "E"

git log --oneline --graph --all     # two lines

git checkout feature
git rebase main                      # C becomes C'
git log --oneline --graph --all     # one straight line, C has a new hash
```

To compare, repeat the same practice and use `git merge main` instead of rebase. Look at `--graph` again.

---

## 14. Quick Summary

| Question | Short answer |
|---|---|
| What is merge? | Combine two branches into one |
| What is a branch? | A label pointing to a commit |
| Fast-forward? | `main` is behind, just move the label forward |
| 3-way merge? | Both moved forward, combine them and make a merge commit |
| Conflict? | Both changed the same line, you decide which one stays |
| `--no-ff`? | Always create a merge commit so the branch stays visible |
| What does revert M do? | Removes the feature code from `main` using a new commit |
| What is C'? | Same changes as C, with a new parent and a new hash |
| Why not rebase pushed commits? | Hashes change and it clashes with teammates' work |
| Conflicts in rebase? | Solve commit by commit, then `--continue` |
| Why rebase? | Clean, straight history |

### Cheat sheet

| Command | Meaning |
|---|---|
| `git merge feature` | Merge `feature` into the current branch |
| `git merge --no-ff feature` | Merge and always create a merge commit |
| `git merge --abort` | Cancel a merge |
| `git rebase main` | Replay my commits on top of `main` |
| `git rebase --continue` | Continue after fixing a conflict |
| `git rebase --abort` | Cancel a rebase |
| `git revert -m 1 M` | Undo a merge commit M |
| `git log --graph --oneline --all` | See the branch shapes |

**Advice for now:** Use `merge` (even `--no-ff`) in your projects. Learn `rebase` later when you feel comfortable.