# DevOps Study Notes: Git and GitHub (Week 3)

Date: 2026-11-24

How to use this file: read it once (about 40 minutes), then do the exercises in the week 3 plan. Each section ends with what I must be able to do without notes. Commands marked "(check)" differ between versions or were not tested on my machine. Before it goes into my repo, I rewrite the parts I use in my own words.


---

## 1. The Big Picture


**Version control** is a system that records every change of a project, so I can go back, compare, and work with other people without overwriting each other.

**Git** is a **distributed** version control system. "Distributed" means every copy (clone) has the **full history**, not only the latest files. Because of that, most work (commit, branch, log) is local and needs no network.

| Term | What it is |
|---|---|
| Git | The program on my computer. Tracks history. |
| Repository (repo) | A project folder plus its history (stored in the hidden `.git` folder). |
| GitHub | A website that hosts Git repos and adds collaboration (pull requests, issues). It is not Git itself. |
| Remote | Another copy of the repo somewhere else (usually on GitHub). Default name: `origin`. |
| Clone | A full copy of a repo, including history. |

DevOps connection: nearly everything later (CI/CD, Docker, Terraform, Kubernetes manifests) lives in Git. A change reaches production through a commit and a pull request.

**I must be able to say:** the difference between Git and GitHub, and why "distributed" matters.

---

## 2. A Commit Is a Snapshot

Git does **not** store "the changes". A commit stores a **snapshot** of all tracked files at that moment (unchanged files are reused, not copied again).

Git has a simple database of **objects**, each named by a hash (a long hexadecimal number computed from its content):

| Object | What it holds |
|---|---|
| **blob** | The content of one file (no name). |
| **tree** | A folder: a list of names pointing to blobs and other trees. |
| **commit** | Points to one tree (the snapshot), to its **parent** commit(s), plus author, time, message. |

```
commit  ->  tree  ->  blob (file A)
   |               ->  blob (file B)
   v
 parent commit
```

Consequences:

- Same content gives the same hash. If I change one file, only that blob, the trees above it, and the new commit are new. Other files are reused.
- The hash of a commit depends on its content **and its parent**. So if I change anything in the history, all hashes after it change. This is why rewriting history (amend, rebase) creates new commits with new hashes.
- A commit is nearly unchangeable. "Changing" a commit really means making a new one.

Look inside (safe, read-only):

```
git cat-file -p HEAD          # show the commit: tree, parent, author, message
git cat-file -p <tree-hash>   # show the folder content
git cat-file -t <hash>        # show the type: blob, tree or commit
```

**I must be able to do:** explain snapshot vs. diff, and show a commit, its tree and a blob with `cat-file`.

---

## 3. The Three Areas

A file moves through three places. This is the most important picture in Git.

```
Working directory  --git add-->  Staging area (index)  --git commit-->  Repository (.git)
(my files, edited)               (what goes into the next commit)       (history)
```

- **Working directory**: the files I see and edit.
- **Staging area (index)**: a list of exactly what the **next commit** will contain. It lets me build a clean commit from a messy day.
- **Repository**: the committed history.

| Command | What it does |
|---|---|
| `git status` | Shows the state of the three areas. I use it all the time. |
| `git add file` | Working directory to staging area. |
| `git commit -m "msg"` | Staging area to repository. |
| `git diff` | Working directory vs. staging area (unstaged changes). |
| `git diff --staged` | Staging area vs. last commit (what I am about to commit). |
| `git restore file` | Throw away my edit in the working directory (**cannot be undone**: the edit was never saved in Git). |
| `git restore --staged file` | Take the file out of staging, keep my edit. |
| `git commit --amend` | Replace the last commit with a new one (new content and/or message). New hash. Do it only if the commit is not pushed yet. |

Good commit habit: one commit = one logical change, and a message that says **why**, not only what.

**I must be able to do:** predict the output of `status`, `diff` and `diff --staged` before running them, and say which command touches which area.

---

## 4. Branches, HEAD, and the Reflog

