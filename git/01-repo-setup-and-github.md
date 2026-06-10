# Repo Setup & Publishing to GitHub

> **Goal:** take a local project folder (a freshly created/downloaded app) and end up with it tracked by Git and published to GitHub — done correctly, from a clean terminal.

## Quick reference

```bash
# one-time per machine
git config --global user.name  "NAME"
git config --global user.email "email@gmail.com"
git config --global init.defaultBranch main

# per project: local folder -> GitHub
cd ~/projects/my-app
git init -b main
# ...write .gitignore...
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:USERNAME/my-app.git   # or use: gh repo create
git push -u origin main
```

The rest of this file explains each step and the auth choices.

## Steps

### 0. One-time machine setup (once per computer)

Install Git (`git --version` to verify; macOS: `brew install git`, or it ships with the Xcode Command Line Tools). Then identify yourself — every commit records an author name/email:

```bash
git config --global user.name  "NAME"
git config --global user.email "email@gmail.com"
git config --global init.defaultBranch main   # new repos start on 'main', not 'master'
git config --global pull.rebase false          # explicit default for `git pull` (merge)
```

`--global` writes to `~/.gitconfig` and applies to every repo. Without a configured name/email, your first commit either fails or is attributed to a guessed identity.

### 1. Initialize the repo

From inside the project folder:

```bash
cd ~/projects/my-app
git init -b main      # creates the hidden .git/ directory; current branch = main
git status            # everything shows as "Untracked"
```

`git init` only creates *local* version control. Nothing is on GitHub yet, and nothing is even committed — files are merely *seen* by Git as untracked.

### 2. Add a `.gitignore` FIRST

This is the step beginners skip and regret. Decide what should **never** be tracked *before* the first commit, because once a file is committed it stays in history even after you later delete it. Ignore: dependencies (`node_modules/`), build artifacts (`dist/`, `playwright-report/`, `test-results/`), local env/secrets (`.env`, `*.key`), and editor/OS noise (`.DS_Store`, `.idea/`).

A solid starting `.gitignore` for a Playwright + TypeScript project:

```gitignore
# Dependencies
node_modules/

# Build / TS output
dist/
build/
*.tsbuildinfo

# Playwright
playwright-report/
test-results/
blob-report/
/playwright/.cache/

# Env & secrets
.env
.env.*
*.pem
*.key

# Editor / OS
.DS_Store
.idea/
.vscode/
```

### 3. First commit (stage → commit)

```bash
git add .                       # stage everything not ignored
git status                      # confirm node_modules etc. are NOT listed
git commit -m "Initial commit"
```

`add` moves changes into the **staging area**; `commit` records a permanent snapshot in the local repo. Still nothing on GitHub.

### 4. Create the remote on GitHub

**Option A — GitHub CLI (fastest):**

```bash
gh auth login                              # one-time: authenticate the CLI
gh repo create my-app --private --source=. --remote=origin --push
```

`gh repo create` with `--source=.` creates the GitHub repo, wires up `origin`, and `--push` publishes in one shot. Drop `--push` if you want to push manually.

**Option B — GitHub web UI + manual remote:**

1. github.com → **New repository** → name it `my-app`. **Do not** initialize it with a README/.gitignore/license — you already have local commits, and an initialized remote creates a conflicting history you'd have to reconcile.
2. Copy the repo URL and wire it up:

```bash
git remote add origin git@github.com:USERNAME/my-app.git   # SSH
# or HTTPS:
git remote add origin https://github.com/USERNAME/my-app.git
git remote -v        # verify origin points where you expect
```

### 5. Publish

```bash
git push -u origin main
```

`-u` (`--set-upstream`) links your local `main` to `origin/main`. After this, `git push` and `git pull` need no arguments. Refresh the GitHub page — your code is live.

## Authentication — pick one

| Method | How it works | When to use |
|---|---|---|
| **HTTPS + PAT** | Clone/push over `https://`. When prompted for a password, paste a **Personal Access Token** (GitHub → Settings → Developer settings → Tokens), not your account password. A credential helper caches it. | Quick start; locked-down networks where SSH is blocked. |
| **SSH keys** | Generate a key (`ssh-keygen -t ed25519 -C "email"`), add the **public** key to GitHub → Settings → SSH keys. Use `git@github.com:...` URLs. | A machine you use regularly — no token prompts, no expiry juggling. |

