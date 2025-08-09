## 1. Working on a Remote Server/VM with My Team, What Should We Do First?
**Problem:** You need to work on a shared server or VM but want to avoid exposing your Git account credentials.

**Solution:** Use SSH keys for secure, password-less authentication.

1.  **Generate an SSH key on the VM:**
    ```bash
    # Use a modern, secure algorithm like ed25519
    ssh-keygen -t ed25519 -C "your_email@example.com"
    ```
2.  **Display your new public key:**
    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```
3.  **Add the key to your Git provider:** Copy the output from the `cat` command and paste it into the "SSH and GPG keys" section of your GitHub, GitLab, or Bitbucket account.

4.  **Clone using SSH:**
    ```bash
    git clone git@github.com:your-org/your-repo.git
    ```
> **Takeaway:** Using SSH is a security best practice for shared environments.

---

## 2. Our Repo is Huge and `git clone` Takes Forever. What Can I Do?
**Problem:** Cloning a large repository with a deep history takes a very long time and consumes significant disk space.

**Solution:** Perform a **shallow clone** to download only the most recent history.

1.  **Clone only the latest commit:**
    ```bash
    # --depth 1 gets only the tip of the branch history
    git clone --depth 1 https://github.com/massive-org/massive-repo.git
    ```
2.  **To get more history later (optional):**
    ```bash
    # Fetch the next 50 commits
    git fetch --depth 50

    # Or convert to a full clone by fetching all history
    git fetch --unshallow
    ```
> **Takeaway:** Ideal for CI/CD pipelines or when you don't need the full project history to start working.

---

## 3. How Can We Automatically Run Linters or Tests Before Anyone Commits?
**Problem:** Teammates push code with simple syntax errors or failing tests, breaking the build.

**Solution:** Use **Git Hooks**, specifically `pre-commit` hooks, to act as an automated quality gate. The [`pre-commit`](https://pre-commit.com/) framework makes this easy.

1.  **Create a `.pre-commit-config.yaml` file in your project root:**
    ```yaml
    repos:
    -   repo: https://github.com/pre-commit/pre-commit-hooks
        rev: v4.3.0
        hooks:
        -   id: check-yaml
        -   id: end-of-file-fixer
        -   id: trailing-whitespace
    -   repo: https://github.com/psf/black
        rev: 22.6.0
        hooks:
        -   id: black # Python code formatter
    ```
2.  When a developer runs `git commit`, the hooks run automatically. If a hook fails, the commit is aborted.
    ```bash
    $ git commit -m "Add new feature"
    black................................................Failed
    - hook id: black
    - files were modified by this hook
    ```
> **Takeaway:** Enforce code standards automatically and prevent simple mistakes from ever reaching the shared repository.

---

## 4. I Accidentally Committed a Password/API Key, How Do I Erase It From History?
**Problem:** You committed and pushed a sensitive file or hardcoded secret. Deleting it in a new commit is not enough; it's still in the history.

**Solution:** Use a tool like `git filter-repo` to completely remove the file from all commits.

1.  **Remove the sensitive file from your current code:**
    ```bash
    # Also add the file/path to your .gitignore
    rm path/to/your/secret.file
    ```
2.  **Rewrite history to erase the file:**
    ```bash
    git filter-repo --path path/to/your/secret.file --invert-paths
    ```
3.  **Force push the rewritten history:**
    ```bash
    # This is a required but destructive action. Coordinate with your team.
    # --force-with-lease is slightly safer than --force.
    git push origin --force --all
    ```
> **CRITICAL:** After cleaning the repository, **you must invalidate the leaked secret** (rotate API keys, change passwords). The secret is considered compromised the moment it was pushed.

---

## 5. A Bug Was Introduced Weeks Ago, How Can I Find the Exact Commit That Broke Everything?
**Problem:** A bug exists on the `main` branch, but you don't know which of the last hundreds of commits introduced it.

**Solution:** Use `git bisect` to perform an automated binary search through your commit history.

1.  **Start the bisect session:**
    ```bash
    git bisect start
    ```
2.  **Mark the current commit as bad and a known working commit as good:**
    ```bash
    # Mark current state as broken
    git bisect bad

    # Mark a past commit/tag where it worked
    git bisect good v1.2.0 # or a commit hash
    ```
3.  **Test and repeat:** Git will check out a commit halfway between `good` and `bad`. Test your code, then run `git bisect good` or `git bisect bad` accordingly. Git will continue narrowing it down.

4.  **Finish the session:** Once Git pinpoints the "first bad commit," reset your HEAD back to where you started.
    ```bash
    git bisect reset
    ```
> **Takeaway:** `git bisect` turns hours of manual guesswork into a few minutes of guided testing.

---

## 6. Our Project Depends on Another Repo, Should We Use Submodules or Subtrees?
**Problem:** You need to include code from another repository (e.g., a shared library) within your project.

**Solution:** Choose between `git submodule` or `git subtree` based on your needs.

#### Git Submodule
A pointer to a specific commit in another repo. The code is kept separate.

*   **Add a submodule:**
    ```bash
    git submodule add https://github.com/your-org/shared-library.git libs/shared-library
    ```
*   **Cloning a repo with submodules:**
    ```bash
    git clone https://github.com/your-org/main-project.git
    cd main-project
    git submodule update --init --recursive
    ```
*   **Pros:** Keeps projects separate, strict version control.
*   **Cons:** Adds complexity for collaborators (extra commands needed).

#### Git Subtree
Copies the files and history from another repo directly into your project.

*   **Add a subtree:**
    ```bash
    # --squash creates a single commit for the library's history
    git subtree add --prefix=libs/shared-library https://github.com/your-org/shared-library.git main --squash
    ```
*   **Pros:** Simple for collaborators (a regular `git clone` just works).
*   **Cons:** Can bloat the main repo's history; pulling updates is more manual.

> **Takeaway:** Use **submodules** for strict separation and versioning. Use **subtrees** for simplicity and a self-contained repo.

---

## 7. How Do We Properly Tag Releases and Manage Hotfixes on Old Versions?
**Problem:** A critical bug is found in an old, tagged release (`v1.5.0`), but `main` has already advanced to `v2.0.0+`.

**Solution:** Create a `hotfix` branch from the old release tag, fix the bug, then merge the fix into all relevant long-lived branches (`main`, `develop`).

1.  **Create a hotfix branch from the old tag:**
    ```bash
    git checkout -b hotfix/v1.5.1 v1.5.0
    ```
2.  **Commit the fix:**
    ```bash
    # ...edit files...
    git add .
    git commit -m "Hotfix: Patch critical security vulnerability"
    ```
3.  **Merge the hotfix back into `main` and `develop`:**
    ```bash
    git checkout main
    git merge hotfix/v1.5.1

    git checkout develop
    git merge hotfix/v1.5.1
    ```
4.  **Tag the new hotfix release and push everything:**
    ```bash
    # Create an annotated tag for the new release
    git tag -a v1.5.1 -m "Version 1.5.1"

    # Push the branches and the new tag
    git push origin main develop
    git push origin v1.5.1
    ```
> **Takeaway:** This disciplined process allows you to patch old versions without disrupting current development.

---

## 8. I Messed Up and Lost My Commits. Is There a Way to Get Them Back?
**Problem:** You ran `git reset --hard` too far or deleted a branch by mistake and your commits are gone.

**Solution:** Use the **reflog**, Git's secret safety net that logs every move your `HEAD` makes.

1.  **View your action history:**
    ```bash
    git reflog
    ```
    ```
    # Output will look like this:
    a1b2c3d HEAD@{0}: reset: moving to HEAD~3
    f4e5d6c HEAD@{1}: commit: Add user dashboard component # <- This is the commit I want back
    g7h8i9j HEAD@{2}: commit: Fix login validation rules
    ```
2.  **Restore your lost work:** Find the hash of the state you want to return to (`f4e5d6c` in this case).
    ```bash
    # Option A (Safe): Create a new branch from that state
    git checkout -b recovered-work f4e5d6c

    # Option B (Confident): Reset your current branch to that state
    git reset --hard f4e5d6c
    ```
> **Takeaway:** The reflog is local and temporary (defaults to 90 days), but it can save you from catastrophic mistakes.

---

## 9. Who Wrote This Line of Code and Why?
**Problem:** You encounter a confusing line of code and need to understand its purpose and history.

**Solution:** Use `git blame` to find the commit that introduced the line, then `git show` to see the full context of that commit.

1.  **Find the author and commit for each line in a file:**
    ```bash
    git blame path/to/the/file.js
    ```
    ```
    # Output shows commit hash, author, date, and line
    f4e5d6c (John Ssuza  2024-08-01...) 3) // Temp fix for negative values, per TICKET-123
    ```
2.  **Inspect the full commit details:**
    ```bash
    git show f4e5d6c
    ```
> **Takeaway:** `blame` isn't for pointing fingers; it's for uncovering the "why" behind code.

---

## 10. How Do I Keep My Fork in Sync with the Original ‘Upstream’ Repo?
**Problem:** Your forked repository is falling behind the original ("upstream") repository, which will lead to merge conflicts in your Pull Requests.

**Solution:** Add the original repository as a remote called `upstream` and regularly sync your `main` branch.

1.  **Add the upstream remote (only need to do this once):**
    ```bash
    git remote add upstream https://github.com/Original-Org/your-repo.git
    ```
2.  **Verify your remotes:**
    ```bash
    git remote -v
    # You should now see both 'origin' (your fork) and 'upstream' (the original)
    ```
3.  **Sync your `main` branch with upstream:**
    ```bash
    # Switch to your local main branch
    git checkout main

    # Fetch the latest changes from the original repo
    git fetch upstream

    # Rebase your local main on top of the original repo's main
    git rebase upstream/main

    # Push the updated main branch to your fork (origin)
    git push origin main
    ```
> **Takeaway:** Regularly syncing with `upstream` is a fundamental workflow for contributing to open source or large team projects.

---

## 11. How Do I Prove My Commits Really Came From Me? (Signing Commits)
**Problem:** You want to add a layer of security and trust to your commits, proving they weren't forged.

**Solution:** Sign your commits with a GPG key. GitHub/GitLab will show a "Verified" badge.

1.  **Generate a GPG key:**
    ```bash
    gpg --full-generate-key
    ```
2.  **List your keys and find the ID:**
    ```bash
    gpg --list-secret-keys --keyid-format=long
    # Look for the string after `sec ed25519/` -> e.g., A1B2C3D4E5F6G7H8
    ```
3.  **Configure Git to use your key:**
    ```bash
    # Replace with your key ID
    git config --global user.signingkey A1B2C3D4E5F6G7H8
    # Tell Git to sign all commits by default
    git config --global commit.gpgsign true
    ```
4.  **Add your public GPG key to your Git provider:**
    ```bash
    # Copy the output and paste it in your account settings
    gpg --armor --export A1B2C3D4E5F6G7H8
    ```
> **Takeaway:** Signed commits increase the integrity and authenticity of your repository's history.

---

## 12. What is a “Detached HEAD” State and How Do I Fix It Without Losing My Work?
**Problem:** You ran `git checkout <commit-hash>` and now you're in a "detached HEAD" state.

**Solution:** This just means `HEAD` is pointing to a commit, not a branch. It's not an error.

*   **If you were just looking and made no changes:** Simply switch back to a branch.
    ```bash
    git switch main
    ```
*   **If you made commits and want to save them:** Create a new branch right where you are. This will "attach" your `HEAD` and save your work.
    ```bash
    git switch -c new-feature-from-old-state
    ```
> **Takeaway:** Don't panic. A detached HEAD is a normal state. If you do any work, just create a branch to save it.

---

## 13. Our Repo is Full of Large Assets, How Do We Handle Them Without Bloating the History?
**Problem:** Your repository is slow and bloated because large binary files (datasets, videos, design files) have been committed.

**Solution:** Use **Git Large File Storage (LFS)** to store pointers in Git, while the actual files live on a separate server.

1.  **Install the LFS extension (one-time setup):**
    ```bash
    git lfs install
    ```
2.  **Tell LFS which file types to track:**
    ```bash
    git lfs track "*.psd"
    git lfs track "assets/videos/*.mp4"
    ```
3.  **Commit the `.gitattributes` file:** This file tells Git which files are managed by LFS.
    ```bash
    git add .gitattributes
    git commit -m "Configure Git LFS"
    ```
4.  **Work normally:** `git add`, `commit`, and `push` large files as you usually would. LFS handles them automatically in the background.
> **Takeaway:** Git LFS gives you version control for large assets without killing repository performance.

---

## 14. This Monorepo is Too Big. How Do I Split a Folder into a New Repository?
**Problem:** You need to extract a sub-folder from a monorepo into its own separate repository, preserving its specific commit history.

**Solution:** Use `git filter-repo` to rewrite the project history, keeping only the history related to that folder.

**Warning: Back up your repository before starting. This is a destructive operation.**

1.  **Create a bare clone of the monorepo:**
    ```bash
    git clone --bare https://github.com/my-org/monorepo.git
    cd monorepo.git
    ```
2.  **Filter the history to only include the desired folder:**
    ```bash
    # This makes it seem as if 'mobile-app/' was always the root directory.
    git filter-repo --path mobile-app/
    ```
3.  **Push the filtered history to a new, empty repository:**
    ```bash
    # Create the new repo on GitHub/GitLab first.
    git remote set-url origin https://github.com/my-org/new-mobile-app-repo.git
    git push --mirror
    ```
> **Takeaway:** A powerful technique for major repository refactoring, but it requires caution and planning.

---

## 15. I Committed the Wrong Code, How Do I Undo My Error?
**Problem:** You made a commit locally but it contains a mistake. You have not pushed it yet.

**Solution:** Use `git reset` to undo the last commit while keeping your code changes.

*   **Undo commit, keep changes staged:**
    ```bash
    # Your files remain staged, ready to be modified and re-committed.
    git reset --soft HEAD~1
    ```
*   **Undo commit, keep changes unstaged:**
    ```bash
    # Your files are modified in your working directory but not staged.
    git reset --mixed HEAD~1 # This is the default
    ```
*   **Undo commit and delete changes (DANGEROUS):**
    ```bash
    # Wipes out the commit AND all code changes associated with it.
    git reset --hard HEAD~1
    ```
> **Takeaway:** For most local mistakes, `git reset --soft HEAD~1` is the safest and most useful option.

---

## 16. When My Work Collided with a Teammate Changes, Now I Can’t Pull?
**Problem:** `git pull` fails because you have local, uncommitted changes that conflict with incoming changes from the remote.

**Solution:** Stash your local changes, pull the remote updates, then re-apply your stashed work.

1.  **Stash your current changes:**
    ```bash
    # `save` adds a descriptive message, `-u` includes new (untracked) files.
    git stash save "my-wip-feature" -u
    ```
2.  **Pull the latest from the remote:**
    ```bash
    git pull
    ```
3.  **Re-apply your stashed changes:**
    ```bash
    # You can see all stashes with `git stash list`
    git stash apply stash@{0}
    ```
> **Takeaway:** Stashing is a clean way to temporarily set aside your work, update your branch, and then continue where you left off.

---

## 17. Why Did Applying My Stash Cause Conflicts?
**Problem:** After running `git stash apply`, you get a merge conflict error.

**Solution:** This is normal and expected. It means that both your stashed changes and the code you pulled from the remote have modified the exact same lines in a file.

*   **Open the conflicting file(s).**
*   **Look for the conflict markers:**
    ```
    <<<<<<< HEAD (Current Change from remote)
    // Code that your teammate wrote
    =======
    // Code from your stash
    >>>>>>> stash
    ```
*   **Resolve the conflict:** Edit the code to combine the changes, or choose one version to keep, and remove the `<<<`, `===`, `>>>` markers.
*   **Stage the resolved file** and continue your work.

> **Takeaway:** Conflict resolution is a core Git skill. Modern editors like VS Code have excellent built-in tools to make this process easier.

---

## 18. I Just Committed but Forgot to Add a Few Files?
**Problem:** You made a commit but forgot to include a file or two.

**Solution:** **Amend** the previous commit to add the missing files. This avoids creating a new, separate "Oops, forgot a file" commit.

1.  **Stage the missing file(s):**
    ```bash
    git add path/to/forgotten/file.js
    ```
2.  **Amend the previous commit:**
    ```bash
    # --no-edit keeps the original commit message without prompting you to change it.
    git commit --amend --no-edit
    ```
*   **To change the commit message as well:**
    ```bash
    git commit --amend -m "A better, more descriptive message"
    ```
> **Takeaway:** `amend` is great for cleaning up local commits. If you've already pushed, you'll need to force push (`git push --force-with-lease`), which should be done with caution on shared branches.

---

## 19. My Branch Name Doesn’t Match the Work Anymore?
**Problem:** The scope of your feature branch has changed, and its name is now misleading.

**Solution:** Rename the branch locally and update the remote to match.

1.  **Rename the local branch:**
    ```bash
    # Make sure you are on the branch you want to rename
    git branch -m new-descriptive-name
    ```
2.  **Push the newly named branch to the remote:**
    ```bash
    git push origin new-descriptive-name
    ```
3.  **Delete the old branch from the remote:**
    ```bash
    git push origin --delete old-misleading-name
    # Or the old shorthand: git push origin :old-misleading-name
    ```
4.  **Link your local branch to the new remote branch:**
    ```bash
    git push --set-upstream origin new-descriptive-name
    ```
> **Takeaway:** Clear branch names improve communication and make your repository history easier to understand.

---

## 20. You Already Pushed Your Commit, But Now You Need to Update It?
**Problem:** You pushed a commit and then noticed a typo or a forgotten change.

**Solution:** Amend the commit locally, then **force push** to update the remote branch.

1.  **Make your fix and stage it:**
    ```bash
    # ...edit file...
    git add path/to/fixed/file.js
    ```
2.  **Amend the last commit:**
    ```bash
    git commit --amend --no-edit
    ```
3.  **Force push with lease:**
    ```bash
    git push --force-with-lease
    ```
> **Takeaway:** Use `--force-with-lease` instead of `--force`. It's safer because it won't overwrite the remote branch if another teammate has pushed new work to it since you last pulled. **Coordinate with your team before force pushing to a shared branch.**

---

## 21. My Push Rejected, I Then Pulled… and Now Conflicts Everywhere!
**Problem:** Your `git push` was rejected. You ran `git pull`, which created a merge commit and now you have conflicts.

**Solution:** This is a standard Git workflow. The remote had changes you didn't have. You must resolve the conflicts before you can push.

1.  The rejection message means you need to integrate remote changes first:
    ```
    ! [rejected]        dev -> dev (fetch first)
    hint: Updates were rejected because the remote contains work that you do not have locally.
    ```
2.  `git pull` attempts to download and merge. If conflicts occur, Git will stop and tell you.
3.  **Resolve the conflicts:** Open the files, fix the code within the `<<<`, `===`, `>>>` markers.
4.  **Complete the merge:**
    ```bash
    git add <path/to/resolved/file.js>
    # Git may have already prepared a commit message for you.
    git commit
    ```
5.  **Now you can push:**
    ```bash
    git push
    ```
*   **If the merge is too messy and you want to start over:**
    ```bash
    git merge --abort
    ```
> **Takeaway:** A push rejection is not an error; it's Git protecting the history. Always pull and resolve conflicts before pushing to a busy branch.

---

## 22. Push Rejected Again? This Time Let’s Rebase Instead of Merging
**Problem:** Your push was rejected. You want to integrate remote changes, but you prefer a clean, linear history without extra merge commits.

**Solution:** Use `git pull --rebase` instead of a standard `git pull`.

1.  **Pull using the rebase strategy:**
    ```bash
    git pull --rebase
    ```
    This fetches the remote changes, "un-does" your local commits, applies the remote changes, and then "re-plays" your local commits one-by-one on top of the new code.

2.  **If conflicts occur during the rebase:** Git will pause and ask you to fix the first conflicting commit.
    *   Open the file and resolve the conflict markers.
    *   Stage the resolved file: `git add <file>`.
    *   Continue the rebase: `git rebase --continue`.
    *   Repeat if other commits have conflicts.

3.  **Once the rebase is complete, push your changes:**
    ```bash
    git push
    ```
*   **To cancel a rebase that has gone wrong:**
    ```bash
    git rebase --abort
    ```
> **Takeaway:** Rebasing creates a cleaner, more readable history. However, since it rewrites history, it should be used with care, especially on branches shared by many people.

---

## 23. My PR Shows Conflicts, How Do I Fix It Before the Team Yells at Me?
**Problem:** You opened a Pull Request, but GitHub/GitLab shows "This branch has conflicts that must be resolved."

**Solution:** You need to update your feature branch with the latest changes from the target branch (e.g., `main`) and resolve the conflicts locally. You can use either a **merge** or a **rebase** strategy.

#### Method 1: Merging the Target Branch (Safer for Teams)

1.  **Get the latest version of the target branch:**
    ```bash
    git checkout main
    git pull
    ```
2.  **Switch back to your branch and merge `main` into it:**
    ```bash
    git checkout my-feature-branch
    git merge main
    ```
3.  **Resolve any conflicts** that arise in your editor.
4.  **Commit the resolved merge and push:**
    ```bash
    git add .
    git commit -m "Merge main to resolve conflicts"
    git push
    ```

#### Method 2: Rebasing onto the Target Branch (Cleaner History)

1.  **Get the latest version of the target branch:**
    ```bash
    git checkout main
    git pull
    ```
2.  **Switch back to your branch and rebase it onto `main`:**
    ```bash
    git checkout my-feature-branch
    git rebase main
    ```
3.  **Resolve any conflicts** that arise. For each conflict: resolve, `git add .`, then `git rebase --continue`.
4.  **Force push your rebased branch:**
    ```bash
    # Rebase rewrites history, so a force push is required.
    git push --force-with-lease
    ```
> **Takeaway:** **Merge** is often recommended for team branches as it's non-destructive. **Rebase** is great for keeping your own feature branches clean before they are merged.

---

## 24. I Have Too Many Commits!
**Problem:** Your feature branch has many small, messy commits ("fix typo", "wip", "oops"). You want to present a single, clean commit in your PR.

**Solution:** Use **interactive rebase (`rebase -i`)** to squash your commits into one.

1.  **Start an interactive rebase session for your last `N` commits:**
    ```bash
    # Let's say you want to combine the last 5 commits
    git rebase -i HEAD~5
    ```
2.  **Your editor will open a file.** Change `pick` to `squash` (or `s`) for the commits you want to merge into the one above them.
    ```
    # Before
    pick f4e5d6c Add user dashboard component
    pick g7h8i9j Fix login validation rules
    pick a1b2c3d Adjust padding
    pick b3c4d5e Fix typo
    pick c5d6e7f Final tweaks

    # After
    pick f4e5d6c Add user dashboard component
    squash g7h8i9j Fix login validation rules
    squash a1b2c3d Adjust padding
    squash b3c4d5e Fix typo
    squash c5d6e7f Final tweaks
    ```
3.  **Save and close the file.** Git will then prompt you to write a new, single commit message for the combined commit.
4.  **Force push the squashed branch:**
    ```bash
    git push --force-with-lease
    ```
> **Takeaway:** Squashing makes your PRs much easier for reviewers to understand. Do it before you ask for a review.

---

## 25. I Committed to One Branch but Need the Same Changes in Another
**Problem:** You made a commit on the wrong branch, or you need to apply the same fix to multiple branches (e.g., `main` and `develop`).

**Solution:** Use `git cherry-pick` to grab a specific commit from one branch and apply it to another.

1.  **Find the hash of the commit you want to copy:**
    ```bash
    # On the branch where the commit currently exists
    git log
    # Copy the commit hash (e.g., a9b7c6d)
    ```
2.  **Switch to the destination branch and cherry-pick:**
    ```bash
    git checkout <destination-branch>
    git cherry-pick a9b7c6d
    ```
> **Takeaway:** `cherry-pick` is a precise tool for duplicating commits across branches without having to re-do the work.

---

## 26. I Merged My Changes… Then Realized They Shouldn’t Be There
**Problem:** A commit or a merged PR has been pushed to a shared branch (`main`, `develop`), but it introduced a bug or was released prematurely.

**Solution:** Use `git revert` to create a new commit that undoes the changes from a previous one. This is the safe way to "undo" on a public branch.

1.  **Find the hash of the commit to revert:**
    ```bash
    git log
    ```
2.  **Create the revert commit:**
    ```bash
    # For a regular commit
    git revert <commit-id-to-undo>

    # For a merge commit, you must specify a parent number (usually 1)
    # This tells Git which side of the merge to consider the "mainline"
    git revert -m 1 <merge-commit-id>
    ```
3.  **Push the new revert commit:**
    ```bash
    git push
    ```
> **Takeaway:** `revert` is non-destructive. It doesn't delete history; it adds to it, making it the correct and safe way to roll back changes on a shared branch.