A **branch** is only a **movable pointer (a name) to one commit**. A new commit moves the pointer forward. That is why branches are cheap: creating one writes a tiny file, it does not copy the project.

**HEAD** is the pointer to "where I am now". Normally HEAD points to a branch, and the branch points to a commit.

```
HEAD -> main -> C3 -> C2 -> C1
```

| Command | What it does |
|---|---|
| `git branch` | List branches. |
| `git switch -c name` | Create a branch and move to it. |
| `git switch name` | Move to a branch. |
| `git log --oneline --graph --all` | Draw the history. Use it to *see* what happens. |

### Detached HEAD

If HEAD points directly to a **commit** (not to a branch), it is "detached" (for example `git switch --detach HEAD~1`). I can look around and even commit, but those new commits belong to no branch. If I switch away, nothing points to them and they look lost.

### Reflog: the safety net

`git reflog` is a local log of where HEAD has been. Commits that look lost are usually still there for some weeks (until Git cleans up).

```
git reflog
git branch rescue HEAD@{1}     # give a name to a commit I would lose
```

The reflog is **local only**: it is not pushed and not shared.

**I must be able to do:** explain "a branch is a pointer", create a detached-HEAD commit on purpose, and rescue it with the reflog.

---

## 5. Merge

Merge combines the work of another branch into the branch I am on (`git merge other`).

| Case | When | Result |
|---|---|---|
| **Fast-forward** | The target branch has **no new commits** since I branched. | Git only moves the pointer forward. No new commit. History stays a straight line. |
| **Merge commit** | Both branches have **new commits**. | Git makes a new commit with **two parents**. |
| `--no-ff` | Force a merge commit even if fast-forward is possible. | Keeps a visible "this branch existed" bubble in the history. |

Merge **never changes existing commits**. It only adds. This makes it safe for shared branches.

**I must be able to do:** predict before running whether a merge will be fast-forward, and show it in `git log --graph`.

---

## 6. Merge Conflicts

A conflict happens when Git **cannot decide** automatically, because the two sides changed the **same part** of a file (or one side changed a file and the other deleted it). Git cannot know my intention, so it asks me.

Three kinds to practice:

1. The same line changed in two branches.
2. One branch modified a file, the other deleted it.
3. A conflict that appears **during a rebase** (see section 8).

What a conflicted file looks like:

```
<<<<<<< HEAD
my side (current branch)
=======
their side (the branch I merge in)
>>>>>>> other-branch
```

Steps:

1. `git status` shows "both modified" files.
2. Edit the file: keep the right result, **delete the three marker lines**.
3. `git add file`, then `git commit` (merge) or `git rebase --continue` (rebase).
4. If I am lost: `git merge --abort` or `git rebase --abort` returns me to the state before.

Conflicts are normal, not an error. Small branches and frequent merging make them rare and small.

**I must be able to do:** create each kind of conflict on purpose, resolve it, and abort it once.

---

## 7. Remotes, GitHub, and SSH Keys

A **remote** is another copy of the repo. `origin` is the usual name for the one I cloned from.

| Command | What it does |
|---|---|
| `git clone url` | Copy a remote repo, with history. |
| `git remote -v` | Show remotes. |
| `git fetch` | Download new commits from the remote. **Does not change my files or branches.** |
| `git pull` | `fetch` + merge into my branch (by default). Can be set to rebase: `git pull --rebase`. |
| `git push` | Send my commits to the remote. |

### Push rejected

If someone else pushed first, my push is refused ("rejected ... fetch first"), because my history has no knowledge of their commit and Git will not overwrite theirs. Fix: bring their commits in first (`git pull --rebase` or `git pull`), resolve conflicts if any, then push again.

Never "fix" it with `git push --force` on a shared branch: it overwrites other people's work. (A safer form is `--force-with-lease`, but the rule stays: not on shared branches.)

### Authentication to GitHub

