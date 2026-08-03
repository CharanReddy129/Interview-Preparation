# Git & Version Control

## 1. Why Version Control Exists

Before Git, teams either emailed files back and forth or relied on centralized servers that everyone had to be connected to just to save work. Neither approach scales once more than one person touches the same codebase, and neither gives you a reliable way to answer "what changed, when, and why?"

### What is a Version Control System (VCS)?

A **Version Control System (VCS)** is software that tracks and manages changes to files and source code over time. It creates a detailed history of your project, allowing you to see exactly what changed, who made the edit, and why — while letting multiple people work in parallel without overwriting each other's work.

- It records every change as a distinct, retrievable point in history, rather than just keeping one "current" version of a file.
- It ties every change to an author and a message, so history is not just data but a story of *why* the project evolved the way it did.
- It's the foundation that makes team collaboration on the same codebase possible without constant file-overwrite conflicts.

**Why this matters for DevOps specifically:** Git isn't just "how developers save code" — it's the backbone of the entire delivery pipeline. CI/CD pipelines trigger off Git events, GitOps tools (Argo CD, Flux) treat a Git repo as the literal source of truth for what's running in production, and infrastructure-as-code (Terraform, Ansible, Kubernetes manifests) is reviewed and merged through Git before it ever touches real infrastructure.

---

## 2. Types of Version Control Systems

### Local VCS
A Local Version Control System keeps all history in a database stored only on the single machine you're working on.
- It works by recording changes as patches (differences between file versions) in a local database.
- There is no way to collaborate with anyone else — the history exists only where it was created.
- If that machine's disk fails or is lost, the entire project history is gone permanently.
- Example: RCS (Revision Control System).