Verify SSH once with `ssh -T git@github.com`. To switch an existing repo between methods, just change the URL: `git remote set-url origin <new-url>`. Account passwords haven't been accepted for Git operations for years — use a PAT or SSH.

## Handy snippets

The complete clean-folder → published flow (SSH, manual remote):

```bash
# one-time machine setup already done (user.name/email, defaultBranch)
cd ~/projects/my-app
git init -b main
printf "node_modules/\ndist/\n.env\n" > .gitignore   # real projects use the fuller list above
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:USERNAME/my-app.git
git push -u origin main
```

Recovering from "I committed `node_modules/` by mistake":

```bash
# add it to .gitignore first, then stop tracking it (keeps it on disk):
git rm -r --cached node_modules
git commit -m "Stop tracking node_modules"
git push
```

## Interview Q&A

**Q:** You created a repo on GitHub *with* a README, then tried to push your local commits and got `! [rejected] ... fetch first`. Why, and how do you fix it?

**A:** The remote already has a commit (the README) that your local history doesn't contain, so the push isn't a fast-forward. Fixes: either start the remote empty next time, or reconcile with `git pull --rebase origin main` to replay your commits on top of the remote one, then `git push`. (`--allow-unrelated-histories` is the blunt fallback if the two histories share no ancestor.)

**Q:** Why shouldn't you commit `node_modules/` or a `.env` file?

**A:** `node_modules/` is large, platform-specific, and fully reproducible from `package-lock.json`/`package.json` — committing it bloats the repo and creates noisy diffs. `.env` typically holds secrets (API keys, credentials); committing it leaks them into history permanently, even if you delete the file later. Both belong in `.gitignore`, and a leaked secret must be rotated, not just removed.

**Q:** What's the difference between `git init` and `git clone`?

**A:** `git init` creates a brand-new empty repository from an existing local folder (no remote, no history). `git clone <url>` copies an existing remote repository — full history plus a pre-configured `origin` remote and tracking branch. You `init` when starting fresh and publishing; you `clone` when joining an existing project.

**Q:** What does the `-u` in `git push -u origin main` actually do?

**A:** It sets the **upstream** (tracking) relationship between local `main` and `origin/main`. After that, bare `git push`/`git pull`/`git status` know which remote branch to talk to and can report ahead/behind counts. Without it you'd specify `origin main` every time.

**Q:** HTTPS or SSH for GitHub auth — how do you choose?

**A:** SSH for a machine I use regularly: set the key once, no prompts, no token expiry. HTTPS + a Personal Access Token when SSH ports are blocked (corporate networks) or for a quick one-off. Either way, account passwords aren't accepted for Git operations anymore.

## Gotchas & Anti-patterns

- **Initializing the GitHub repo with a README when you already have local commits** — creates divergent histories and a confusing first push. Create the remote empty, or reconcile with `pull --rebase`.
- **Committing before writing `.gitignore`** — `node_modules/`/secrets land in history; removing them later requires `git rm --cached` (and history rewriting if it's a secret).
- **Pushing a secret, then "deleting" it in a later commit** — it's still in history. You must rotate the secret and scrub history (`git filter-repo` / BFG).
- **Using your GitHub password at the push prompt** — fails; it expects a PAT (HTTPS) or you should be on SSH.
- **`master` vs `main` mismatch** — older Git defaults to `master`; set `init.defaultBranch main` so local and GitHub's default agree and `push -u origin main` doesn't create an unexpected branch.
- **Wrong remote URL / typo'd username** — `git remote -v` to verify; `git remote set-url origin <url>` to fix rather than removing and re-adding.

## Further Reading

- [Git — Getting Started: First-Time Setup](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [GitHub Docs — Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
- [GitHub Docs — About authentication to GitHub (PAT vs SSH)](https://docs.github.com/en/authentication)
- [GitHub CLI manual — `gh repo create`](https://cli.github.com/manual/gh_repo_create)
- [GitHub `gitignore` templates](https://github.com/github/gitignore)