GitHub accepts HTTPS (token) or SSH. I use **SSH** with a **separate key for GitHub** (not the key of my lab VM):

```
ssh-keygen -t ed25519 -C "github-niludev"
# add the PUBLIC key (.pub) in GitHub: Settings -> SSH and GPG keys   (check: menu names)
ssh -T git@github.com
```

Same rule as in Linux: the **private key never leaves my computer**, only the `.pub` file goes to GitHub. One key per purpose means that if one leaks I replace only that one.

**I must be able to do:** explain fetch vs. pull, cause a rejected push on purpose and fix it, and explain why the GitHub key is separate.

---

## 8. Merge vs. Rebase

Both combine work from two branches. They differ in the **history they leave**.

| | Merge | Rebase |
|---|---|---|
| What it does | Adds a merge commit joining both histories | **Replays** my commits on top of the other branch's tip |
| History | True history, with branches visible | Straight line, looks as if I worked after the others |
| Existing commits | Unchanged | **Rewritten: new hashes** |
| Safe on shared branches | Yes | **No** |

**Golden rule of rebase:** never rebase commits that other people may already have (pushed, shared). Rebase only my own local or private-branch commits. Reason: rewritten commits have new hashes, so other people's history no longer matches.

### Interactive rebase

`git rebase -i HEAD~3` lets me edit the last 3 commits before I share them: `squash` (join commits into one), `reword` (change a message), `drop` (remove), reorder. Use it to clean a messy branch before a pull request.

**I must be able to do:** draw the history of the same scenario with merge and with rebase, and explain why the hashes change.

---

## 9. Pull Requests and GitHub Merge Options

A **pull request (PR)** is a request on GitHub: "please review my branch and merge it into `main`". It gives a place for **review and discussion** before the code lands. It is a GitHub feature, not a Git command.

Working flow from now on (also alone, to build the habit):

1. Create a branch. 2. Commit. 3. Push the branch. 4. Open a PR. 5. Review it myself (read the diff). 6. Merge. 7. Delete the branch.

Nothing goes straight to `main`.

GitHub's three ways to merge a PR:

| Option | Result on `main` |
|---|---|
| **Merge commit** | All commits stay, plus one merge commit. |
| **Squash and merge** | All PR commits become **one** commit. Clean history, details lost. |
| **Rebase and merge** | The PR commits are replayed one by one, no merge commit (new hashes). |

Which one a team uses is a team decision. What matters is knowing what each does to the history.

**I must be able to do:** open and merge one real PR, and explain the three options.

---

## 10. Undoing Things (Disaster Drills)

First question always: **has it been pushed?** Unpushed: I may rewrite. Pushed: do not rewrite, add a new commit that undoes it.

### reset: moves the branch pointer (rewrites history)

| Mode | Branch pointer | Staging area | Working files |
|---|---|---|---|
| `--soft` | moves | kept | kept |
| `--mixed` (default) | moves | reset | kept |
| `--hard` | moves | reset | **reset (my edits are lost)** |

`git reset --hard HEAD~1` removes the last commit and my uncommitted work. The commit itself is often still recoverable through `git reflog`. Uncommitted work is **not** recoverable.

### revert: adds a new commit that cancels an old one

`git revert <commit>` does not rewrite history, so it is **safe for pushed commits**.

To revert a **merge commit**, Git needs to know which parent to keep: `git revert -m 1 <merge-commit>`. `-m 1` means "parent number 1", which is the branch I merged **into** (for example `main`).

| | reset | revert |
|---|---|---|
| History | Rewritten (commits disappear) | Kept, a new commit is added |
| Use for | Local, unpushed mistakes | Pushed commits |

### Other tools

- **Stash**: `git stash` puts my uncommitted work aside, `git stash pop` brings it back. For "I must switch branch now but I am not ready to commit".
- **Wrong branch**: I committed on `main` by mistake. Create a branch where I am (`git branch new-work`), then move `main` back (`git reset --hard HEAD~1`). The commit lives on `new-work`.
- **Lost commit**: `git reflog`, find it, `git branch rescue <hash>`.