![Local VCS Image](https://git-scm.com/book/ms/v2/images/local.png)

### Centralized VCS (CVCS)
A Centralized Version Control System stores the entire project history on one single central server, and every developer's machine only holds a working copy of the current files.
- Everyone connects to this one server to check out the latest version or save (commit) new changes.
- This gives one clear "source of truth" that everyone can see, which was a big improvement over local VCS for team visibility.
- The critical weakness: if the central server goes down, nobody can commit, view history, or collaborate at all — it's a single point of failure for the whole team.
- Examples: SVN (Subversion), CVS, Perforce.

![Centralized VCS image](https://git-scm.com/book/ms/v2/images/centralized.png)

### Distributed VCS (DVCS)
A Distributed Version Control System gives every single developer's machine a **complete copy of the entire project history**, not just the current snapshot of files.
- There's no requirement for a central server for day-to-day operations — committing, branching, viewing history, and comparing versions all happen locally and instantly.
- If the remote hosting server (e.g., GitHub) goes down, every developer still has the full history on their own machine and can keep working uninterrupted.
- Multiple people can also sync changes directly with each other, not only through one central hosting service.
- Examples: **Git**, Mercurial.

![Distributed VCS Image](https://git-scm.com/book/ms/v2/images/distributed.png)

| Type | Where history lives | Example tools | Key limitation |
| --- | --- | --- | --- |
| Local VCS | Only on your own machine | RCS | No collaboration; one disk failure = total data loss |
| Centralized VCS | One central server | SVN, CVS, Perforce | Server down = nobody can commit or view history |
| Distributed VCS | Full copy on every clone + server | Git, Mercurial | None of the above — this is why Git is the industry default today |

---

## 3. Git vs GitHub/GitLab/Bitbucket

**Git** is the version control tool itself. It runs entirely on your own machine and has no built-in concept of "the cloud" — you can create a repository, make commits, view history, and create branches with zero internet connection.

**GitHub, GitLab, and Bitbucket** are hosting platforms. They store Git repositories remotely on their servers and add a layer of collaboration features on top of plain Git:
- Pull Requests / Merge Requests, for proposing and reviewing changes before merging.
- Issue tracking, for managing bugs and feature requests.
- CI/CD runners, for automatically building/testing/deploying code on certain Git events.
- Access control, for managing who can read, write, or administer a repository.

**Why this distinction matters in interviews:** You can use Git fully offline with zero GitHub account — GitHub only becomes relevant the moment you want a shared remote copy of your repository (`git remote add origin ...`) or team workflows like Pull Requests.

---

## 4. Git's Core Architecture — The Four Areas

Every Git operation moves data between four distinct places. Understanding this model is what makes every command make logical sense instead of feeling like memorized magic.

1. **Working Directory:** The Working Directory is the actual folder on your computer's file system where you view and edit your project files.
   - **What it does:** It holds the physical files you open and edit in your code editor.
   - **Status:** Changes made here are "untracked" (brand new files) or "modified" (changed existing files) — not yet saved into Git's history in any way.
   - **Analogy:** Your physical desk where you're actively writing a document by hand.

2. **Staging Area (Index):** The Staging Area (historically called the "Index") is a preview area that holds the exact set of changes you intend to include in your *next* commit.
   - **What it does:** It acts as a preparation zone. You move changes here from the Working Directory using the `git add` command.
   - **Status:** Files here are marked "staged" and are ready to be permanently snapshotted the next time you commit.
   - **Analogy:** A shipping box where you place finished items before sealing it up and sending it off.

3. **Local Repository:** The Local Repository is the hidden `.git` folder inside your project directory that stores the complete history of your project on your local machine.
   - **What it does:** It contains all your saved snapshots (commits), branches, tags, and historical data. You save changes here from the Staging Area using the `git commit` command.
   - **Status:** Changes here are safely and permanently logged into your local history.
   - **Analogy:** A filing cabinet in your personal office where you store sealed boxes of records.

4. **Remote Repository:** The Remote Repository is a copy of your project hosted on a distant server or cloud platform (like GitHub, GitLab, or Bitbucket), accessible over the internet.
   - **What it does:** It allows multiple team members to share code, sync changes, and collaborate. You send your local commits here using `git push`, and download updates using `git pull`.
   - **Status:** Changes here are visible to your entire team and backed up in the cloud.
   - **Analogy:** A central company warehouse where everyone sends their finished work to share with the whole team.

```bash
Working Directory --(git add)--> Staging Area --(git commit)--> Local Repo --(git push)--> Remote Repo
       ^                                                                          |
       |------------------------(git pull / git checkout)-------------------------|
```

**Why the staging area exists:** It gives you fine-grained control over what goes into a commit. If you've edited five files but only want two of those changes saved right now, `git add` lets you choose exactly which changes are staged, instead of forcing an all-or-nothing commit of every modified file.

---

## 5. How Git Actually Stores Data (Internals)

This is the section that separates "I memorized the commands" from "I actually understand Git" — and interviewers at any half-serious company will probe here.

Git stores everything as one of four **object types** inside the `.git/objects/` folder, each identified by a **SHA-1 hash** computed from its content.

- **Blob:** Stores the raw content of a single file — just the bytes of data, with no filename or metadata attached to it directly.
- **Tree:** Represents a directory. It maps filenames to the blobs (files) or other trees (subfolders) that live inside that directory — essentially a snapshot of a folder's structure.
- **Commit:** Points to exactly one tree object (representing the full project snapshot at that moment), along with its parent commit(s), the author, a timestamp, and the commit message.
- **Tag:** A named, permanent pointer to one specific commit — typically used to mark release versions.

```
Commit ---> Tree ---> Blob (file1.txt)
                  ---> Blob (file2.txt)
                  ---> Tree (subfolder) ---> Blob (file3.txt)
```

**Critical interview point:** Git does **not** store diffs between commits — it stores a full snapshot of the entire project at every commit (with delta compression applied later during packing, purely for storage efficiency behind the scenes). Diffs (as shown by `git diff` or `git show`) are *computed on the fly for display*, not what's actually saved on disk. This single fact answers a large number of "how does Git work under the hood" interview questions.

### Branches, internally
A branch is not a copy of any files. It is simply a small text file, stored in `.git/refs/heads/`, containing a single commit's SHA hash.
- This is exactly why creating and switching branches is instant in Git — no files are copied anywhere, only a small pointer is created or moved.
- Older, centralized VCS tools often made branching slow and resource-heavy because they worked very differently internally.

### HEAD, internally
`HEAD` is a pointer to whatever you currently have checked out.
- Normally, `HEAD` points to a branch name (e.g., `main`), and that branch name in turn points to a specific commit.
- You can inspect this yourself: `cat .git/HEAD` typically outputs something like `ref: refs/heads/main`.

### Detached HEAD state
If `HEAD` points directly to a commit instead of to a branch, you are in a "detached HEAD" state.
- This typically happens when you run `git checkout <commit-hash>` directly, instead of checking out a branch name.
- Any new commits made while in this state are not attached to any branch.
- If you switch to another branch afterward without first saving your work, those detached commits can become unreachable and eventually get garbage-collected — effectively lost.
- The fix, if you want to keep the work: create a branch at that point *before* switching away, e.g., `git branch new-branch-name`.

---

## 6. Initial Setup & Getting a Repository

### git config
The `git config` command is used to set configuration values that control how Git behaves — most importantly, your identity, which gets attached to every commit you make.

```bash
git config --global user.name "Your Name"          # sets the name recorded as the author on your commits
git config --global user.email "you@example.com"   # sets the email recorded as the author on your commits
git config --list                                  # displays all currently active configuration settings
```
- The `--global` flag applies the setting to every repository on your machine, stored in `~/.gitconfig`. Without it, the setting only applies to the current repository (stored in `.git/config`).
- Setting your identity is not optional in practice — Git will refuse to let you commit without a name and email configured somewhere.

### git init
The `git init` command initializes a brand new, empty Local Repository (or re-initializes an existing one) inside your current project folder.

When you run `git init`, Git creates a hidden directory named `.git` at the root of your folder.
- This hidden folder contains all internal configuration files, objects, and refs that Git needs to track your project.
- It transforms a standard, regular directory into a **Working Directory** tracked by Git.
- It automatically creates your initial default branch (usually named `main` or `master`, depending on your Git version/configuration).

```bash
git init         # turns the current folder into a brand-new, empty Git repository
```

### git clone
The `git clone` command creates a complete copy of an existing remote repository on your local machine, including every commit in its entire history — not just the latest snapshot of files.

- It automatically sets up a remote connection named `origin` pointing back to the source repository, so you can immediately `push`/`pull` without extra configuration.
- It checks out the default branch (usually `main`) into a new folder for you to start working in right away.
- Because it copies full history, cloning a very large, old repository can take significantly longer than a fresh `git init`.

```bash
git clone <url>                 # downloads a full copy of a remote repo, including all commit history
git clone <url> <folder-name>     # clone into a specific folder name instead of the default (the repo's own name)
git clone --depth 1 <url>           # SHALLOW clone — only fetches the latest commit, not full history (much faster; commonly used in CI pipelines that only need the current code)
```

---

## 7. The Everyday Workflow

### git status
The `git status` command shows the current state of your Working Directory and Staging Area — essentially, "what has changed since the last commit, and what's ready to be committed."

- It lists modified files that are not yet staged.
- It lists files that are staged and ready for the next commit.
- It lists untracked files — new files Git has never seen before and isn't tracking yet.
- It is a purely informational command — running it never changes anything in your repository.

```bash
git status            # shows staged, unstaged, and untracked changes
```

### git add
The `git add` command moves changes from the Working Directory into the Staging Area, marking them as ready to be included in the next commit.

- Until a change is staged with `git add`, it exists only in your working files and is not part of what the next commit will save.
- You can stage an entire file, all changed files at once, or even just specific portions ("hunks") of a single file.

```bash
git add <file>                # stage one specific file
git add .                     # stage every changed/new file in the current folder and below
git add -p                    # interactively review and choose which HUNKS (parts) of a file to stage, rather than the whole file at once
```

### git commit
The `git commit` command takes everything currently in the Staging Area and permanently saves it as a new snapshot in the project's history, along with a message describing the change.

- Each commit gets a unique SHA-1 hash identifier and records the author, timestamp, and the full snapshot (via the tree object, as covered in the internals section).
- A commit is the fundamental "save point" of Git — the unit that all history, branching, and merging are built around.

```bash
git commit -m "Add login validation"     # commit staged changes with an inline message
git commit -am "Fix typo"                # stage all TRACKED (already known) files AND commit in one step — skips brand-new untracked files
git commit --amend                       # replace the most recent commit with a new one (edit its message and/or add more staged changes to it)
```
**Critical interview point:** `--amend` rewrites history by generating a brand-new commit SHA in place of the old one. Never amend a commit that has already been pushed and pulled by others — the same rule that applies to rebasing shared history applies here too.

### git log
The `git log` command displays the commit history of the current branch, from most recent to oldest.

```bash
git log                              # full history: SHA, author, date, and full commit message for every commit
git log --oneline                    # compact view — one line per commit, showing shortened SHA + message only
git log --oneline --graph --all      # compact + a visual graph of branches and merges — extremely useful for understanding branch structure, and commonly demoed in interviews
git log -p                           # shows the full line-by-line diff alongside every commit
git log --author="arjun"             # filters history to commits made by a specific author
```

### git diff
The `git diff` command shows the exact line-by-line differences between two states of your files, so you can review precisely what changed before staging or committing.

```bash
git diff                       # differences in the Working Directory that are NOT yet staged
git diff --staged              # differences that are staged, compared against the last commit (also written as --cached, identical meaning)
git diff <branch1> <branch2>   # differences between two entire branches
```

### git show
The `git show` command displays full details of one specific commit, including its metadata and the exact changes it introduced.

```bash
git show <commit-sha>          # shows author, date, message, and the diff introduced by that specific commit
```

---

## 8. Branching

A **branch** is a separate, parallel line of development. Internally, it's a lightweight, movable pointer to a specific commit, which lets you build a new feature, fix a bug, or safely experiment without touching the stable `main` line of the project. If the work doesn't pan out, you simply delete the branch, with zero impact on `main`.

```bash
git branch                      # lists all local branches; the current one is marked with an asterisk
git branch <name>               # creates a new branch pointing at the current commit (does NOT switch to it)
git branch -a                   # lists ALL branches, including remote-tracking branches (e.g., origin/main)
git branch -d <name>            # deletes a branch SAFELY — Git refuses if it has unmerged changes
git branch -D <name>            # FORCE-deletes a branch even if it has unmerged commits (⚠️ can permanently lose work)
git branch -m <old-name> <new-name>     # renames a branch
```

### git checkout / git switch
Both commands are used to change which branch you currently have checked out, updating your Working Directory to match that branch's state.

- **`git checkout`** is the older, multi-purpose command — historically it was also used to restore files, detach HEAD, and more, all under one command name.
- **`git switch`** is a newer, more focused command introduced specifically to handle only branch switching, making its intent clearer and reducing accidental mistakes.
- Both are still widely used in real codebases and tutorials, so it's worth being comfortable with either syntax.

```bash
git checkout <name>              # switch to an existing branch (older syntax)
git checkout -b <name>             # create AND switch to a new branch in a single step (older syntax)
git switch <name>                    # switch to an existing branch (newer syntax)
git switch -c <name>                   # create AND switch to a new branch in a single step (newer syntax)
```

---

## 9. Merging

The `git merge <branch>` command combines the changes from `<branch>` into whichever branch you are currently on, integrating both histories together.

### Fast-Forward Merge
A fast-forward merge happens when the branch you're merging into (e.g., `main`) has had **no new commits** since the branch you're merging from originally diverged.
- Since there's nothing new to reconcile on `main`, Git simply moves `main`'s pointer forward to match the tip of the other branch.
- No new commit is created — the result is a single, straight-line history.
```
Before:  main: A---B          feature: A---B---C---D
After a fast-forward merge: main now simply points directly to D.
```

### Three-Way Merge
A three-way merge happens when **both** branches have diverged with new commits since they last shared a common point.
- Git looks at three references: the tip of your current branch, the tip of the branch being merged in, and their common ancestor commit.
- From these three points, Git creates a brand-new **merge commit**, which has two parent commits, tying both histories together into one.
```
main:      A---B-------M   (M = merge commit, with two parents: B and D)
                \      /
feature:         C----D
```

### Merge Conflicts
A merge conflict occurs when Git cannot automatically reconcile changes on its own — almost always because both branches modified the **same lines** of the **same file** in different ways. Git isn't malfunctioning here; it is deliberately asking a human to make the final judgment call.

Git marks the conflicting section directly inside the affected file:
```
<<<<<<< HEAD
this is your current branch's version of the line(s)
=======
this is the incoming branch's version of the line(s)
>>>>>>> feature-branch
```

**Resolution steps:**
```bash
# 1. Open the file and manually edit it to the correct final content
# 2. Delete the <<<<<<<, =======, and >>>>>>> marker lines entirely
git add <file>              # 3. tell Git this specific conflict is resolved
git commit                    # 4. finalize the merge by creating the merge commit
git merge --abort               # escape hatch at any point — cancels the merge entirely, returning to the pre-merge state
```

---

## 10. Rebasing

The `git rebase <branch>` command takes every commit on your current branch and replays them, one at a time, on top of `<branch>`'s latest commit — producing a clean, linear history with no merge commits at all (unlike a three-way merge).

```bash
git rebase main                  # replay current branch's commits on top of main's latest state
git rebase --continue            # after manually resolving a conflict mid-rebase, continue replaying the remaining commits
git rebase --abort               # cancel the entire rebase, returning your branch to the state it was in before you started
```

### Interactive Rebase
The `git rebase -i` command opens an editable list of your recent commits, letting you rewrite your own local history before sharing it.

```bash
git rebase -i HEAD~3            # opens the last 3 commits for interactive editing
```
Available actions inside an interactive rebase:
- **pick** — keep the commit exactly as it is.
- **reword** — keep the changes, but let you edit the commit message.
- **squash** — merge this commit's changes into the previous commit, combining both commit messages (editable).
- **fixup** — same as squash, but discards this commit's message entirely, keeping only the previous one.
- **drop** — remove the commit entirely from history.

### Merge vs Rebase

| | Merge | Rebase |
| --- | --- | --- |
| History shape | Preserves the true, branching history | Rewrites history into a clean, linear sequence |
| Commit SHAs | Remain unchanged | A brand-new SHA is generated for every replayed commit |
| Safe on shared/pushed branches? | Yes, always safe | **No — never rebase commits that others have already pulled** |
| Typical use case | Integrating a finished feature branch into `main` | Cleaning up your own local branch's history before opening a Pull Request |

**The golden rule of rebasing:** Never rebase a branch that other people are also working on or have already pulled. Because rebase generates new commit SHAs for every replayed commit, anyone who already has the old commits ends up with a diverged, conflicting history compared to yours — a genuinely common and painful mistake on real shared branches.

---

## 11. Working with Remotes

### git remote
The `git remote` command manages the connections between your local repository and one or more remote repositories.

```bash
git remote -v                        # lists all configured remotes along with their URLs (both fetch and push)
git remote add origin <url>           # connects your local repo to a remote, naming it "origin" (the conventional default name)
git remote remove origin                # removes a configured remote connection entirely
git remote set-url origin <new-url>       # updates a remote's URL (e.g., after a repository migration)
```

### git fetch
The `git fetch` command downloads new commits, branches, and tags from a remote repository into your local remote-tracking references, **without** touching or modifying your current working branch in any way.

```bash
git fetch origin                # download new data from the "origin" remote
git fetch --all                   # fetch from every configured remote at once
```
- This is considered the "safe" networking command — it only updates Git's internal bookkeeping (e.g., `origin/main`) so you can inspect what's changed before deciding to integrate it.

### git pull
The `git pull` command is a combination of two steps performed automatically: it runs `git fetch`, and then immediately merges (or rebases) the newly downloaded changes into your current branch.

```bash
git pull                       # fetch + merge into the current branch
git pull --rebase                # fetch + rebase your local commits on top, instead of merging (produces cleaner, linear history)
```
- Because `pull` immediately changes your working branch, it's less "safe" than `fetch` alone — if you want to review incoming changes first, `fetch` followed by manual inspection is the more cautious approach.

### git push
The `git push` command uploads your local commits to a remote repository, making them visible to the rest of your team.

```bash
git push origin <branch>             # upload local commits on <branch> to the "origin" remote
git push -u origin <branch>            # push AND set this remote+branch as the default upstream target (lets you just type "git push" going forward)
git push --tags                          # push all local tags to the remote at once
git push origin --delete <branch>          # delete a branch on the remote (does not affect your local branch)
git push --force                             # ⚠️ overwrites the remote branch's history unconditionally — genuinely risky
git push --force-with-lease                    # a SAFER force push — first checks that the remote hasn't changed since your last fetch, and fails instead of overwriting if it has
```
**Critical interview point:** Always prefer `--force-with-lease` over plain `--force`. It protects you from accidentally destroying a teammate's newly-pushed commits that you simply hadn't fetched yet, by refusing to push if the remote has moved since you last checked it.

---

## 12. Undoing Changes

### git restore
The `git restore` command discards changes, reverting files back to a previous known state — either undoing edits in your Working Directory, or unstaging a file.

```bash
git restore <file>                # discard uncommitted changes in the Working Directory, reverting the file back to its last committed version
git restore --staged <file>         # unstage a file — moves it back to "modified" without discarding the actual edits themselves
```

### git reset
The `git reset` command moves the current branch's pointer backward to an earlier commit, effectively "undoing" the commit(s) after that point — with three different levels of impact on your actual file changes.

```bash
git reset --soft HEAD~1           
# undoes the last commit, but keeps all its changes STAGED, ready to be re-committed

git reset --mixed HEAD~1            
# undoes the last commit, keeps the changes but moves them to UNSTAGED (this is the default mode if no flag is given)

git reset --hard HEAD~1               
# undoes the last commit AND completely discards all associated changes —  destructive, difficult to recover from without reflog
```

### git revert
The `git revert` command creates a brand-new commit that introduces the exact opposite of an earlier commit's changes, effectively cancelling it out — without deleting or altering the original commit in history at all.

```bash
git revert <commit-sha>           # create a new commit that reverses the specified commit
git revert --no-commit <sha>        # apply the reverting changes to the working directory/staging area WITHOUT auto-committing — lets you batch multiple reverts into a single combined commit
```

### git reset vs git revert
`git reset` rewrites history by physically moving the branch pointer backward, which changes what commits exist on the branch. This is safe only for local commits that nobody else has already seen or pulled. 
`git revert` instead adds a brand-new commit that cancels out an old change while leaving all original history fully intact — this makes it the only genuinely safe option for undoing something on a shared or production branch, since it doesn't disrupt anyone else's copy of history.

---

## 13. Recovery & Utility Tools

### git reflog
The `git reflog` command displays a chronological log of every position `HEAD` has pointed to on your local machine — including commits that no longer appear in a normal `git log` after operations like a reset.

```bash
git reflog                     # shows every HEAD movement: commits, resets, checkouts, rebases, and more
```
- This is your safety net after an accidental `git reset --hard` or a similar destructive action. You find the SHA of the commit from right before the mistake, then use `git reset --hard <sha>` or `git cherry-pick <sha>` to bring the lost work back.
- Reflog entries aren't kept forever — by default they typically persist for around 90 days before Git's garbage collection may remove them.

### git stash
The `git stash` command temporarily saves all of your uncommitted changes (both staged and unstaged) somewhere safe, and restores a clean Working Directory — without requiring you to make a commit.

- This is useful when you need to switch context quickly (e.g., an urgent bug fix request) but aren't ready to commit your half-finished work yet.
- Think of it as "putting your unfinished work in a drawer" so you can come back to it later exactly as you left it.

```bash
git stash                        # stash all changes to TRACKED files (both staged and unstaged)
git stash -u                     # ALSO stash untracked files (brand-new files not yet added) — easy to forget, and often needed
git stash list                    # view all stashes currently saved, most recent first
git stash pop                     # reapply the most recent stash to your working directory AND remove it from the stash list
git stash apply                   # reapply the most recent stash but KEEP it in the stash list (useful if you need to apply the same stash to multiple branches)
git stash apply stash@{2}            # apply a specific, older stash by referencing its index
git stash drop stash@{1}             # delete one specific stash without applying it
git stash clear                       # delete every stash currently saved
git stash show -p stash@{0}            # preview the actual diff contents of a stash before applying it
```
**Critical interview point:** By default, `git stash` does **not** include untracked (brand-new) files — only files Git is already tracking. This causes a common real-world mistake: someone stashes, switches branches expecting a totally clean state, but their new files are still sitting in the Working Directory. Use `git stash -u` whenever new files need to be included too.

### git cherry-pick
The `git cherry-pick` command applies the changes from one specific commit on another branch onto your current branch, without bringing in any of that branch's other commits.

```bash
git cherry-pick <commit-sha>         # apply one specific commit onto the current branch
git cherry-pick <sha1> <sha2>           # cherry-pick multiple specific commits, one after another
git cherry-pick --continue                # after resolving a conflict during a cherry-pick, continue the operation
git cherry-pick --abort                     # cancel the cherry-pick entirely
```
- A classic real-world use case: a critical hotfix is committed directly to `main`, and that exact same fix needs to be ported into a `release` branch, without merging every other unrelated, in-progress change from `main`.

### git bisect
The `git bisect` command performs an automated binary search through your commit history to find the exact commit that introduced a bug.

```bash
git bisect start                # begin a bisect session
git bisect bad                    # mark the CURRENT commit as containing the bug
git bisect good <old-sha>            # mark a known-good older commit as a starting reference point
                                          # Git will now automatically check out a commit roughly halfway between the good and bad points for you to test
git bisect good                          # (or) git bisect bad — you tell Git the result for each commit it presents, and it narrows the range further
git bisect reset                             # end the session and return to your original branch/commit
```
- This narrows down the exact culprit commit in a logarithmic number of steps, which is dramatically faster than manually testing every commit one by one across a large history.

---

## 14. Repository Hygiene

### .gitignore
The `.gitignore` file lists file and folder patterns that Git should never track or show as "untracked" — commonly used for build artifacts, dependency folders, secrets, and temporary files.

```
.env
node_modules/
*.log
.terraform/
*.tfstate
```

### git rm --cached
The `git rm --cached` command stops Git from tracking a specific file going forward, while leaving the actual file untouched on your disk.

```bash
git rm --cached <file>              # stop tracking a single file (keeps it on disk)
git rm --cached -r <folder>            # stop tracking an entire folder, recursively
```
**Critical interview point:** `.gitignore` only affects files that Git is not **already** tracking. If a file was committed to the repository before it was added to `.gitignore`, simply adding the pattern does nothing on its own — you must explicitly run `git rm --cached` on that file first to actually stop tracking it.

### Removing secrets from Git history
Because Git's history is permanent by design, a secret committed once (an API key, password, token, etc.) remains in history forever — even if it's deleted in a later commit — unless the history itself is deliberately rewritten.

```bash
git filter-repo --path secrets.env --invert-paths     # strip a specific file entirely from ALL historical commits (modern, recommended tool over the older git filter-branch)
```
**Real-world remediation steps for a leaked secret:**
1. Rotate/revoke the exposed credential immediately — treat it as compromised the moment it exists in Git history, regardless of whether it can be removed.
2. Use `git filter-repo` (or BFG Repo-Cleaner) to purge the secret from every historical commit.
3. Force-push the cleaned history to the remote repository.
4. Notify the entire team — everyone must re-clone or hard-reset, since every commit SHA downstream of the purge point will have changed.
5. Add prevention for the future: proper `.gitignore` coverage, plus a pre-commit secret-scanning hook (e.g., Gitleaks).

---

## 15. Tags

A **tag** is a permanent, named pointer to one specific commit — most commonly used to mark a release version, such as `v1.0.0`.

```bash
git tag v1.0.0                          # creates a LIGHTWEIGHT tag — just a name pointing at a commit, nothing more
git tag -a v1.0.0 -m "First release"      # creates an ANNOTATED tag — includes a message, date, and author metadata (the recommended type for real releases)
git tag                                     # lists all tags in the repository
git tag -d v1.0.0                             # deletes a tag locally
```

```bash
git push origin v1.0.0            # tags are NOT pushed automatically along with regular commits — they must be pushed explicitly
git push origin --tags               # pushes every local tag to the remote at once
git push origin --delete v1.0.0        # deletes a tag on the remote repository
```
**Why this matters for DevOps:** CI/CD pipelines very commonly trigger an automated release build or deployment specifically when a tag matching a pattern like `v*` is pushed — this is one of the most standard release-automation patterns used in the industry.

---

## 16. Git Hooks

A **Git hook** is a script that Git runs automatically at a specific point in its workflow, stored inside the `.git/hooks/` folder. These are not versioned/shared automatically by default — teams typically use a framework like the `pre-commit` tool or Husky to version and share hook configuration across everyone's machine.

| Hook | Runs when | Common DevOps use |
| --- | --- | --- |
| `pre-commit` | Just before a commit is created | Linting, code formatting, secret scanning (e.g., Gitleaks) |
| `commit-msg` | Just after the commit message is written | Enforce message conventions (e.g., Conventional Commits format) |
| `pre-push` | Just before pushing to a remote | Run automated tests; block the push entirely if they fail |
| `post-receive` | On the server, right after a push is received | Trigger a deployment (an older-style pattern, largely replaced by modern CI/CD platforms) |

**Why it matters:** Hooks let a team automatically enforce quality and safety checks *before* bad code or secrets ever leave a developer's own machine, rather than catching the problem later during CI or code review.

---

## 17. Branching Workflows / Strategies

### Git Flow
Git Flow is a workflow built around long-lived `main` and `develop` branches, supplemented by short-lived `feature/*`, `release/*`, and `hotfix/*` branches for specific purposes.
- It's well suited to larger teams working with scheduled, versioned releases.
- It's considered a heavier process today, and is mostly seen in larger, slower-moving enterprises rather than fast-moving startups.

### GitHub Flow
GitHub Flow is a simpler workflow using just a `main` branch plus short-lived feature branches, integrated back in through Pull Requests.
- It's the most common pattern in the industry today, especially for teams practicing continuous deployment.

### Trunk-Based Development
Trunk-Based Development has every developer committing directly (and frequently) to `main` ("the trunk"), using feature flags to hide unfinished work from end users instead of relying on long-lived feature branches.
- It requires high CI/CD maturity, since integration happens constantly and needs to be safe and automated.
- It enables very fast release cycles due to minimal merge conflicts and small, frequent changes.

**Why this matters for interviews:** Know that **GitHub Flow** and **Trunk-Based Development** are the modern, industry-favored patterns (faster integration, smaller Pull Requests, less painful merging), while Git Flow is viewed as the older, heavier approach.

---

## 18. Real-World DevOps Ties

- **CI/CD triggers:** GitHub Actions and Jenkins pipelines are triggered directly by Git events — for example, `on: push: branches: [main]`, `on: pull_request`, or a tag being pushed.
- **GitOps (Argo CD / Flux):** A Git repository containing Kubernetes manifests or Helm charts is treated as the single source of truth for what should be running. Argo CD continuously watches this repository and automatically syncs the live cluster to match it — deployments happen exclusively through a `git push` (e.g., updating an image tag), never through a manual `kubectl apply` in production. This gives a complete audit trail, and makes rollback as simple as a `git revert`.
- **Terraform + Git:** Infrastructure changes are proposed through a Pull Request, where the `terraform plan` output is reviewed (often posted automatically as a PR comment) before merging triggers `terraform apply` inside a CI pipeline — preventing unreviewed infrastructure changes from reaching real cloud resources directly.
- **Branch protection rules:** Real production repositories lock down `main` — requiring Pull Request reviews, requiring CI status checks to pass, and disallowing force-pushes — specifically to prevent the exact mistakes covered above (destructive force-pushes, unreviewed merges, broken production history).
- **Secret scanning (Gitleaks):** Since Git's history is permanent, a leaked secret remains in it forever unless deliberately purged — this is precisely why pre-commit secret scanning exists, to catch the problem *before* the commit is ever made.

---

## Quick Reference Cheat Sheet

```bash
# Setup
git config --global user.name/email

# Start a repo
git init | git clone <url>

# Everyday loop
git status
git add <file>/. | git add -p
git commit -m "msg" | git commit --amend
git log --oneline --graph --all

# Branching
git branch | git switch -c <name> | git checkout -b <name>
git branch -d <name>   # safe delete | -D force delete

# Merge & Rebase
git merge <branch>
git rebase <branch> | git rebase -i HEAD~n
git rebase --continue / --abort

# Remotes
git remote -v | git fetch | git pull --rebase
git push -u origin <branch> | git push --force-with-lease

# Undo
git restore <file> | git restore --staged <file>
git reset --soft/mixed/hard HEAD~1
git revert <sha>

# Recovery & utility
git reflog
git stash | git stash -u | git stash pop | git stash apply | git stash list
git cherry-pick <sha>
git bisect start/good/bad/reset

# Hygiene
git rm --cached <file>
git filter-repo --path <file> --invert-paths

# Tags
git tag -a v1.0.0 -m "msg" | git push origin --tags
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What is version control, and why is it needed?**
> A: Version control tracks changes to files over time — recording who changed what, when, and why — while allowing multiple people to work on the same project without overwriting each other's changes. It also lets you revert to any earlier state whenever needed.

**Q2: What are the three types of VCS, and why did the industry move to distributed systems like Git?**
> A: Local VCS (single machine, no collaboration), Centralized VCS like SVN (one server holds all history — a single point of failure), and Distributed VCS like Git (every clone has the complete history). The industry moved to distributed systems because almost every operation — commit, log, diff, branch — works offline and instantly, and there's no single point of failure if the central server goes down.

**Q3: What's the difference between Git and GitHub?**
> A: Git is the version control tool itself, and it works entirely locally with no dependency on the internet. GitHub is a cloud platform that hosts Git repositories and layers collaboration features on top — Pull Requests, Issues, code review, CI/CD runners, and access control.

**Q4: Explain what actually happens when you run `git commit`. Does Git store diffs or snapshots?**
> A: Git stores a full snapshot of the entire project state at that moment (via the tree/blob object model), not a diff. A commit points to one tree object representing that snapshot, along with its parent commit(s), author, and message. Diffs are computed on the fly for display (e.g., in `git diff` or `git log -p`) — they aren't what's actually stored.

**Q5: What's the difference between `git merge` and `git rebase`, and when would you use each?**
> A: Merge combines two branches' histories with a new merge commit, preserving true, non-linear history — it's safe to use anytime, including on shared/public branches. Rebase replays your branch's commits on top of another branch, producing clean, linear history with no merge commits — but it generates new commit SHAs, so it should only be used on your own local branch before it's shared with anyone else. Never rebase commits that others have already pulled.

**Q6: What's a fast-forward merge versus a three-way merge?**
> A: A fast-forward merge happens when the target branch has no new commits since you branched off — Git just moves the pointer forward, no merge commit is created. A three-way merge happens when both branches have diverged with new commits — Git uses the common ancestor plus both branch tips to create a new merge commit with two parents.

**Q7: How do you resolve a merge conflict?**
> A: Git marks the conflicting sections in the file with `<<<<<<<`, `=======`, and `>>>>>>>` markers. You manually edit the file to the correct final content, delete the marker lines, then run `git add <file>` to mark it resolved, followed by `git commit` to finalize the merge (or `git rebase --continue` if the conflict happened during a rebase instead).

**Q8: What's the difference between `git reset` and `git revert`, and which would you use on a production branch?**
> A: `git reset` rewrites history by moving the branch pointer backward — it's destructive to shared history and should only be used on local, unpushed commits. `git revert` creates a brand-new commit that cancels out an earlier one, leaving all original history intact — this makes it the safe choice for undoing a change on a shared or production branch.

**Q9: What's the difference between `git fetch` and `git pull`?**
> A: `git fetch` downloads new commits and branches from the remote into your remote-tracking refs, without touching your current working branch at all. `git pull` does a fetch AND automatically merges (or rebases, with `--rebase`) those changes into your current branch. `fetch` is the safer, "look before you act" option.

**Q10: Why would you use `--force-with-lease` instead of `--force` when force-pushing?**
> A: `--force` overwrites the remote branch unconditionally, which can silently destroy a teammate's commits you simply hadn't fetched yet. `--force-with-lease` first checks that the remote hasn't changed since your last fetch, and fails safely if it has — protecting against accidentally clobbering someone else's work.

**Q11: What is a detached HEAD state, and how do you recover from it safely?**
> A: It's when `HEAD` points directly to a specific commit instead of a branch — typically after checking out a commit hash directly rather than a branch name. Any new commits made in this state can be lost once you switch away, unless you first create a branch (`git branch <name>` or `git switch -c <name>`) to preserve that work.

**Q12: You just ran `git reset --hard` and realize you deleted commits you needed. How do you get them back?**
> A: `git reflog` shows a log of every place HEAD has pointed to, including commits that no longer appear in `git log` after a reset. Find the SHA of the commit right before the reset, then either `git reset --hard <sha>` to fully restore that state, or `git cherry-pick <sha>` to selectively bring back specific commits.

**Q13: What's the difference between `git stash` and `git stash -u`, and why does this distinction matter practically?**
> A: `git stash` by default only shelves changes to files Git is already tracking — it does NOT stash new, untracked files. `git stash -u` also includes untracked files. This is a common real-world gotcha: someone stashes, switches branches assuming a clean slate, but their new files are still sitting in the working directory, sometimes causing confusing conflicts with the other branch's state.

**Q14: What is `git cherry-pick` used for, with a real example?**
> A: It applies one specific commit from another branch onto your current branch, without merging everything else from that branch. A classic use case: a critical hotfix is committed directly to `main`, and you need that exact same fix ported into a `release` branch without bringing in `main`'s other unrelated, in-progress changes.

**Q15: How would you find the exact commit that introduced a bug across a large number of commits?**
> A: `git bisect`. You mark a known-bad commit (usually the current one) and a known-good older commit, and Git performs a binary search — checking out commits halfway between the two, which you test and mark `good` or `bad`. This narrows down the exact culprit commit in a logarithmic number of steps rather than checking every commit sequentially.

**Q16: A secret (API key) was accidentally committed and pushed. What's your remediation process?**
> A: First and most importantly, rotate/revoke the exposed credential immediately — treat it as compromised the moment it's in Git history, regardless of whether you can remove it. Then use `git filter-repo` (or BFG Repo-Cleaner) to strip it from all historical commits, force-push the cleaned history, and notify the team to re-clone since all downstream commit SHAs will have changed. Going forward, add `.gitignore` coverage and a pre-commit secret-scanning hook like Gitleaks to prevent recurrence.

**Q17: What is a branch, internally, and why is branching so fast in Git compared to older VCS tools?**
> A: A branch is just a small text file in `.git/refs/heads/` containing a single commit's SHA hash — it's a pointer, not a copy of any files. Creating or switching a branch only creates or moves this pointer, which is why it's essentially instant, unlike older centralized VCS tools where branching could be slow and resource-heavy.

**Q18: Your `.gitignore` entry doesn't seem to be working for a file that's still showing up in `git status`. What's the likely cause and fix?**
> A: The most common cause is that the file was already being tracked by Git *before* it was added to `.gitignore` — the ignore rule only applies to untracked files. The fix is `git rm --cached <file>` to stop tracking it (keeping it on disk), after which the `.gitignore` rule will correctly apply going forward.

**Q19: What are Git hooks, and give one practical DevOps example.**
> A: Hooks are scripts Git runs automatically at specific points in the workflow, stored in `.git/hooks/`. A practical example: a `pre-commit` hook running Gitleaks to scan staged changes for secrets, blocking the commit entirely if a potential credential is detected — catching the problem before it ever leaves the developer's machine.

**Q20: How does Git tie into a GitOps workflow with a tool like Argo CD?**
> A: In GitOps, a Git repository holding Kubernetes manifests (or Helm charts) is treated as the single source of truth for what should be running in the cluster. Argo CD continuously watches this repo and automatically syncs the live cluster state to match it. Deployments happen exclusively through Git commits (e.g., updating an image tag and pushing), never through direct `kubectl apply` — which gives a complete audit trail and makes rollback as simple as a `git revert`.