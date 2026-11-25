# Git Basics

## 1. The Three Areas

| Area | What it is | Example |
|---|---|---|
| Working Directory | Files on disk that I edit | `text-main-test.txt` |
| Staging Area (Index) | Changes prepared for the next commit | files after `git add` |
| Repository (`.git`) | Saved history of commits | all commits |

```
Working Directory --git add--> Staging Area --git commit--> Repository
```

## 2. Tracked vs Untracked

| Type | Meaning |
|---|---|
| Tracked | Git knows the file (it was committed before) |
| Untracked | New file, Git does not know it yet |

## 3. `git add` vs `git commit`

| Command | What it does |
|---|---|
| `git add <file>` | Moves the file to staging (choose what goes in the next commit) |
| `git commit -m "msg"` | Saves everything in staging permanently as a snapshot |

## 4. `git restore`

Throws away uncommitted changes and returns a file to the last committed version.

```bash
git restore <file>            # discard changes in working directory (cannot be undone)
git restore --staged <file>   # unstage a file (keeps the changes)
```

- It needs a file path (`git restore` alone gives an error).
- It only works on tracked files. An untracked file has no saved version to go back to.

## 5. `git stash`

A temporary drawer for uncommitted changes.

| Command | What it does |
|---|---|
| `git stash` | Puts all tracked changes in the drawer, working directory becomes clean |
| `git stash list` | Shows what is in the drawer |
| `git stash pop` | Takes the changes out and applies them again |
| `git stash -u` | Also stashes untracked files (`-u` = untracked) |

| Command | Result |
|---|---|
| `git restore` | Changes are deleted forever |
| `git stash` | Changes are saved temporarily and can come back |

Typical flow: edit → `git stash` → `git switch feature` → `git switch main` → `git stash pop`.

## 6. Git Stores Snapshots, Not Diffs

| | Diff | Snapshot |
|---|---|---|
| What is stored | Only the changes | The full state of the project |
| Reaching version 50 | Apply 50 diffs in order | Open that snapshot directly |

Git does not copy unchanged files again. It points to the old copy, so it stays small.

## 7. Git Objects

| Object | What it is |
|---|---|
| blob | The content of one file (no file name) |
| tree | A folder: list of file names pointing to blobs |
| commit | Points to one tree + parent + author + message |

```
HEAD → branch → commit → tree → blob
```

- A commit **points to** a tree. It is not the tree itself.
- The branch is just a name that points to a commit.
- `HEAD` points to the current branch (where I am now).

## 8. Reading a Commit

```bash
git cat-file -p HEAD
```

```
tree      <snapshot of the files>
parent    <hash of the previous COMMIT>
author    <who wrote the change>
committer <who created the commit>

<commit message>
```

- `parent` is the hash of the previous commit, not its tree.
- The message comes after an empty line.
- To see hashes of commits: `git log --oneline`
- To see files and blob hashes: `git ls-tree HEAD`

## 9. If I Change One File and Commit

| Object | New? |
|---|---|
| Blob of the changed file | 1 new |
| Blobs of other files | Same as before |
| Tree | 1 new |
| Commit | 1 new |

## 10. Hash

A hash is a fingerprint of the content.

- Same content → same hash.
- Any tiny change → completely different hash.

```bash
echo "hello"  | git hash-object --stdin
echo "hello." | git hash-object --stdin
```

## 11. `git commit --amend`

Fixes the last commit (message or forgotten file).

```bash
git commit --amend -m "new message"
```

- It creates a **new commit** with a **new hash**, because the content changed.
- Never amend a commit that is already pushed.

## 12. `git switch`

- Moves `HEAD` to another branch.
- Git then updates my files to match the tree of that branch's last commit.
- A branch is cheap: it is only a small file with one commit hash.

```bash
cat .git/refs/heads/main
```

## 13. Messages I Saw

| Message | Meaning |
|---|---|
| `fatal: you must specify path(s) to restore` | I forgot the file name |
| `Your branch is ahead of 'origin/main' by 4 commits` | I have 4 local commits not on GitHub yet. Use `git push` |
| `Changes not staged for commit` | File changed, not added yet |
| `Changes to be committed` | File is in staging |
| `working tree clean` | Nothing to commit |

## 14. Commands Cheat Sheet

| Command | Purpose |
|---|---|
| `git status` | See where each file is |
| `git add <file>` | Stage a file |
| `git commit -m "msg"` | Save a snapshot |
| `git restore <file>` | Discard uncommitted changes |
| `git restore --staged <file>` | Unstage a file |
| `git stash` / `git stash pop` | Save changes temporarily / bring them back |
| `git switch <branch>` | Go to another branch |
| `git log --oneline --graph --all` | See history |
| `git cat-file -p HEAD` | Look inside the current commit |
| `git ls-tree HEAD` | List files and blob hashes |
| `git commit --amend` | Fix the last commit |