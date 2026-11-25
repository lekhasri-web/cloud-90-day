# Git Day 11: Branches, merge, PRs, .gitignore

## Concepts
- **Branch:** a separate line of commits. I can work without touching `main`, and `main` stays safe until I merge.
- **Merge:** combines one branch's commits into another. A branch stays after merging until I delete it.
- **Pull request (PR):** a proposal on GitHub to merge my branch into `main`. It adds review, CI checks, a comment thread and a permanent record. Review and CI are only required if the repo has branch protection rules.
- **Merge conflict:** both branches changed the same lines differently since they split, so Git can't choose. It's Git that asks me to decide, and it doesn't need GitHub.
- **Conflict markers:**
  - `<<<<<<< HEAD` to `=======`: the branch I'm on (here `main`)
  - `=======` to `>>>>>>> branch-two`: the incoming branch
  - Resolve by keeping one version, both, or writing a new one, then **delete the markers**.

## .gitignore and secrets
- `.gitignore` only affects files Git is **not tracking yet**. A file already committed stays tracked.
- To stop tracking a file but keep it on disk: `git rm --cached <file>`, then commit.
- If a secret reached a public repo, **revoke or rotate the key first**, because the old commit still holds it. Removing the file afterwards isn't enough.
- Add `.gitignore` before the first commit.
- Use `>>` to append to `.gitignore`. A single `>` overwrites the file.

## Good commit messages
- One short summary line, around 50 characters
- Imperative present tense: "Add", "Fix", "Remove"
- One logical change per commit
- Add a body explaining *why* when it isn't obvious

## Lab: what I ran
**PR to myself**
```
git switch -c practice-branch
git add git-practice.md        # by name, never "git add ."
git commit -m "Add git practice file"
git push -u origin practice-branch
```
Then I opened a PR on GitHub, merged it, and ran `git switch main` and `git pull`.

**Merge conflict on purpose**
```
git switch -c branch-one       # changed line 2, committed
git switch main
git switch -c branch-two       # changed line 2 differently, committed
git switch main
git merge branch-one           # clean
git merge branch-two           # CONFLICT
```
Resolve: edit the file, remove the markers, `git add <file>`, `git status`, then `git commit`.
`git merge --abort` backs out of a messy merge.

**.gitignore**
```
echo ".env" >> .gitignore
echo "*.pem" >> .gitignore
git check-ignore -v .env       # shows which line of .gitignore ignores it
```

**Cleanup:** `git branch -d` (refuses if a branch isn't merged), `git rm`, `git push`.

## Useful commands
- `git log --oneline --graph` shows the history, including where it split and rejoined
- `git remote -v` shows which GitHub repo I'm pushing to
- `ssh -T git@github.com` shows which GitHub account my key logs in as

## My mistakes today
- Local repo folder was `linux-labs` but the GitHub repo was `linux-lab`, so the push said "Repository not found". I found it with `git remote -v`, then fixed it with `git remote set-url`.
- Ran `git switch` in my home folder, which isn't a repo.