**I must be able to do:** run these five disasters (lost commit, wrong branch, revert of a merge, the three resets, stash) and explain reset vs. revert.

---

## 11. Security: Secrets and Git

Git history is **permanent and copied**. If I commit a password or key and then delete it in the next commit, it is **still in the history** (`git log -p` shows it), and anyone with a clone has it. After a push, assume it is public.

Rule for a real leak, in this order:

1. **Rotate (revoke and replace) the secret first.** The old one must be treated as stolen.
2. Only then clean the history if needed (special tools; this rewrites history for everyone).
3. Cleaning history alone is **not** a fix, because copies and caches may exist.

Prevention:

- **`.gitignore`**: a file listing what Git must never track: private keys (`*.pem`, `id_*`), `.env` files, temp files. It only affects files **not yet tracked**; a file already committed stays tracked.
- **Secret scanner**: tools like `gitleaks` search the files and history for passwords and keys. Command differs by version: `gitleaks detect` (older) or `gitleaks git` (newer) (check with `gitleaks --help`).
- Use only **FAKE** secrets in practice repos.
- Config belongs in environment variables or a secret manager, not in the repo.

**I must be able to do:** show with `git log -p` that a deleted secret is still in the history, and say why "rotate first, then clean".

---

## 12. Key Points to Remember

- Git is distributed: every clone has the full history. GitHub is a website around Git.
- A commit is a snapshot (tree of blobs), not a diff. Its hash depends on content and parent.
- Three areas: working directory, staging area, repository. `status` and `diff` show where I am.
- A branch is a pointer. HEAD is where I am. Branches are cheap.
- `reflog` is the local safety net for "lost" commits.
- Fast-forward moves a pointer. A merge commit has two parents. Merge never rewrites history.
- A conflict means Git cannot guess my intention. Remove the markers, `add`, continue. `--abort` is always possible.
- `fetch` downloads. `pull` is fetch plus merge (or rebase). A rejected push means someone pushed first.
- Rebase rewrites commits (new hashes). Never rebase shared commits.
- A PR is for review before merge. GitHub has three merge options.
- Pushed mistake: `revert`. Local mistake: `reset`. `reset --hard` loses uncommitted work.
- A committed secret stays in the history. Rotate first, then clean. Use `.gitignore` and a scanner.

---

## 13. Interview Answers (short, in my own words later)

**What is the difference between merge and rebase?**
Merge joins two branches with a new merge commit and keeps the real history. Rebase replays my commits on top of the other branch, which gives a straight history but creates new commits with new hashes, so I never rebase shared commits.

**What is a branch and what is HEAD?**
A branch is a pointer to a commit that moves forward with each new commit. HEAD points to where I am now, usually to a branch.

**What is the difference between `reset` and `revert`?**
`reset` moves the branch pointer and rewrites history, so it is for local commits. `revert` adds a new commit that undoes an old one, so it is safe for pushed commits.

**I deleted a commit with `reset --hard`. Can I get it back?**
Usually yes, from the reflog: find the hash with `git reflog` and create a branch on it. Uncommitted changes are lost for good.

**A password was committed and pushed. What do you do?**
First rotate the secret, because it must be treated as leaked. Then, if needed, clean the history and add prevention (`.gitignore`, a scanner). Deleting the file in a new commit is not enough, because the history still contains it.

**My push was rejected. Why and what do I do?**
Someone pushed commits I do not have. I bring them in with `git pull --rebase` (or a merge), resolve conflicts, and push again. I do not force-push on a shared branch.

---

## 14. Still To Study (not in week 3)

- Branching strategies (trunk-based, GitFlow) and tags/releases (SemVer, week 4)
- Branch protection rules and required reviews on GitHub
- CI on pull requests (GitHub Actions, later phases)
- `git bisect`, `cherry-pick`, submodules
- Signing commits (GPG/SSH)
- Cleaning secrets from history with dedicated tools