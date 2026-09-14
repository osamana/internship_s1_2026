# Week 1: Prep work

---

## Learning objectives

By Friday you can, without looking anything up:

1. Open a terminal, navigate the filesystem, chain commands with pipes, redirect output, and find any file or string on disk with `find`/`grep`/`rg` in under a minute.
2. Explain Git's object model (blob, tree, commit, ref) on a whiteboard and predict what `git status`, `git log --graph`, and `git reflog` will show before running them.
3. Create a branch, commit with a Conventional Commits message, rebase it onto an updated `main`, resolve a real merge conflict, and open a pull request that passes CI — end to end, in under 15 minutes.
4. Recover a "lost" commit with `git reflog` and find a bug-introducing commit with `git bisect`.
5. Bootstrap a Python project with `uv` (`uv init`, `uv add`, `uv sync`, `uv run`), with `ruff`, `pyright`, `pytest` and `pre-commit` wired up so that a badly formatted commit is rejected locally.
6. (Web interns) Bootstrap a Node project with `fnm` + `pnpm`, with ESLint and Prettier running from `package.json` scripts.
7. Set breakpoints and step through a Python program and a TypeScript program in VS Code using a hand-written `launch.json`.
8. Read an HTTP request/response on the wire and state the method, status class, content type, cookies and whether the request is idempotent; craft the same request with `curl` and `httpie`.
9. Write a `Dockerfile` and a `compose.yaml` that run a FastAPI app plus PostgreSQL with a healthcheck, and debug a failing container with `logs`, `exec` and `inspect`.
10. Explain processes, signals, exit codes, file permissions and the 12-factor "config in the environment" rule, and show that no secret has ever been committed to your repo.
11. Use an AI coding assistant to draft code and *then* explain every line of it to the supervisor.



## Why this week matters

A note from your supervisor: You already know more theory than most working developers. What you lack is the muscle memory that makes theory shippable: the 200 small tool interactions per day that either flow or fight you. Weeks 2–12 assume all of it. If in week 5 you lose an afternoon to a rebase gone wrong, or in week 9 your Docker build is 2 GB because you did not understand layers, that is time stolen from your capstone.

This week is deliberately dense and hands-on. Every module has a lab with acceptance criteria I will actually check. You will use the Git exercise in module 3 to make your first real pull request into the internship repo; that PR is graded (rubric in the *Assessment & Rubrics* document).

Two rules for the week:

- **Type every command yourself.** Do not paste from this document into a terminal until you have read what the command does. The point is fluency, not completion.
- **When something breaks, read the error message twice before searching.** Then search. Then ask. Bring me the error, what you tried, and what you expected — that is the format for every question for the next 12 weeks.

---



## 1. Machine setup



### Theory

A professional development machine has four properties: (a) a package manager so software installs are reproducible and upgradable, (b) a shell you have configured and understand, (c) SSH keys so you never type a password to Git, and (d) dotfiles under version control so the setup survives a laptop replacement.

**macOS.** Ships with `zsh` as the default shell and a usable Terminal.app. Install [Homebrew](https://brew.sh/) as the package manager; everything else (`git`, `uv`, `fnm`, `jq`, `httpie`, Docker Desktop) comes from it. Xcode Command Line Tools install automatically with Homebrew.

**Ubuntu / WSL2.** Use `apt` for system packages. On Windows, install WSL2 with Ubuntu (`wsl --install`), do *all* development inside the Linux filesystem (`~/projects`, not `/mnt/c/...` — the cross-filesystem I/O is 10–50× slower and breaks file watchers), and install Docker Desktop for Windows with the WSL2 backend. VS Code connects into WSL with the "WSL" extension. Change the default shell to zsh with `chsh -s $(which zsh)`.

**zsh configuration.** `~/.zshrc` runs on every interactive shell. Keep it small and readable. Oh My Zsh is optional; if you install it, only enable plugins you understand (`git`, `zsh-autosuggestions`, `zsh-syntax-highlighting`). A framework you cannot debug is a liability when your prompt breaks at 11 pm.

**Dotfiles.** `~/.zshrc`, `~/.gitconfig`, `~/.ssh/config`, `~/.config/` — these *are* your environment. Put them in a `dotfiles` repo and symlink them into place. When your laptop dies, `git clone` + one script restores everything.

**SSH keys.** Generate one `ed25519` key per machine with a passphrase. Register the public key with GitHub for authentication. Optionally use the same key for *commit signing* so your commits show "Verified" on GitHub — cheap and increasingly expected on protected branches.

**GitHub account hygiene.** Enable 2FA with an authenticator app (GitHub requires 2FA for contributors anyway), save the recovery codes in a password manager, set your primary email to be private if you wish and use the `@users.noreply.github.com` address in `git config user.email` so it matches. Never create a personal access token with more scope than you need; prefer SSH for Git and `gh auth login` for the GitHub CLI.

### Reading

- [Homebrew — install and basics](https://brew.sh/)
- [Install WSL on Windows (Microsoft)](https://learn.microsoft.com/en-us/windows/wsl/install)
- [GitHub Docs — Generating a new SSH key and adding it to the ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [GitHub Docs — Configuring two-factor authentication](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/configuring-two-factor-authentication)
- [GitHub Docs — About commit signature verification (SSH signing)](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
- [GitHub does dotfiles](https://dotfiles.github.io/) — patterns and example repos
- [Oh My Zsh](https://ohmyz.sh/) — optional
- [Missing Semester — Command-line Environment (dotfiles, aliases, SSH)](https://missing.csail.mit.edu/2020/command-line/)



### Videos

- **"Lecture 1: Course Overview + The Shell (2020)"** — Missing Semester (MIT), ~49 min. Watch the first 20 minutes tonight; focus on *how the shell finds programs* (`$PATH`) and how arguments are split. [youtube.com/watch?v=Z56Jmr9Z34Q](https://www.youtube.com/watch?v=Z56Jmr9Z34Q)



### Lab 1 — Set up the machine (1.5 h)

**macOS**

```bash
# 1. Homebrew (installs Xcode CLT as a side effect)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# follow the printed "Next steps" to add brew to your PATH, then:
brew --version

# 2. Core tools
brew install git gh jq httpie ripgrep fd tree wget
brew install --cask visual-studio-code docker
```

**Ubuntu 22.04+/WSL2**

```bash
sudo apt update && sudo apt install -y \
  build-essential curl wget git zsh jq httpie ripgrep fd-find tree unzip ca-certificates
# Debian names fd 'fdfind'; add an alias in ~/.zshrc: alias fd=fdfind
chsh -s "$(which zsh)"   # log out and back in afterwards

# GitHub CLI (official apt repo): https://github.com/cli/cli/blob/trunk/docs/install_linux.md
# Docker Engine: https://docs.docker.com/engine/install/ubuntu/  then the post-install step
#   https://docs.docker.com/engine/install/linux-postinstall/  (adds your user to the docker group)
# On WSL2, install Docker Desktop for Windows with the WSL2 backend instead of Docker Engine.
```

**Both platforms**

```bash
# 3. Git identity (use your GitHub noreply address if your email is private)
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global rebase.autoStash true
git config --global push.autoSetupRemote true
git config --global fetch.prune true
git config --global diff.algorithm histogram
git config --global rerere.enabled true
git config --global core.editor "code --wait"

# 4. SSH key for GitHub
ssh-keygen -t ed25519 -C "you@example.com"        # accept the default path, set a passphrase
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519                          # macOS: ssh-add --apple-use-keychain ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub                          # paste into GitHub → Settings → SSH and GPG keys → New SSH key (type: Authentication)
ssh -T git@github.com                              # expect: "Hi <user>! You've successfully authenticated..."

# 5. (Optional but recommended) sign commits with the same key
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
# then add the same public key on GitHub again, this time with type: Signing

# 6. GitHub CLI
gh auth login   # choose GitHub.com → SSH → upload the key if asked → login with a web browser

# 7. Dotfiles repo
mkdir -p ~/projects/dotfiles && cd ~/projects/dotfiles && git init
cp ~/.zshrc zshrc && cp ~/.gitconfig gitconfig
ln -sf ~/projects/dotfiles/zshrc ~/.zshrc
ln -sf ~/projects/dotfiles/gitconfig ~/.gitconfig
git add . && git commit -m "chore: initial dotfiles"
gh repo create dotfiles --private --source=. --push
```

macOS only — make the agent remember your passphrase in Keychain by creating `~/.ssh/config`:

```
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

**Acceptance criteria** (supervisor checks on Wednesday):

- [ ] `ssh -T git@github.com` greets you by username.
- [ ] `git config --list --show-origin | grep -E "user\.|gpgsign"` shows name, email and (optionally) signing.
- [ ] GitHub → Settings → Password and authentication shows 2FA enabled with a TOTP app.
- [ ] A private `dotfiles` repo exists with at least `zshrc` and `gitconfig`, and `ls -la ~ | grep '\->'` shows the symlinks.
- [ ] `brew doctor` (macOS) prints "Your system is ready to brew" or only warnings you can explain.
- [ ] `docker run --rm hello-world` prints the hello message without `sudo` (Linux: after the post-install step and a re-login).



### Pitfalls

- **WSL: cloning into** `/mnt/c`**.** Slow and breaks Next.js/uvicorn file watching. Use `~/projects` inside WSL.
- **Two Pythons, two Gits.** macOS ships an old Apple Git and `python3` shim; Homebrew installs newer ones. `which -a git python3` shows every candidate on `$PATH`; the first one wins. If `git --version` is not the Homebrew one, your `PATH` order in `~/.zshrc` is wrong.
- **Keys without passphrases.** A stolen laptop then equals your GitHub identity. Use a passphrase and let the agent cache it.
- **Recovery codes in a Notes app.** Store them in a password manager. Losing 2FA on GitHub is a multi-day support process.

---



## 2. Shell fluency



### Theory

The shell is a programming language whose values are text streams and whose functions are programs. Three ideas make it powerful:

1. **Everything is a file descriptor.** Each process has stdin (0), stdout (1), stderr (2). `>` redirects stdout, `2>` stderr, `2>&1` merges them, `<` feeds a file to stdin. `|` connects one process's stdout to the next's stdin. This is why `grep ERROR app.log | wc -l` works without either tool knowing the other exists.
2. **Small tools, composed.** `grep` (filter lines by pattern), `sed` (edit lines), `awk` (columns and light computation), `sort`/`uniq -c` (histograms), `cut`, `xargs` (turn stdin into arguments), `find`/`fd` (walk the tree). You will do 90% of log analysis with `grep | awk | sort | uniq -c | sort -rn | head`.
3. **Exit codes and control flow.** Every command returns 0 (success) or non-zero (failure); `echo $?` prints the last one. `a && b` runs `b` only if `a` succeeded; `a || b` only if it failed. CI systems and Docker healthchecks use nothing but exit codes.

Environment variables are inherited by child processes (`export FOO=bar` then `python -c 'import os; print(os.environ["FOO"])'`), which is exactly how configuration reaches your app in Docker and on Dokploy. `.env` files are a *convenience for local development*: a list of `KEY=value` lines that a tool (`uv run --env-file`, `docker compose`, `pydantic-settings`) loads into the environment. They are never committed.

Ports: a server binds a port; only one process can bind a given port. When `uvicorn` says "address already in use", `lsof -i :8000` tells you who holds it and `kill <pid>` frees it.

JSON on the command line: `jq` is `awk` for JSON. `curl -s https://api.github.com/repos/astral-sh/uv | jq '.stargazers_count'` is a one-liner you will type variants of a hundred times. `httpie` (`http`) is `curl` with sane defaults for JSON APIs: automatic JSON encoding of `key=value` args, coloured output, session cookies.

Quoting rules you must internalize: double quotes expand `$VAR` and `$(cmd)`; single quotes are literal; unquoted variables split on whitespace (`rm -rf $DIR/` with an empty `DIR` is the classic disaster — always `"$DIR"`).

### Reading

- [Missing Semester — Shell Tools and Scripting](https://missing.csail.mit.edu/2020/shell-tools/)
- [Missing Semester — Data Wrangling (grep/sed/awk pipelines)](https://missing.csail.mit.edu/2020/data-wrangling/)
- [Julia Evans — Bite Size Command Line (zine announcement; the zine is worth the price)](https://jvns.ca/blog/2018/08/05/new-zine--bite-size-command-line/)
- [jq manual](https://jqlang.github.io/jq/manual/) and [jq tutorial](https://jqlang.org/tutorial/)
- [HTTPie CLI — Usage](https://httpie.io/docs/cli/usage)
- [The Twelve-Factor App — III. Config](https://12factor.net/config)
- [hstr — shell history search (optional tool)](https://github.com/dvorka/hstr)



### Videos

- **"Lecture 1: Course Overview + The Shell (2020)"** — Missing Semester, ~49 min, finish it. Focus on redirection and the `$PATH` demo. [youtube.com/watch?v=Z56Jmr9Z34Q](https://www.youtube.com/watch?v=Z56Jmr9Z34Q)



### Lab 2 — Shell drills (1.5 h)

Work in `~/projects/week1-shell`. Each step has an expected output you must be able to explain.

```bash
mkdir -p ~/projects/week1-shell && cd ~/projects/week1-shell

# 1. Generate a fake access log (10k lines) to practice on
python3 - <<'EOF'
import random, datetime
paths=["/","/api/users","/api/orders","/login","/health","/api/items/42"]
codes=[200]*70+[201]*5+[301]*3+[404]*12+[500]*7+[502]*3
with open("access.log","w") as f:
    for i in range(10000):
        ts=(datetime.datetime(2026,9,14)+datetime.timedelta(seconds=i*7)).isoformat()
        f.write(f'{ts} 10.0.{random.randint(0,3)}.{random.randint(1,254)} "{random.choice(["GET","POST","PUT","DELETE"])} {random.choice(paths)}" {random.choice(codes)} {random.randint(2,900)}ms\n')
EOF

# 2. Navigation and inspection
pwd; ls -lah; file access.log; wc -l access.log; head -3 access.log; tail -n 2 access.log

# 3. Filter, count, histogram
grep ' 500 ' access.log | wc -l
awk '{print $5}' access.log | sort | uniq -c | sort -rn          # status code histogram
awk '$5 >= 500 {print $2}' access.log | sort | uniq -c | sort -rn | head -5   # top IPs causing 5xx
grep -c '"POST' access.log

# 4. sed: rewrite in place safely (write to a new file, then compare)
sed 's#/api/#/v1/#g' access.log > access.v1.log && diff <(head -2 access.log) <(head -2 access.v1.log)

# 5. find / fd / ripgrep
find . -name '*.log' -size +100k
rg -c ' 404 ' .                        # ripgrep counts per file
fd -e log                              # (fdfind on Debian/Ubuntu)

# 6. Redirection and exit codes
grep ' 999 ' access.log; echo "exit=$?"                   # 1 = no match
grep ' 999 ' access.log > /dev/null 2>&1 || echo "no 999s, as expected"
ls /nonexistent 2> errors.txt; cat errors.txt

# 7. Processes and ports
python3 -m http.server 8123 &          # background job; note the PID
lsof -i :8123 -P -n                    # who owns the port
jobs; kill %1; sleep 1; lsof -i :8123 || echo "port free"

# 8. Environment and .env
export APP_ENV=dev
python3 -c 'import os; print(os.environ.get("APP_ENV"))'
printf 'DATABASE_URL=postgresql://app:secret@localhost:5432/app\nAPP_ENV=dev\n' > .env
printf '.env\n*.log\n' > .gitignore
set -a; source .env; set +a; echo "$DATABASE_URL"     # one way to load a .env into the current shell

# 9. jq and HTTP clients
curl -s https://api.github.com/repos/astral-sh/uv | jq '{name, stars: .stargazers_count, lang: .language}'
http GET https://httpbin.org/get X-Demo:1 q==hello          # header and query param syntax
http --print=HhBb POST https://httpbin.org/post name=osama age:=42  # string vs raw JSON value
```

**Acceptance criteria**

- [ ] You can state, from memory, the 5xx percentage of `access.log` and the command that produced it.
- [ ] `git status` in this directory shows `.env` and `*.log` as ignored (`git init` first if needed; `git check-ignore -v .env` names the rule).
- [ ] You can explain the difference between `>`, `>>`, `2>`, `2>&1`, `|`, and `$(...)` in one sentence each.
- [ ] You can free a port held by a process without restarting the machine.
- [ ] You can write a `jq` filter that extracts one nested field and one array length from any JSON response the supervisor gives you.



### Pitfalls

- `rm -rf` with an unquoted or empty variable. Always `"$var"`; consider `set -u` in scripts.
- `sed -i` differs between GNU (`sed -i 's/a/b/' f`) and BSD/macOS (`sed -i '' 's/a/b/' f`). Write to a new file, or install `gnu-sed`.
- `grep` regex flavours: use `grep -E` for extended syntax or just `rg`.
- Running `python3 -m http.server &` and forgetting it — `jobs`, `fg`, `kill %1`. Orphaned servers are why "port already in use" happens.
- Loading `.env` with `source` executes it as shell — a value with spaces or `$` breaks. Prefer tool-native loaders (`uv run --env-file .env`, Compose `env_file:`).

---



## 3. Git, deeply



### Theory

**The model, not the commands.** Git is a content-addressed database plus a set of pointers.

- A **blob** is file content, named by the SHA-1 (SHA-256 in newer repos) of that content.
- A **tree** is a directory listing: names → blobs/trees, with modes.
- A **commit** is a snapshot: one root tree + parent commit(s) + author/committer + message. Commits form a DAG; a commit's hash covers its parents, so history cannot be altered without every descendant hash changing.
- A **ref** is a named pointer to a commit: `refs/heads/main` (a branch), `refs/tags/v1.0`, `refs/remotes/origin/main` (your last-known position of the remote's branch). `HEAD` points to the current branch (or directly to a commit when "detached").

Three areas: the **working tree** (files on disk), the **index/staging area** (the exact tree that will become the next commit), the **repository** (`.git/objects`, refs). `git add` copies working-tree content into the index; `git commit` turns the index into a tree + commit and moves the current branch pointer. `git add -p` stages hunks, which is how you make small, reviewable commits from messy work.

**Branches are cheap** because a branch is a 41-byte file containing a hash. Creating one copies nothing.

**Merge vs rebase.** Both integrate `feature` with `main`.

- `git merge main` (on `feature`) creates a *merge commit* with two parents. History is truthful but noisy.
- `git rebase main` (on `feature`) *re-applies* each of your commits on top of `main`'s tip, creating new commits with new hashes; the old ones become unreachable (still in reflog). History is linear and each commit is reviewable in isolation. The cost: you rewrote history, so if the branch was already pushed you must `git push --force-with-lease` — never `--force`, which can overwrite a teammate's push. **Rule in this internship:** rebase *your own unmerged feature branches*; never rebase `main` or anything someone else has based work on.

**Interactive rebase** (`git rebase -i main`) lets you reorder, squash (`s`), reword (`r`), edit (`e`), or drop commits before review. A PR of 3 clean commits reviews faster than one of 17 "wip" commits. `git commit --fixup <sha>` + `git rebase -i --autosquash` is the professional way to fix an earlier commit.

**Conflicts** occur when both sides changed the same lines. Git writes both versions between `<<<<<<<`, `=======`, `>>>>>>>` markers. Resolving means: understand *both intents*, produce the correct combined code (often neither side verbatim), remove the markers, `git add` the file, then `git rebase --continue` (or `git merge --continue`). `git diff` while conflicted shows a combined diff; `git checkout --ours/--theirs <file>` takes one side wholesale (note that during a *rebase* "ours" is the branch you are rebasing *onto*, which surprises everyone once). `rerere` records resolutions so repeated conflicts auto-resolve.

**Stash** parks uncommitted changes (`git stash push -m "msg"`, `git stash list`, `git stash pop`). Use it to switch branches quickly; do not use it as long-term storage — a stash is not on any branch and is easily forgotten. Prefer a WIP commit on a branch.

**Bisect** binary-searches history for the commit that introduced a bug: `git bisect start; git bisect bad; git bisect good v1.2` then test and mark until it names the commit. With a test script: `git bisect run pytest tests/test_x.py -q`. 1,000 commits take ~10 steps.

**Reflog** is your undo log: every time `HEAD` or a branch moves, Git records it (`git reflog`, kept ~90 days). A "lost" commit after a bad rebase or `reset --hard` is always in the reflog: `git reset --hard HEAD@{3}` or `git branch rescue <sha>`. Objects that are unreachable from any ref *or reflog* get garbage-collected eventually; committed work is essentially never lost within a couple of weeks.

`.gitignore` patterns exclude untracked files. It does *not* remove already-tracked files — `git rm --cached <file>` does. Generate a base from [github/gitignore](https://github.com/github/gitignore) (`Python.gitignore`, `Node.gitignore`) and add `.env`, `.venv/`, `node_modules/`, `.DS_Store`, `*.log`, `.idea/`, `.vscode/`* (except `launch.json`/`settings.json` if the team shares them).

**Conventional Commits.** `type(scope): imperative summary` with `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`. Body explains *why*; footer references issues (`Closes #12`) and breaking changes (`BREAKING CHANGE:`). Tools generate changelogs and version bumps from this; reviewers scan history faster. Summary line ≤ 72 chars, imperative mood ("add", not "added"), no trailing period.

**GitHub flow.** `main` is always deployable and protected. Every change: branch from `main` (`feat/12-user-login`), commit, push, open a PR, get review + green CI, squash-or-rebase-merge, delete the branch. Small PRs (< 400 lines changed) get reviewed same day; big ones rot.

**Code review etiquette** (both directions):

- Author: self-review the diff first; write a description that states the *what*, the *why*, how to test it, and screenshots for UI; keep it focused; respond to every comment (fix, or explain why not); never take it personally.
- Reviewer: review within 24 h; prefix comments with intent (`nit:`, `question:`, `blocking:`); explain *why*; approve when the code is better than what exists, not when it is perfect; praise good things specifically.

**Protected branches.** On `main`: require a PR, require ≥1 approval, require status checks (lint, tests) to pass, require linear history or squash merges, block force-pushes and deletions. The supervisor sets this on the internship repo; you will set it on your capstone repo in week 4.

### Reading

- [Pro Git — 10.2 Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) — the 20 minutes that make Git stop being magic
- [Julia Evans — Inside .git](https://jvns.ca/blog/2024/01/26/inside-git/) and [git branches: intuition & reality](https://jvns.ca/blog/2023/11/23/branches-intuition-reality/)
- [Pro Git — 3.2 Basic Branching and Merging (includes conflicts)](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Pro Git — 3.6 Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) and [7.6 Rewriting History (interactive rebase)](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History)
- [Pro Git — 7.3 Stashing and Cleaning](https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning), [7.10 Debugging with Git (bisect)](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git), [10.7 Maintenance and Data Recovery (reflog)](https://git-scm.com/book/en/v2/Git-Internals-Maintenance-and-Data-Recovery)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)
- [GitHub Docs — GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow), [About pull request reviews](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews), [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Google Engineering Practices — How to do a code review](https://google.github.io/eng-practices/review/reviewer/) and [The CL author's guide](https://google.github.io/eng-practices/review/developer/)
- [Julia Evans — Popular git config options](https://jvns.ca/blog/2024/02/16/popular-git-config-options/) and [Confusing git terminology](https://jvns.ca/blog/2023/11/01/confusing-git-terminology/)
- Interactive: [Learn Git Branching](https://learngitbranching.js.org/?locale=en_US) — do "Main: Introduction Sequence" and "Ramping Up", plus "Remote: Push & Pull — Git Remotes!" levels 1–8



### Videos

- **"How Git Works: Explained in 4 Minutes"** — ByteByteGo, 4 min. Watch first for the three-areas picture. [youtube.com/watch?v=e9lnsKot_SQ](https://www.youtube.com/watch?v=e9lnsKot_SQ)
- **"Git Internals by John Britton of GitHub — CS50 Tech Talk"** — CS50, ~1 h. Focus on the live `.git/objects` walkthrough; skip Q&A. [youtube.com/watch?v=lG90LZotrpo](https://www.youtube.com/watch?v=lG90LZotrpo)
- **"Lecture 6: Version Control (git) (2020)"** — Missing Semester, ~1 h 25 min. The data-model-first explanation this module follows; watch at 1.5×. [youtube.com/watch?v=2sjqTHE0zok](https://www.youtube.com/watch?v=2sjqTHE0zok)



### Lab 3 — Graded Git exercise (3 h)

Part A builds intuition in a throwaway repo. Part B is the deliberately conflicting scenario, graded. Part C is your first real PR.

**Part A — objects and refs (30 min)**

```bash
mkdir -p ~/projects/git-lab && cd ~/projects/git-lab && git init
echo "hello" > a.txt && git add a.txt
git cat-file -p "$(git write-tree)"          # the tree the index would commit
git commit -m "feat: add a.txt"
git log --oneline
git cat-file -p HEAD                          # commit object: tree, author, message
git cat-file -p HEAD^{tree}                   # tree → blob
cat .git/HEAD; cat .git/refs/heads/main       # refs are files containing hashes
git switch -c feat/b && echo "b" > b.txt && git add b.txt && git commit -m "feat: add b.txt"
git log --oneline --graph --all --decorate
git switch main && git merge feat/b           # fast-forward: main just moves
git reflog                                    # every HEAD movement is logged
```

**Part B — the conflict scenario (graded, 90 min)**

Simulate two developers (you and "Sara") working on the same file. Do it exactly as written; the point is the conflict.

```bash
cd ~/projects && rm -rf git-conflict && mkdir git-conflict && cd git-conflict && git init
cat > pricing.py <<'EOF'
def price(items: list[float]) -> float:
    total = sum(items)
    return total


def format_price(value: float) -> str:
    return f"${value:.2f}"
EOF
git add pricing.py && git commit -m "feat(pricing): add price and format_price"

# --- Sara's work lands on main first: she adds tax to price()
git switch -c sara/tax
cat > pricing.py <<'EOF'
TAX_RATE = 0.16


def price(items: list[float]) -> float:
    subtotal = sum(items)
    return subtotal * (1 + TAX_RATE)


def format_price(value: float) -> str:
    return f"${value:.2f}"
EOF
git commit -am "feat(pricing): apply 16% VAT in price()"
git switch main && git merge --ff-only sara/tax

# --- Your branch started from the ORIGINAL commit and adds a discount to price()
git switch -c feat/discount HEAD~1
cat > pricing.py <<'EOF'
def price(items: list[float], discount: float = 0.0) -> float:
    total = sum(items)
    return total * (1 - discount)


def format_price(value: float) -> str:
    return f"${value:.2f}"
EOF
git commit -am "feat(pricing): support percentage discount in price()"
echo "def test_smoke(): assert True" > test_pricing.py
git add test_pricing.py && git commit -m "test(pricing): add smoke test"

git log --oneline --graph --all         # you should see two diverged lines from the base commit
```

Now integrate your branch onto the new `main` with **rebase** and resolve the conflict so that `price()` applies the discount *and then* the tax, keeps `TAX_RATE`, and the signature keeps the `discount` parameter.

```bash
git rebase main
# CONFLICT (content): Merge conflict in pricing.py
git status                                # "both modified: pricing.py"
git diff                                  # combined diff with <<<<<<< ======= >>>>>>>
code pricing.py                           # resolve by hand; the correct result is NEITHER side verbatim
```

The correct resolution:

```python
TAX_RATE = 0.16


def price(items: list[float], discount: float = 0.0) -> float:
    subtotal = sum(items) * (1 - discount)
    return subtotal * (1 + TAX_RATE)


def format_price(value: float) -> str:
    return f"${value:.2f}"
```

```bash
python3 -c 'from pricing import price; assert round(price([100], 0.1), 2) == 104.4, price([100], 0.1)'
git add pricing.py && git rebase --continue      # editor opens for the commit message; keep it
git log --oneline --graph --all                  # linear: base → sara → your two commits
```

Then practise the rest of the toolbox on this repo:

```bash
# interactive rebase: squash your two commits into one, reword the message
git rebase -i main            # mark the second commit 's' (squash); write: "feat(pricing): support percentage discount"
git log --oneline             # exactly 3 commits: base, sara's tax commit, your single squashed commit

# reflog recovery: "accidentally" destroy your work, then get it back
git reset --hard main
git log --oneline             # your commit is gone
git reflog                    # find the line "rebase (finish)" or your commit message
git reset --hard HEAD@{1}     # or: git reset --hard <sha>
git log --oneline             # it is back

# stash
echo "# TODO" >> pricing.py && git stash push -m "wip note"
git stash list && git stash pop && git restore pricing.py

# bisect: plant a bug 5 commits deep, then find it
for i in 1 2 3 4 5 6 7 8; do echo "line $i" >> notes.txt; if [ "$i" = 5 ]; then sed -i.bak 's/TAX_RATE = 0.16/TAX_RATE = 1.16/' pricing.py && rm -f pricing.py.bak; fi; git add -A; git commit -qm "chore: note $i"; done
cat > check.sh <<'EOF'
#!/bin/sh
python3 -c 'from pricing import price; import sys; sys.exit(0 if round(price([100]),2)==116.0 else 1)'
EOF
chmod +x check.sh
git bisect start HEAD HEAD~8
git bisect run ./check.sh        # prints "<sha> is the first bad commit" → chore: note 5
git bisect reset
```

**Part C — your first real PR (60 min)**

1. `git clone git@github.com:<supervisor-org>/internship_2026s1.git` (URL given at kickoff). Read the *Engineering Practices* document first.
2. Branch: `git switch -c chore/<yourhandle>-week1-intro`.
3. Add `interns/<yourhandle>/README.md` with: your track, machine (OS, shell), links to your dotfiles repo, and a 5-line "what I found hardest this week". Add `interns/<yourhandle>/git-lab/` containing `pricing.py` and `check.sh` from Part B (not the `.git` directory).
4. Commit with a Conventional Commit message; push with `git push` (autoSetupRemote makes this work first time); open the PR with `gh pr create --fill --base main` and edit the description to follow the PR template.
5. Ask the other intern for a review (assign them on GitHub); review *their* PR the same day with at least one `question:` and one `nit:` comment.
6. Address review, then the supervisor merges.

**Acceptance criteria (graded — rubric in the *Assessment & Rubrics* document)**

- [ ] Part B repo: `git log --oneline --graph` is linear, contains no merge commits, and `pricing.py` at `HEAD` passes both `python3 -c 'from pricing import price; assert round(price([100],0.1),2)==104.4'` and `./check.sh` after `git bisect reset`.
- [ ] You can show, in `git reflog`, the entries for the rebase, the destructive reset and the recovery.
- [ ] `git bisect run` output identifies "chore: note 5".
- [ ] Part C PR: title and every commit follow Conventional Commits; description has What/Why/How-to-test; CI (lint) is green; at least one review round happened; branch deleted after merge.
- [ ] In the Wednesday 1:1 you whiteboard blob/tree/commit/ref and answer "what does `git rebase` do to commit hashes and why".



### Pitfalls

- `git push --force` on a shared branch. Use `--force-with-lease`, and only on your own feature branches.
- Resolving a conflict by deleting the markers and keeping "your" side without reading the other side. Every conflict is a design question: what did *both* people want?
- `git pull` creating surprise merge commits. `pull.rebase true` (set in Lab 1) makes `pull` rebase instead.
- Committing `.env`, `node_modules/`, `.venv/`, `__pycache__/`. Check `git status` before every commit; if a secret was ever committed, *rotate the secret* — deleting the file in a later commit does not remove it from history.
- Stashing and forgetting. `git stash list` weekly; prefer WIP commits on branches.
- Huge PRs. If your diff exceeds ~400 lines, split it. Reviewers approve big PRs without reading them, which is worse than no review.

---



## 4. Python and Node toolchains



### Theory

**Python.** In 2026 the toolchain is [uv](https://docs.astral.sh/uv/): one binary that installs Python interpreters, creates virtual environments, resolves and locks dependencies, and runs scripts and tools. It replaces `pyenv` + `pip` + `venv` + `pip-tools` + `pipx` (you may still see `pyenv` in older projects; know what it is, do not install it).

Key concepts:

- `pyproject.toml` is the single project manifest (PEP 621): name, version, `requires-python`, `dependencies`, and tool config sections (`[tool.ruff]`, `[tool.pytest.ini_options]`, `[tool.pyright]`).
- `uv.lock` is the cross-platform lockfile with *exact* resolved versions and hashes. Commit it. `uv sync` makes `.venv/` match it exactly (removing extras). Reproducibility on the Dokploy server depends on this.
- **Dependency groups**: `uv add --dev pytest ruff pyright` puts tooling into `[dependency-groups] dev` so production images can `uv sync --no-dev`.
- `uv run <cmd>` runs a command inside the project's environment, syncing first if needed. You almost never activate the venv by hand.
- `uvx <tool>` / `uv tool install <tool>` run CLI tools in isolated environments (like `pipx`).
- `uv python install 3.13` downloads a managed interpreter; `uv python pin 3.13` writes `.python-version` so everyone on the project gets the same one.

Quality tooling:

- **ruff** — linter *and* formatter, replaces flake8 + isort + black. `ruff check --fix` and `ruff format`. Configure rule sets in `pyproject.toml`; start with `E, F, I, UP, B, SIM` and add more later.
- **pyright** (or mypy) — static type checker. Type hints are documentation the machine verifies; FastAPI, Pydantic, SQLAlchemy 2 and every LLM SDK are heavily typed, so a checker catches a whole class of bugs before running anything. Start in `basic` mode; `strict` later per package.
- **pytest** — test runner. `test_*.py` files, `test_`* functions, plain `assert`, fixtures for setup. `uv run pytest -q`.
- **pre-commit** — runs hooks on staged files before each commit (ruff, trailing whitespace, YAML validity, `uv lock` consistency). It makes "CI failed on formatting" impossible.

**Node (web interns required; AI intern: install fnm + pnpm, skip ESLint/Prettier until needed).**

- **fnm** manages Node versions per project via `.node-version` / `.nvmrc`; install the current LTS (24.x).
- **pnpm** is the package manager: strict `node_modules` (no phantom dependencies), content-addressed store (fast, disk-light), `pnpm-lock.yaml` committed. Commands: `pnpm install`, `pnpm add <pkg>`, `pnpm add -D <pkg>`, `pnpm run <script>` (or just `pnpm <script>`), `pnpm dlx <pkg>` (like `npx`).
- `package.json` **scripts** are the project's task runner: `dev`, `build`, `lint`, `format`, `typecheck`, `test`. CI and Dokploy call these; never rely on a command that exists only in your shell history.
- **ESLint** finds bugs and enforces rules; **Prettier** formats. They are separate tools with separate jobs; use `eslint-config-prettier` so they do not fight. Next.js scaffolds ESLint for you in week 6; this week you only need to know how to run them.



### Reading

- [uv — Working on projects](https://docs.astral.sh/uv/guides/projects/), [Installing and managing Python](https://docs.astral.sh/uv/guides/install-python/), [Locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/), [Using tools (uvx)](https://docs.astral.sh/uv/guides/tools/)
- [Ruff — Tutorial](https://docs.astral.sh/ruff/tutorial/) and [Configuring Ruff](https://docs.astral.sh/ruff/configuration/)
- [Pyright — Getting started](https://github.com/microsoft/pyright/blob/main/docs/getting-started.md) (or [mypy — Getting started](https://mypy.readthedocs.io/en/stable/getting_started.html))
- [pytest — Get Started](https://docs.pytest.org/en/stable/getting-started.html)
- [pre-commit](https://pre-commit.com/), [ruff-pre-commit](https://github.com/astral-sh/ruff-pre-commit), [uv-pre-commit](https://github.com/astral-sh/uv-pre-commit)
- [fnm — README](https://github.com/Schniz/fnm), [pnpm — Installation](https://pnpm.io/installation), [npm docs — scripts](https://docs.npmjs.com/cli/v10/using-npm/scripts)
- [ESLint — Getting started](https://eslint.org/docs/latest/use/getting-started), [Prettier — Install](https://prettier.io/docs/install)



### Videos

- **"Python Tutorial: UV — A Faster, All-in-One Package Manager to Replace Pip and Venv"** — Corey Schafer, ~27 min. Focus on `uv init/add/sync/run` and the lockfile; ignore the pip-compat section. [youtube.com/watch?v=AMdG7IjgSPM](https://www.youtube.com/watch?v=AMdG7IjgSPM)



### Lab 4 — Toolchains (2 h)

**Python (everyone)**

```bash
# 1. Install uv (macOS/Linux/WSL)
curl -LsSf https://astral.sh/uv/install.sh | sh
exec $SHELL -l && uv --version       # 0.12.x or newer

# 2. Python interpreter (managed by uv; you do NOT need pyenv)
uv python install 3.13
uv python list --only-installed

# 3. Project
mkdir -p ~/projects/week1-py && cd ~/projects/week1-py
uv init --name week1py --python 3.13      # creates pyproject.toml, .python-version, main.py, README.md, .gitignore
git init && git add -A && git commit -qm "chore: scaffold with uv"
uv add "fastapi[standard]" pydantic-settings
uv add --dev pytest ruff pyright pre-commit
cat pyproject.toml; ls -la .venv; head -20 uv.lock
```

Append tool configuration to `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py313"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "N"]

[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "basic"
venvPath = "."
venv = ".venv"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

Write a tiny app and test:

```bash
mkdir -p app tests
cat > app/__init__.py <<'EOF'
EOF
cat > app/settings.py <<'EOF'
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_env: str = "dev"
    database_url: str = "postgresql://app:app@localhost:5432/app"
EOF
cat > app/main.py <<'EOF'
from fastapi import FastAPI

from app.settings import Settings

settings = Settings()
app = FastAPI(title="week1")


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok", "env": settings.app_env}


def slugify(text: str) -> str:
    return "-".join(text.lower().split())
EOF
cat > tests/test_main.py <<'EOF'
from fastapi.testclient import TestClient

from app.main import app, slugify


def test_slugify() -> None:
    assert slugify("Hello  World") == "hello-world"


def test_health() -> None:
    client = TestClient(app)
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"
EOF
rm -f main.py
uv run pytest
uv run ruff check . && uv run ruff format --check .
uv run pyright
uv run fastapi dev app/main.py     # open http://127.0.0.1:8000/docs, then Ctrl+C
```

pre-commit:

```bash
cat > .pre-commit-config.yaml <<'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.7
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.12.13
    hooks:
      - id: uv-lock
EOF
uv run pre-commit install
uv run pre-commit run --all-files
# Prove it bites: introduce an unused import and try to commit
printf 'import os\n' >> app/main.py
git add -A && git commit -m "test: should fail"      # ruff rejects; the hook auto-fixes; re-add and commit properly
git add -A && git commit -m "feat: week1 scaffold with tests and hooks"
```

**Node (web interns; AI intern does steps 1–2 only)**

```bash
# 1. fnm + Node LTS
curl -fsSL https://fnm.vercel.app/install | bash     # or: brew install fnm
# add to ~/.zshrc (the installer prints the exact line): eval "$(fnm env --use-on-cd --shell zsh)"
exec $SHELL -l
fnm install --lts && fnm default lts-latest && node --version    # v24.x

# 2. pnpm via Corepack (bundled with Node) or the standalone installer
corepack enable && corepack prepare pnpm@latest --activate
pnpm --version     # 10.x

# 3. A minimal TypeScript project with scripts, ESLint and Prettier
mkdir -p ~/projects/week1-ts && cd ~/projects/week1-ts && git init
echo "24" > .node-version
pnpm init
pnpm add -D typescript tsx @types/node eslint @eslint/js typescript-eslint prettier eslint-config-prettier
pnpm exec tsc --init --rootDir src --outDir dist --strict --module nodenext --target es2022
mkdir -p src
cat > src/index.ts <<'EOF'
export function slugify(text: string): string {
  return text.toLowerCase().split(/\s+/).filter(Boolean).join("-");
}

console.log(slugify("Hello  World"));
EOF
cat > eslint.config.js <<'EOF'
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import prettier from "eslint-config-prettier";

export default tseslint.config(js.configs.recommended, ...tseslint.configs.recommended, prettier);
EOF
printf '{ "semi": true, "singleQuote": false, "printWidth": 100 }\n' > .prettierrc
printf 'node_modules/\ndist/\n.env\n' > .gitignore
```

Set `"type": "module"` and the scripts in `package.json`:

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx src/index.ts",
    "build": "tsc",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

```bash
pnpm dev && pnpm lint && pnpm typecheck && pnpm format:check
git add -A && git commit -m "chore: scaffold TypeScript project with eslint and prettier"
```

**Acceptance criteria**

- [ ] `uv run pytest` passes 2 tests; `uv run ruff check .`, `uv run ruff format --check .` and `uv run pyright` all exit 0.
- [ ] `git log` shows the commit with the unused import was *rejected* by pre-commit (show the terminal history or explain the sequence).
- [ ] `uv.lock` and `.python-version` are committed; `.venv/` is not.
- [ ] You can explain the difference between `pyproject.toml` `dependencies` and `uv.lock`, and why the server uses `uv sync --frozen --no-dev`.
- [ ] (Web) `pnpm lint`, `pnpm typecheck`, `pnpm format:check` exit 0; `pnpm-lock.yaml` committed; `node_modules/` ignored.



### Pitfalls

- Activating `.venv` in one terminal and running `pytest` in another without it. Use `uv run` everywhere; it is unambiguous.
- `pip install` inside a uv project. It works but does not update `pyproject.toml`/`uv.lock`, so the next `uv sync` removes it. Use `uv add`.
- Global `npm install -g` for project tools. Use `pnpm add -D` and scripts; or `pnpm dlx` for one-offs.
- Mixing `npm`, `yarn` and `pnpm` in one repo creates multiple lockfiles that disagree. One package manager per repo; delete the others' lockfiles.
- Pinning `rev:` in `.pre-commit-config.yaml` to `main`. Pin tags; run `pre-commit autoupdate` monthly.
- ruff-format and a separate `isort` both installed — they fight. ruff's `I` rules replace isort.

---



## 5. VS Code (or Cursor) setup



### Theory

An IDE earns its keep through three things: language intelligence (types, go-to-definition, rename), an integrated debugger, and Git tooling. Cursor is a VS Code fork; everything below applies to both, and settings/extensions transfer.

**Extensions to install** (by ID; `code --install-extension <id>`):

- `ms-python.python` (includes Pylance = pyright language server) and `ms-python.debugpy`
- `charliermarsh.ruff` — lint/format on save
- `tamasfe.even-better-toml`
- `dbaeumer.vscode-eslint`, `esbenp.prettier-vscode`, `bradlc.vscode-tailwindcss` (web)
- `ms-azuretools.vscode-containers` (Docker/containers view)
- `eamodio.gitlens` — blame, history, PR review inside the editor
- `ms-vscode-remote.remote-wsl` (WSL users), `ms-vscode-remote.remote-containers` (optional)
- `editorconfig.editorconfig`, `redhat.vscode-yaml`

**Settings hierarchy.** User settings (`~/Library/Application Support/Code/User/settings.json` on macOS, `~/.config/Code/User/settings.json` on Linux) apply everywhere; workspace settings (`.vscode/settings.json`) override per repo and *can* be committed when they encode team conventions (formatter, interpreter path). Personal preferences (font, theme) stay in user settings.

**Debugging.** `launch.json` describes *how to start* a program under the debugger: type (`debugpy`, `node`), what to run (`module`/`program`), arguments, env, working dir. Breakpoints, conditional breakpoints (right-click → "Edit breakpoint"), logpoints (print without stopping), the Variables and Call Stack panes, and the Debug Console (evaluate expressions in the paused frame) replace 80% of `print` debugging. For a web server you attach the debugger to the running process or launch it with the debugger from the start.

**Integrated terminal.** Same shell as your terminal app (`terminal.integrated.defaultProfile.`*); split panes; the editor auto-links file paths and URLs in output. Learn `Ctrl+```  to toggle it.

**GitLens.** Inline blame tells you *who and why* for any line (then read the PR). The Commit Graph and "File History" views make `git log -p -- path` visual. Use it for reading; do your Git operations in the terminal this month so you learn the commands.

### Reading

- [VS Code — User and workspace settings](https://code.visualstudio.com/docs/configure/settings)
- [VS Code — Getting started with Python](https://code.visualstudio.com/docs/python/python-tutorial) and [Python debugging](https://code.visualstudio.com/docs/python/debugging)
- [VS Code — Debugging (general)](https://code.visualstudio.com/docs/debugtest/debugging)
- [VS Code — Node.js debugging](https://code.visualstudio.com/docs/nodejs/nodejs-debugging) and [TypeScript debugging](https://code.visualstudio.com/docs/typescript/typescript-debugging)
- [VS Code — Terminal basics](https://code.visualstudio.com/docs/terminal/basics)
- [GitLens — Marketplace page and feature tour](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
- [Cursor Docs](https://cursor.com/docs) (if you choose Cursor)



### Videos

- **"Getting Started with Python in VS Code (Official Video)"** — Visual Studio Code, ~10 min. Interpreter selection and the test explorer. [youtube.com/watch?v=D2cwvpJSBX4](https://www.youtube.com/watch?v=D2cwvpJSBX4)
- **"Getting Started with Debugging in VS Code (Official Beginner Guide)"** — Visual Studio Code, ~7 min. Breakpoints, step over/into, watch expressions. [youtube.com/watch?v=3HiLLByBWkg](https://www.youtube.com/watch?v=3HiLLByBWkg)



### Lab 5 — Debug Python and TypeScript (1 h)

```bash
code --install-extension ms-python.python --install-extension ms-python.debugpy \
  --install-extension charliermarsh.ruff --install-extension tamasfe.even-better-toml \
  --install-extension eamodio.gitlens --install-extension ms-azuretools.vscode-containers \
  --install-extension dbaeumer.vscode-eslint --install-extension esbenp.prettier-vscode \
  --install-extension editorconfig.editorconfig --install-extension redhat.vscode-yaml
```

In `~/projects/week1-py`, create `.vscode/settings.json`:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": { "source.fixAll.ruff": "explicit", "source.organizeImports.ruff": "explicit" }
  },
  "files.exclude": { "**/__pycache__": true, "**/.pytest_cache": true },
  "editor.rulers": [100]
}
```

and `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI (uvicorn, reload off)",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["app.main:app", "--port", "8000"],
      "cwd": "${workspaceFolder}",
      "envFile": "${workspaceFolder}/.env",
      "jinja": true,
      "justMyCode": true
    },
    {
      "name": "pytest: current file",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["${file}", "-q"],
      "cwd": "${workspaceFolder}",
      "justMyCode": false
    }
  ]
}
```

Steps:

1. Open `app/main.py`, click in the gutter next to `return {"status": "ok", ...}` to set a breakpoint.
2. Run "FastAPI (uvicorn, reload off)" (F5). In a terminal: `http :8000/health`. Execution pauses; inspect `settings` in Variables; evaluate `settings.model_dump()` in the Debug Console; step over (F10); continue (F5).
3. Right-click the breakpoint → Edit → Condition: `settings.app_env == "prod"`. Re-request; it should *not* pause. Change `.env` to `APP_ENV=prod`, restart the debugger, re-request; it pauses.
4. Open `tests/test_main.py`, set a breakpoint in `test_slugify`, run "pytest: current file"; step *into* `slugify` (F11).
5. (Web interns) In `~/projects/week1-ts` create `.vscode/launch.json` with a `node` configuration that runs `tsx`, set a breakpoint inside `slugify` and hit it:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "tsx: src/index.ts",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["exec", "tsx", "src/index.ts"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

1. In `week1-py`, open GitLens → File History on `app/main.py`; hover a line to see inline blame.

**Acceptance criteria**

- [ ] Screenshot (or live demo Wednesday) of the debugger paused inside the `/health` handler with `settings` expanded in Variables.
- [ ] The conditional breakpoint behaves as described.
- [ ] Saving a Python file with a misordered import reorders it automatically (ruff on save).
- [ ] (Web) Debugger paused inside `slugify` in TypeScript with the `text` argument visible.
- [ ] `.vscode/settings.json` and `launch.json` are committed; `.vscode/*.log` or personal state files are not.



### Pitfalls

- Wrong interpreter. If Pylance underlines `fastapi` as unresolved, the interpreter is not `.venv`; run "Python: Select Interpreter".
- `--reload` with the debugger: uvicorn's reloader spawns a child process and breakpoints do not bind. Debug without `--reload`, or use `"subProcess": true`.
- `justMyCode: true` hides library frames — good by default, but turn it off when the bug is *in* how you call the library.
- Format-on-save with two formatters registered (Prettier and ESLint both claiming `.ts`) — set `editor.defaultFormatter` per language.

---



## 6. HTTP fundamentals



### Theory

HTTP is a text protocol (HTTP/1.1; HTTP/2 and /3 binary-frame the same semantics) of stateless request/response pairs.

**Request anatomy**: a start line `METHOD /path?query HTTP/1.1`, headers (`Host`, `Accept`, `Content-Type`, `Authorization`, `Cookie`, `User-Agent`), a blank line, optional body. **Response anatomy**: `HTTP/1.1 STATUS REASON`, headers (`Content-Type`, `Content-Length`, `Set-Cookie`, `Cache-Control`, `Location`), blank line, body.

**Methods and semantics** — this is what makes an API "RESTful" in practice:


| Method  | Meaning                        | Safe (no side effects) | Idempotent     | Has body |
| ------- | ------------------------------ | ---------------------- | -------------- | -------- |
| GET     | read                           | yes                    | yes            | no       |
| HEAD    | headers only                   | yes                    | yes            | no       |
| POST    | create / non-idempotent action | no                     | **no**         | yes      |
| PUT     | replace whole resource         | no                     | yes            | yes      |
| PATCH   | partial update                 | no                     | not guaranteed | yes      |
| DELETE  | remove                         | no                     | yes            | rarely   |
| OPTIONS | capabilities (CORS preflight)  | yes                    | yes            | no       |


*Idempotent* means "sending the same request N times has the same effect on the server as sending it once". Retrying a failed `PUT /users/42` is safe; retrying `POST /orders` may create two orders — which is why payment APIs use an `Idempotency-Key` header. You will design around this in week 2 (web) and when your agents call tools over the network (AI).

**Status codes** by class: 1xx informational; **2xx** success (200 OK, 201 Created + `Location`, 204 No Content); **3xx** redirection (301/308 permanent, 302/307 temporary, 304 Not Modified for caching); **4xx** client error (400 bad request, 401 *unauthenticated*, 403 *unauthorized/forbidden*, 404, 405 method not allowed, 409 conflict, 422 validation error — FastAPI's default, 429 rate limited); **5xx** server error (500, 502 bad gateway — usually your app is down behind Traefik, 503 unavailable, 504 gateway timeout). Never return 200 with `{"error": ...}` in the body; clients and proxies rely on the status.

**Headers you must know**: `Content-Type` (media type of the body: `application/json; charset=utf-8`, `application/x-www-form-urlencoded`, `multipart/form-data` for uploads, `text/html`), `Accept` (what the client wants back), `Authorization: Bearer <token>`, `Cache-Control`/`ETag`/`If-None-Match`, `Location`, `X-Request-ID` (for tracing), and the `X-Forwarded-`* family that Traefik sets so your app knows the original scheme and client IP.

**Cookies** are `Set-Cookie` response headers the browser stores and sends back automatically on matching requests. Attributes: `HttpOnly` (JS cannot read it — protects session tokens from XSS), `Secure` (HTTPS only), `SameSite=Lax|Strict|None` (CSRF mitigation), `Path`, `Domain`, `Max-Age`. Session cookies vs. bearer tokens in `Authorization` is the week-4 auth debate; both need HTTPS.

**CORS in one paragraph.** Browsers enforce the same-origin policy: JavaScript at `https://app.example.com` may not read responses from `https://api.example.com` unless the API opts in with `Access-Control-Allow-Origin` (and, for credentials, `Access-Control-Allow-Credentials: true` with an explicit origin, never `*`). For "non-simple" requests (JSON bodies, custom headers, non-GET/POST methods) the browser first sends an `OPTIONS` *preflight* and expects `Access-Control-Allow-Methods/Headers`. CORS is a browser-only mechanism — `curl` never sees a CORS error — and it is *not* a security boundary for your API; authentication is.

**TLS in one paragraph.** HTTPS is HTTP inside a TLS session. The client and server perform a handshake: they agree on a cipher suite, the server presents an X.509 certificate chaining to a CA the client trusts, they establish a shared symmetric key (ECDHE gives forward secrecy), and everything after — including the URL path, headers and body — is encrypted and integrity-protected; only the hostname (SNI) and IP are visible on the wire. On Dokploy, Traefik terminates TLS with free Let's Encrypt certificates and forwards plain HTTP to your container on the internal Docker network; your app never touches certificates, but it *must* trust `X-Forwarded-Proto` to know it was HTTPS.

### Reading

- [MDN — Overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) and [HTTP messages](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages)
- [MDN — HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods) and [Glossary: Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [MDN — Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies)
- [MDN — Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Cloudflare — What happens in a TLS handshake?](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)
- [HTTPie — Usage](https://httpie.io/docs/cli/usage); [httpbin.org](https://httpbin.org/) — an echo API for experiments; [GitHub REST API quickstart](https://docs.github.com/en/rest/quickstart)



### Videos

- **"HTTP Crash Course & Exploration"** — Traversy Media, ~38 min. Request/response cycle, status codes, headers; the Express examples are incidental. [youtube.com/watch?v=iYM2zFP3Zn0](https://www.youtube.com/watch?v=iYM2zFP3Zn0)
- **"TLS Handshake Explained — Computerphile"** — Computerphile, ~17 min. Enough TLS to reason about Traefik and certificates. [youtube.com/watch?v=86cQJ0MMses](https://www.youtube.com/watch?v=86cQJ0MMses)



### Lab 6 — Read the wire with curl and HTTPie (1 h)

```bash
# 1. See raw request and response (-v), no body download (-I / HEAD)
curl -v https://httpbin.org/get 2>&1 | grep -E '^(>|<)'     # lines starting with > are sent, < received
curl -I https://github.com                                     # HEAD: status + headers only

# 2. Status codes on demand
for code in 200 201 204 301 302 401 403 404 409 422 429 500 502 503; do
  printf "%s -> " "$code"; curl -s -o /dev/null -w "%{http_code} redirected_to=%{redirect_url}\n" "https://httpbin.org/status/$code"
done
curl -sL -o /dev/null -w "%{http_code} after %{num_redirects} redirects\n" https://httpbin.org/redirect/3

# 3. Methods, bodies, content types
http POST https://httpbin.org/post name=osama role=intern            # JSON body (default)
http --form POST https://httpbin.org/post name=osama role=intern     # application/x-www-form-urlencoded
curl -s -X PUT https://httpbin.org/put -H 'Content-Type: application/json' -d '{"id":42,"name":"x"}' | jq '.json, .headers."Content-Type"'
curl -s -X DELETE https://httpbin.org/delete | jq '.url'
curl -s -X PATCH https://httpbin.org/patch -H 'Content-Type: application/json' -d '{"name":"y"}' | jq '.json'

# 4. Headers and auth
http https://httpbin.org/bearer Authorization:"Bearer not-a-real-token"     # 200 echoes the token
http https://httpbin.org/bearer                                             # 401
http -a user:pass https://httpbin.org/basic-auth/user/pass                  # basic auth → 200
http https://httpbin.org/basic-auth/user/pass                               # 401 + WWW-Authenticate header

# 5. Cookies
curl -c jar.txt -s https://httpbin.org/cookies/set/session/abc123 -o /dev/null && cat jar.txt
curl -b jar.txt -s https://httpbin.org/cookies | jq
http --session=lab https://httpbin.org/cookies/set/theme/dark ; http --session=lab https://httpbin.org/cookies

# 6. Caching / conditional requests against GitHub's API
curl -si https://api.github.com/repos/astral-sh/uv | grep -iE '^(etag|cache-control|x-ratelimit-remaining)'
ETAG=$(curl -si https://api.github.com/repos/astral-sh/uv | grep -i '^etag' | cut -d' ' -f2 | tr -d '\r')
curl -s -o /dev/null -w "%{http_code}\n" -H "If-None-Match: $ETAG" https://api.github.com/repos/astral-sh/uv   # 304

# 7. CORS preflight, seen from the server side
curl -si -X OPTIONS https://httpbin.org/anything -H 'Origin: https://app.example.com' \
  -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type' | grep -i '^access-control'

# 8. TLS: inspect the certificate chain
curl -vI https://github.com 2>&1 | grep -E 'SSL connection|subject:|issuer:|expire'
openssl s_client -connect github.com:443 -servername github.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# 9. Your own app from Lab 4 (run `uv run fastapi dev app/main.py` in another terminal)
http :8000/health
http :8000/nope                                   # 404 JSON from FastAPI
curl -si :8000/health | head -5                   # note server, content-type, content-length
```

Write `interns/<yourhandle>/http-notes.md` (goes in your week-1 PR or a follow-up) answering: (a) which of the requests in step 3 are idempotent and why; (b) what the browser would do differently from `curl` in step 7; (c) why step 6 returned 304 and what the client should do with it; (d) the three cookie attributes a session cookie for your capstone must have.

**Acceptance criteria**

- [ ] `http-notes.md` committed with correct answers (a)–(d).
- [ ] You can produce, from memory, a `curl` that sends a JSON `POST` with a bearer token and prints only the status code.
- [ ] Given any status code from the table, you name its class and one situation your API would return it.



### Pitfalls

- `curl -d` without `-H 'Content-Type: application/json'` sends `application/x-www-form-urlencoded`; FastAPI then returns 422. Read the request you sent, not the one you meant.
- `-L` follows redirects silently; without it a 301 looks like "the API returned nothing".
- Trusting `X-Forwarded-For` from any client — only from your own proxy (week 11).
- "CORS error" in the browser console with a *200* in the Network tab means the server responded fine but without the `Access-Control-`* headers; fix the server, do not disable browser security.

---



## 7. Docker fundamentals



### Theory

A **container** is a normal Linux process (or process tree) isolated with kernel namespaces (own PID space, network stack, mount table) and limited with cgroups. It is not a VM: no separate kernel, starts in milliseconds, shares the host kernel (on macOS/Windows, Docker Desktop runs one hidden Linux VM for all containers).

An **image** is an immutable, layered filesystem plus metadata (default command, env, exposed ports, working dir). Each `Dockerfile` instruction that changes the filesystem (`RUN`, `COPY`, `ADD`) creates a **layer**; layers are content-addressed and cached. Docker rebuilds from the first instruction whose inputs changed and everything after it. Therefore: copy dependency manifests and install dependencies *before* copying source code, so editing a `.py` file does not reinstall every package.

**Dockerfile essentials**: `FROM` (base image; pin a tag like `python:3.13-slim`, not `latest`), `WORKDIR`, `COPY`, `RUN` (build-time), `ENV`, `EXPOSE` (documentation only; publishing is `-p`), `USER` (do not run as root), `CMD` (default runtime command, exec form `["uvicorn", "..."]` so signals reach the process), `ENTRYPOINT` (fixed executable, args from `CMD`). `.dockerignore` keeps `.git`, `.venv`, `node_modules`, `.env` and caches out of the build context — smaller uploads, faster builds, no secrets baked into layers.

**Multi-stage builds** use one stage to build (compilers, dev dependencies, `uv` itself) and a final `FROM` stage that copies only the artifacts (`.venv`, compiled assets). Result: a 150 MB image instead of 1.2 GB, with no build tools available to an attacker.

**Volumes** persist data beyond a container's life: *named volumes* (`pgdata:/var/lib/postgresql/data`, managed by Docker — use for databases) and *bind mounts* (`./app:/app/app`, a host directory — use for live-reload in development, never in production). Containers are disposable; anything not in a volume disappears with `docker compose down -v`.

**Networks**: Compose creates a per-project bridge network; services reach each other by *service name* via built-in DNS (`postgresql://app:app@db:5432/app` — host is `db`, not `localhost`). `localhost` inside a container is the container itself. `-p 8000:8000` (`ports:` in Compose) publishes a container port on the host; without it, only other containers can connect.

**Compose** (`compose.yaml`, v2 spec — no `version:` key) declares services, their images/builds, env, ports, volumes and dependencies. `depends_on` with `condition: service_healthy` plus a `healthcheck` on the database is the only correct way to wait for Postgres; plain `depends_on` only orders *start*, and Postgres accepts TCP before it is ready. Dokploy deploys exactly this file (see `shared/deployment-dokploy.md`), so what you write this week is production-shaped.

**Debugging loop**: `docker compose ps` (state and health), `docker compose logs -f <svc>` (stdout/stderr — log to stdout, never to files), `docker compose exec <svc> sh` (shell into a running container), `docker inspect` (env, mounts, IP), `docker compose config` (the fully resolved file, with env substituted). If a container exits immediately, `docker compose logs <svc>` and `docker compose run --rm <svc> sh` to poke around.

### Reading

- [Docker Docs — What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/) and [Writing a Dockerfile](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/)
- [Docker Docs — Dockerfile reference](https://docs.docker.com/reference/dockerfile/) and [Building best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Docs — Build context and .dockerignore](https://docs.docker.com/build/concepts/context/) and [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Docs — Persisting container data (volumes)](https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/)
- [Docker Docs — Compose Quickstart](https://docs.docker.com/compose/gettingstarted/), [Networking in Compose](https://docs.docker.com/compose/how-tos/networking/), [Compose services reference (healthcheck, depends_on)](https://docs.docker.com/reference/compose-file/services/)
- [uv — Using uv in Docker](https://docs.astral.sh/uv/guides/integration/docker/) and [FastAPI — Docker deployment](https://fastapi.tiangolo.com/deployment/docker/)
- [Docker Hub — postgres image (env vars,](https://hub.docker.com/_/postgres) `PGDATA`[, healthcheck notes)](https://hub.docker.com/_/postgres)



### Videos

- **"Docker in 100 Seconds"** — Fireship, 2 min. Vocabulary primer before reading. [youtube.com/watch?v=Gjnup-PuquQ](https://www.youtube.com/watch?v=Gjnup-PuquQ)
- **"100+ Docker Concepts you Need to Know"** — Fireship, ~9 min. Rapid glossary; pause on layers, volumes, compose. [youtube.com/watch?v=rIrNIzy6U_g](https://www.youtube.com/watch?v=rIrNIzy6U_g)
- **"Docker Tutorial for Beginners [FULL COURSE in 3 Hours]"** — TechWorld with Nana, ~2 h 45 min. Watch the chapters on containers vs VMs, main commands, debugging, Compose, Dockerfile and volumes (~1.5 h); skip the AWS registry chapter. [youtube.com/watch?v=3c-iBn73dDE](https://www.youtube.com/watch?v=3c-iBn73dDE)



### Lab 7 — Containerize FastAPI + Postgres with Compose (2.5 h)

Continue in `~/projects/week1-py`. First extend the app to actually use the database (raw SQL via `psycopg`; SQLAlchemy comes in week 3).

```bash
uv add "psycopg[binary]"
cat > app/db.py <<'EOF'
import psycopg

from app.settings import Settings


def ping(settings: Settings) -> str:
    with psycopg.connect(settings.database_url) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT version()")
            row = cur.fetchone()
            return row[0] if row else "unknown"
EOF
cat > app/main.py <<'EOF'
from fastapi import FastAPI

from app.db import ping
from app.settings import Settings

settings = Settings()
app = FastAPI(title="week1")


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok", "env": settings.app_env}


@app.get("/db")
def db() -> dict[str, str]:
    return {"postgres": ping(settings)}


def slugify(text: str) -> str:
    return "-".join(text.lower().split())
EOF
```

`Dockerfile` (multi-stage, uv, non-root):

```dockerfile
# ---- build stage: resolve and install dependencies into /app/.venv
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS builder
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0
WORKDIR /app
# dependency manifests first → this layer is cached until they change
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev
# then the source
COPY app ./app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# ---- runtime stage: no uv, no build tools, non-root user
FROM python:3.13-slim-bookworm AS runtime
RUN useradd --create-home --uid 1000 appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/.venv /app/.venv
COPY --from=builder --chown=appuser:appuser /app/app /app/app
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`.dockerignore`:

```
.git
.venv
.env
.env.*
**/__pycache__
.pytest_cache
.ruff_cache
.vscode
tests
*.md
```

`compose.yaml`:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set it in .env}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"     # host access for psql/DBeaver; not exposed to the network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s

  api:
    build: .
    environment:
      APP_ENV: docker
      DATABASE_URL: postgresql://app:${POSTGRES_PASSWORD}@db:5432/app
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health').status==200 else 1)\""]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 5s

volumes:
  pgdata:
```

Run it:

```bash
printf 'APP_ENV=dev\nPOSTGRES_PASSWORD=devpass\nDATABASE_URL=postgresql://app:devpass@localhost:5432/app\n' > .env
docker compose config              # resolved file; confirm the password was substituted and nothing is missing
docker compose up -d --build
docker compose ps                  # both "healthy"
docker compose logs -f api         # Ctrl+C to stop following
http :8000/db                      # {"postgres": "PostgreSQL 17.x ..."}
docker compose exec db psql -U app -d app -c '\l'
docker compose exec api sh -c 'id && env | grep -E "APP_ENV|DATABASE_URL" && ls -la'
docker image ls | grep week1       # note the size; compare with: docker image history week1-py-api

# Break it on purpose and diagnose
docker compose stop db && http :8000/db                    # 500; read the traceback in: docker compose logs api
docker compose start db && sleep 6 && http :8000/db        # recovers without restarting api

# Persistence
docker compose exec db psql -U app -d app -c 'CREATE TABLE notes(id serial primary key, body text); INSERT INTO notes(body) VALUES ($$survives$$);'
docker compose down                 # containers gone, volume kept
docker compose up -d && sleep 8 && docker compose exec db psql -U app -d app -c 'SELECT * FROM notes;'
docker compose down -v              # -v deletes the volume: data gone. Understand this before running it on the server.
```

Layer-cache experiment: edit `app/main.py` (change the health message), `docker compose build` and observe that only the last `COPY app` and `uv sync` layers rerun (seconds). Then add a dependency with `uv add httpx` and rebuild: the dependency layer reruns. Explain why in your notes.

**Acceptance criteria**

- [ ] `docker compose up -d --build` from a clean state (`docker compose down -v --rmi local` first) reaches two `healthy` services within 60 s and `http :8000/db` returns the Postgres version.
- [ ] `docker compose exec api id` shows `uid=1000(appuser)`, not root.
- [ ] Final image < 250 MB (`docker image ls`); you can explain what the builder stage contained that the runtime stage does not.
- [ ] `.env` is not in the image: `docker compose exec api sh -c 'ls -la /app; cat /app/.env'` fails with "No such file".
- [ ] You can explain, in the 1:1, why the `DATABASE_URL` host is `db` inside Compose and `localhost` in your local `.env`.
- [ ] `Dockerfile`, `.dockerignore`, `compose.yaml` committed; `.env` not.



### Pitfalls

- `localhost` inside a container. It is the container. Use the service name.
- `depends_on` without a healthcheck condition; the app crashes on first start because Postgres is not ready yet; "works on retry" is not a fix.
- Copying source before installing dependencies: every code change reinstalls everything.
- `FROM python:latest`. Pin major.minor and the variant (`-slim`); rebuilds must be reproducible on the server.
- Running as root in the container; writing logs to files inside the container (they vanish; log to stdout).
- `docker compose down -v` on a server. It deletes the database volume. Dokploy's UI has an equivalent button; read the deployment guide you receive in week 9 before touching it.
- Baking secrets with `ENV SECRET=...` or `COPY .env` — they are in the image layers forever, visible with `docker history`.

---



## 8. Linux and ops basics



### Theory

**Processes.** Every running program is a process with a PID, a parent, a user, an environment, open file descriptors, and eventually an exit code. `ps aux`, `pgrep -f uvicorn`, `top`/`htop` show them. A process ends when it exits or receives a fatal signal. Containers are processes; `docker compose stop` sends SIGTERM to PID 1 in the container.

**Signals** are asynchronous notifications: `SIGINT` (Ctrl+C; interactive interrupt), `SIGTERM` (polite "please stop"; the default for `kill`, `docker stop`, systemd), `SIGKILL` (uncatchable, immediate; `kill -9`; data loss likely), `SIGHUP` (terminal closed / "reload config" by convention). A well-behaved server catches SIGTERM, stops accepting connections, finishes in-flight requests, closes the DB pool, exits 0 — this is *graceful shutdown*, and it only works if your process is PID 1's direct child or PID 1 itself (hence `CMD` exec form) and you do not wrap it in a shell that swallows signals. Docker waits 10 s after SIGTERM then sends SIGKILL.

**Exit codes**: 0 success, 1 generic failure, 2 misuse, 126/127 not executable / not found, 128+N killed by signal N (137 = SIGKILL, often the OOM killer; 143 = SIGTERM). `docker compose ps` shows them; CI fails on non-zero.

**systemd at a glance.** On Ubuntu servers, systemd is PID 1 and manages *units* (`.service`, `.timer`, `.socket`). `systemctl status docker`, `systemctl restart nginx`, `systemctl enable --now foo` (start now and at boot), `journalctl -u docker -f` (logs). On the Dokploy server, Docker itself is a systemd service; your apps are containers *under* Docker, so you will rarely write unit files — but you must be able to read `systemctl status` output when the supervisor shares it.

**File permissions.** Every file has an owner, a group, and three permission triplets (owner/group/others) of `r`/`w`/`x`. `ls -l` shows `-rwxr-x---`; numerically 750. `chmod 600 ~/.ssh/id_ed25519` (private key: owner read/write only — SSH refuses keys that are more permissive), `chmod +x script.sh`, `chown appuser:appuser /app`. Directories need `x` to be entered. Inside containers, UID matching matters: a bind-mounted directory owned by your host UID 501 will be unwritable to a container user with UID 1000 — the cause of most "permission denied" errors in Compose development setups.

**Twelve-factor principles you apply from day one:**

1. One codebase in Git, many deploys (local, staging, prod).
2. Dependencies explicitly declared and isolated (`uv.lock`, `pnpm-lock.yaml`, Docker image).
3. **Config in the environment** — anything that differs between deploys (DB URL, API keys, feature flags) is an env var, never a constant or a committed file. Local `.env` is a loader convenience; on Dokploy, env vars are set in the UI.
4. Backing services (Postgres, Redis, an LLM API) are attached resources addressed by URL; swapping them is a config change.
5. Build, release, run are separate stages (Docker image → image + env → container).
6. Processes are stateless; persistent state lives in backing services (never on the container's disk).
7. Port binding: the app exports HTTP by binding a port; Traefik routes to it.
8. Disposability: fast start, graceful shutdown.
9. Logs are event streams to stdout; the platform collects them.

**Secrets never in Git.** Not in code, not in `.env` committed "just for dev", not in Docker images, not in screenshots in PRs, not in Notion. `.env.example` with placeholder values *is* committed so newcomers know which variables exist. If a secret leaks into a commit — even one you never pushed — treat it as compromised and rotate it; rewriting history is a cleanup step, not the fix. Pre-commit's `detect-private-key` and GitHub's push protection are backstops, not the plan.

### Reading

- [Julia Evans — Should you be scared of Unix signals?](https://jvns.ca/blog/2016/06/13/should-you-be-scared-of-signals/)
- [Missing Semester — Command-line Environment (job control, signals)](https://missing.csail.mit.edu/2020/command-line/)
- [DigitalOcean — How To Use Systemctl to Manage Systemd Services and Units](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)
- [Ubuntu manpage — chmod](https://manpages.ubuntu.com/manpages/trusty/man1/chmod.1.html)
- [The Twelve-Factor App](https://12factor.net/) — read all twelve; they are short
- [Docker Docs — Linux post-installation steps (docker group, service)](https://docs.docker.com/engine/install/linux-postinstall/)
- [pydantic-settings — settings management from env and .env](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)



### Videos

- **"Linux Crash Course — Understanding File & Directory Permissions"** — Learn Linux TV, ~28 min. Watch until the numeric-mode section makes `chmod 640` obvious. [youtube.com/watch?v=4e669hSjaX8](https://www.youtube.com/watch?v=4e669hSjaX8)



### Lab 8 — Signals, permissions, config (30 min)

```bash
cd ~/projects/week1-py

# 1. Signals and graceful shutdown of your API (outside Docker)
uv run uvicorn app.main:app --port 8000 &
PID=$!; sleep 2
kill -TERM $PID; wait $PID; echo "exit=$?"          # uvicorn logs "Shutting down" and exits 0
uv run uvicorn app.main:app --port 8000 &
PID=$!; sleep 2; kill -KILL $PID; wait $PID; echo "exit=$?"   # 137: killed by signal 9, no cleanup

# 2. Same inside Docker: watch the stop time and the logged shutdown
docker compose up -d && sleep 8
time docker compose stop api                        # ~1 s if SIGTERM is handled; ~10 s means it was SIGKILLed
docker compose logs --tail=5 api

# 3. Permissions
ls -l ~/.ssh                                        # id_ed25519 must be -rw------- (600)
touch script.sh && ls -l script.sh && chmod +x script.sh && ls -l script.sh
stat -f '%Su:%Sg %Sp' script.sh 2>/dev/null || stat -c '%U:%G %A' script.sh    # macOS || Linux

# 4. Config in the environment, not the code
cp .env .env.example && sed -i.bak 's/=.*/=/' .env.example && rm -f .env.example.bak && cat .env.example
git check-ignore -v .env                            # must print a rule; if not, fix .gitignore NOW
git log --all -p -S 'devpass' --oneline | head      # prove the dev password never entered history
APP_ENV=staging uv run python -c 'from app.settings import Settings; print(Settings().app_env)'   # env beats .env

# 5. systemd (Linux/WSL with systemd, or on the supervisor's server during the 1:1)
systemctl status docker --no-pager | head -5 2>/dev/null || echo "no systemd here (macOS) — read the output the supervisor shows you"
```

**Acceptance criteria**

- [ ] You can state the exit code for SIGTERM-handled, SIGKILL, and "command not found", and what 137 usually means in Docker.
- [ ] `docker compose stop api` completes in about a second and the log shows a graceful shutdown.
- [ ] `.env.example` committed; `.env` ignored; the `git log -S` search for your password returns nothing.
- [ ] You can list the 12 factors from memory and give one concrete example from your own repo for factors 2, 3, 6 and 11.



### Pitfalls

- `CMD uvicorn ...` (shell form) makes `/bin/sh` PID 1; SIGTERM goes to the shell, not uvicorn; Docker SIGKILLs after 10 s. Use exec form.
- `chmod 777` to make a permission error go away. Find the actual owner/UID mismatch.
- Reading config with `os.environ["X"]` scattered across modules. Centralize in one `Settings` object, validate at startup, fail fast.
- A `.env` committed "temporarily". There is no temporarily in Git.

---



## 9. AI coding assistants



### Theory

In 2026 every professional developer uses an agentic coding tool — Claude Code in the terminal, Cursor as an editor, Copilot inside VS Code/GitHub — the way they use a compiler: constantly, and without ceding judgement. The supervisor uses them daily and expects you to. What separates a good user from a liability is not prompting tricks; it is whether *you* remain the engineer of record.

How professionals actually use them:

- **Reading and navigation first.** "Explain how auth flows through this repo", "where is the retry logic", "summarize this 400-line migration". Faster than grep, and it teaches you the codebase.
- **Scaffolding and boilerplate** — tests from a spec, a Pydantic model from a JSON sample, a Dockerfile draft — followed by *your* review line by line.
- **Plan, then execute.** For anything non-trivial: ask for a plan, correct it, *then* let it edit. Tools with a plan mode (Claude Code's plan mode, Cursor's agent) exist for this reason.
- **Tight verification loops.** Give the tool the command that proves success (`uv run pytest tests/test_x.py`, `pnpm typecheck`) and let it iterate against real output instead of guessing.
- **Project memory.** A `CLAUDE.md` / `.cursor/rules` / `.github/copilot-instructions.md` with build commands, conventions and gotchas makes every session start informed. You will write one for your capstone in week 4.
- **Small diffs, reviewed like a human's PR.** Generated code goes through the same PR, CI, and review as yours — because it *is* yours once you commit it.

What you never do:

- **Paste secrets, tokens, customer data or the supervisor's server credentials into any assistant.** `.env` contents, API keys, database dumps — never. Assume everything you type may be logged.
- **Accept code you cannot explain.** In every review the supervisor may point at any line and ask "why is this here?". "The AI wrote it" is a failing answer; "it handles the case where X because Y" is the only acceptable one.
- **Let it run destructive commands unattended.** `rm -rf`, `git push --force`, `docker compose down -v`, `DROP TABLE`, anything touching production. Read every command before approving; keep auto-approve off for shell access this month.
- **Skip the tests because "it said it works".** Run them yourself. Models confidently report success on code that does not compile.
- **Use it to avoid learning.** This week's labs are to be typed by you. Use the assistant to *explain* errors and concepts, not to do the exercise.

How the supervisor expects them used in this internship (full policy, including disclosure rules and what counts as your own work in assessments, in the *Using AI Coding Assistants* document):

- Allowed and encouraged for all coding work from week 2, with disclosure in the PR description ("Drafted with Claude Code; reviewed and tested by me").
- Week 1 labs: assistants for explanation only. Commands and code are typed by you.
- Every PR you open, you can walk through line by line. The Friday demo includes a random "explain this function" check.
- You maintain the project memory file for your capstone and keep it accurate.
- Cost and access: the supervisor provides the API keys or seats; you never put a personal key into a shared repo or server.



### Reading

- *Using AI Coding Assistants* — the internship policy document you received at kickoff (read this one first)
- [Claude Code — Best practices](https://code.claude.com/docs/en/best-practices) and [Anthropic Engineering — Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Cursor Docs — Agent overview](https://cursor.com/docs/agent/overview)
- [GitHub Docs — Best practices for using GitHub Copilot](https://docs.github.com/en/copilot/get-started/best-practices)
- [Google Engineering Practices — How to do a code review](https://google.github.io/eng-practices/review/reviewer/) (re-read it as "how to review AI output")



### Videos

- **"Mastering Claude Code in 30 minutes"** — Anthropic (Boris Cherny, Code with Claude 2025), ~30 min. Focus on CLAUDE.md, plan mode, and the test-driven loop. [youtube.com/watch?v=6eBSHbLKuN0](https://www.youtube.com/watch?v=6eBSHbLKuN0)
- **"Claude Code & the evolution of agentic coding — Boris Cherny, Anthropic"** — AI Engineer, ~20 min. The "why" behind terminal-native agents and verification loops. [youtube.com/watch?v=Lue8K2jqfKk](https://www.youtube.com/watch?v=Lue8K2jqfKk)



### Lab 9 — Use an assistant the right way (30 min, part of Lab 3/7 time)

1. Install one assistant (Claude Code, Cursor, or Copilot — the supervisor tells you which seat you have at kickoff).
2. In `~/projects/week1-py`, ask it to *explain* your `Dockerfile` line by line and to list three things it would change. Do not let it edit. Write down which suggestions you agree with and why; one of them is probably wrong or unnecessary — find it.
3. Ask it to explain the `git rebase` conflict from Lab 3 given the two versions of `pricing.py`. Compare its explanation with your resolution.
4. Ask it to generate two additional pytest cases for `slugify` (edge cases). Read them. Run them. If one is wrong or redundant, delete it and note why.
5. Create `CLAUDE.md` (or `.cursor/rules/project.md`) in `week1-py` with: how to run tests, lint, and the app; the rule "never commit `.env`"; the rule "use `uv add`, never `pip install`".

**Acceptance criteria**

- [ ] `interns/<yourhandle>/README.md` contains a short "AI assistant notes" section: which tool, one suggestion you rejected and why.
- [ ] The extra `slugify` tests exist, pass, and you can justify each.
- [ ] `CLAUDE.md` (or equivalent) committed in `week1-py` with accurate commands.
- [ ] You can recite the three "never" rules without notes.



### Pitfalls

- Letting the assistant `git commit` with its own message style, or add itself as co-author, without your review of the diff and message.
- Prompting with a screenshot of your terminal that contains a token.
- Asking for "the whole feature" — you get 600 lines you do not understand. Ask for the plan, then one step.
- Trusting library APIs it cites. It may be describing a 2023 version of a library. Check the docs the way you did for every link in this document.

---



## 10. Week deliverables & acceptance checklist

Submit by Friday 18 September, 12:00, via PR to the internship repo (Part C of Lab 3) plus follow-up commits to the same PR or a second one.

**In the internship repo, under** `interns/<yourhandle>/`**:**

- [ ] `README.md` — track, machine, dotfiles repo link, hardest thing this week, AI assistant notes.
- [ ] `git-lab/pricing.py` and `git-lab/check.sh` from Lab 3 Part B (final resolved versions).
- [ ] `http-notes.md` from Lab 6 with answers (a)–(d).
- [ ] `week1-checklist.md` — this checklist copied with every box honestly ticked or annotated "not done: reason".

**In your own GitHub account (links in** `README.md`**):**

- [ ] Private `dotfiles` repo (Lab 1).
- [ ] `week1-py` repo: uv project with `pyproject.toml`, `uv.lock`, `.python-version`, `app/`, `tests/`, `.pre-commit-config.yaml`, `.vscode/{settings,launch}.json`, `Dockerfile`, `.dockerignore`, `compose.yaml`, `.env.example`, `CLAUDE.md`; CI-free this week, but `uv run pre-commit run --all-files`, `uv run pytest`, `uv run pyright` all clean.
- [ ] (Web) `week1-ts` repo with scripts, ESLint, Prettier, `launch.json`.
- [ ] `git-conflict` repo pushed (Lab 3 Part B) so the supervisor can inspect `git log --graph`. The reflog is local and does not push; be ready to show it live.

**Process:**

- [ ] Week-1 PR follows Conventional Commits, has What/Why/How-to-test, received and gave one review, merged by the supervisor.
- [ ] Wednesday 1:1 whiteboard done (see "Prepare for your Wednesday 1:1 and Friday demo" below).
- [ ] Friday demo: 10 minutes — `docker compose up` from clean, hit `/db`, show the debugger pausing, answer one random "explain this line" question.

Grading weights for the week: Git exercise 40%, toolchain + Docker labs 35%, process (PR quality, review given, checklist honesty) 25%.

---



## Self-check quiz

Answer without tools first; then verify. Answers at the bottom.

1. What are the three "areas" in Git and which command moves content between each pair?
2. A commit object contains which fields? Why does changing a commit's message change its hash?
3. After `git rebase main` on `feature`, are the commit hashes on `feature` the same as before? What does that imply for a branch you had already pushed?
4. During a rebase conflict, which side does `git checkout --ours` take?
5. You ran `git reset --hard HEAD~3` by mistake. Give the two commands that get the work back.
6. Write a Conventional Commit message for "fixed the bug where the price endpoint returned tax twice on discounted items", referencing issue 17.
7. What does `.gitignore` do to a file that is already tracked? What command fixes that?
8. Explain `2>&1 | tee out.log` word by word.
9. What is the difference between `$VAR`, `"$VAR"` and `'$VAR'`?
10. A server says "address already in use" on port 8000. Give the command to find the process and the one to stop it politely.
11. Why should `pyproject.toml` list `fastapi>=0.115` while `uv.lock` pins `fastapi==0.115.6`? Which one does the server use, and with which flag?
12. What does `uv run pytest` do that `pytest` alone might not?
13. Name the two things ruff replaces and the tool that checks types. Which pre-commit hook must run before the formatter, and why?
14. In `launch.json`, what is the difference between `"module": "uvicorn"` and `"program": "app/main.py"`? Why can breakpoints fail with `--reload`?
15. Classify: 201, 204, 304, 401, 403, 409, 422, 502, 504. For 401 vs 403, give the one-word distinction.
16. Which of GET, POST, PUT, PATCH, DELETE are idempotent? Why does a payment API need an `Idempotency-Key` header?
17. Your browser shows a CORS error but the Network tab shows 200. Where is the bug and what is *not* the fix?
18. Which three cookie attributes should a session cookie have and what does each defend against?
19. What does Traefik do with TLS on Dokploy, and how does your app know the original request was HTTPS?
20. Why do we `COPY pyproject.toml uv.lock` and install *before* `COPY app`? What happens to build time if you reverse the order?
21. Inside a Compose network, what hostname does the API use for Postgres? What does `localhost` mean inside a container?
22. What does `depends_on: db: condition: service_healthy` require on the `db` service? What goes wrong with plain `depends_on`?
23. What is the difference between `docker compose down` and `docker compose down -v`?
24. Exit code 137 in `docker compose ps` means what? What about 143?
25. State 12-factor factor III in one sentence and give one violation you have seen or could imagine in a student project.



### Answers

1. Working tree → index: `git add` (and `git add -p` for hunks); index → repository: `git commit`; repository → working tree/index: `git checkout`/`git switch`/`git restore` (`git restore --staged` unstages).
2. Tree hash, parent hash(es), author (name/email/time), committer, message (and optionally GPG/SSH signature). The hash is a SHA over all of these bytes, so any change — including the message — yields a different object; it is a new commit, not an edited one.
3. No: rebase creates new commits (new parents → new hashes). A pushed branch now diverges from the remote; you must `git push --force-with-lease`, and anyone who based work on the old commits must rebase too. Hence: only rebase unshared feature branches.
4. During a rebase, "ours" is the branch you are rebasing *onto* (upstream, e.g. `main`) and "theirs" is the commit being replayed (your work). The opposite of what most people expect; during a merge, "ours" is the current branch.
5. `git reflog` to find the entry before the reset (e.g. `HEAD@{1}`), then `git reset --hard HEAD@{1}` (or `git branch rescue <sha>` to keep it safely on a branch first).
6. `fix(pricing): apply tax once for discounted items` with body explaining the cause and footer `Closes #17`.
7. Nothing — ignore rules only affect untracked files. `git rm --cached <file>` removes it from the index (keeps it on disk); commit that, and from then on it is ignored.
8. `2>&1` redirects stderr (fd 2) to wherever stdout (fd 1) currently points; `|` pipes the combined stream to `tee`; `tee out.log` writes its stdin to `out.log` *and* to its own stdout so you see it live.
9. `$VAR` expands and then word-splits and globs (dangerous with spaces/empties); `"$VAR"` expands to exactly one argument; `'$VAR'` is the literal five characters.
10. `lsof -i :8000 -P -n` (or `lsof -t -i :8000` for just the PID); `kill <pid>` sends SIGTERM. `kill -9` only if it ignores SIGTERM.
11. `pyproject.toml` declares *compatible* ranges (what the code needs); `uv.lock` records the exact resolution that was tested, cross-platform. The server uses the lock: `uv sync --frozen --no-dev` (`--frozen` refuses to re-resolve; `--no-dev` skips test/lint tools).
12. Ensures the project environment exists and matches the lockfile (syncing if needed) and runs `pytest` from *that* environment, regardless of which venv is activated in your shell.
13. ruff replaces flake8 (+ plugins) and black (and isort via `I` rules); pyright (or mypy) checks types. `ruff-check --fix` runs before `ruff-format` because auto-fixes can produce code that then needs reformatting.
14. `module` runs `python -m uvicorn ...` (needed for CLIs and packages, uses the venv's module resolution); `program` runs a file as a script. `--reload` starts a supervisor process that spawns the real server as a child; the debugger attaches to the parent, so breakpoints in the child never bind (unless `subProcess: true`).
15. 201 Created (2xx), 204 No Content (2xx), 304 Not Modified (3xx), 401 Unauthorized (4xx), 403 Forbidden (4xx), 409 Conflict (4xx), 422 Unprocessable Content (4xx), 502 Bad Gateway (5xx), 504 Gateway Timeout (5xx). 401 = *unauthenticated* (who are you?), 403 = *unauthorized* (I know who you are; no).
16. GET, PUT, DELETE (and HEAD/OPTIONS) are idempotent; POST is not; PATCH is not guaranteed. Network retries after a timeout would otherwise create duplicate charges; the key lets the server recognise and de-duplicate the retry.
17. The bug is server-side: the response lacks the `Access-Control-Allow-*` headers for that origin/method (or the preflight `OPTIONS` fails). The fix is not disabling browser security, a `*` origin with credentials, or a proxy hack in production.
18. `HttpOnly` (JavaScript cannot read it → XSS cannot steal it), `Secure` (only sent over HTTPS → not sniffable), `SameSite=Lax` or `Strict` (not sent on cross-site requests → CSRF mitigation). Plus a sensible `Max-Age`/expiry and `Path`.
19. Traefik terminates TLS (Let's Encrypt certificates) at the edge and forwards plain HTTP over the internal Docker network. The app learns the original scheme from `X-Forwarded-Proto` (and client IP from `X-Forwarded-For`), which it must trust only from the proxy.
20. Layers are cached by content of their inputs; manifests change rarely, source changes constantly. With manifests first, a code edit only rebuilds the final `COPY`/sync layers (seconds). Reversed, every edit invalidates the dependency install (minutes) and the cache is useless.
21. The service name, `db` (Compose DNS). `localhost` inside a container is that container's own loopback interface — not the host, not other containers.
22. The `db` service must define a `healthcheck` (e.g. `pg_isready`); Compose waits until it reports healthy. Plain `depends_on` only orders container *start*; Postgres accepts connections a few seconds after start, so the API's first connection fails and it crashes.
23. `down` stops and removes containers and the default network but keeps named volumes (data survives). `down -v` also deletes the volumes — the database is gone.
24. 137 = 128 + 9: killed by SIGKILL, typically the OOM killer or Docker's post-SIGTERM timeout. 143 = 128 + 15: terminated by SIGTERM (normal stop, though a graceful app should exit 0 instead).
25. Store config — everything that varies between deploys — in environment variables, strictly separated from code. Violations: hard-coded `DATABASE_URL` or an API key in `settings.py`; a committed `.env`; `if hostname == "prod-server"` branches in code.

---



## Prepare for your Wednesday 1:1 and Friday demo

**Wednesday 1:1 (60 minutes, one intern at a time).** Bring your laptop. The whiteboard is for you to draw, so practise the drawings before you come. Be ready to:

1. **Show your machine works**: `ssh -T git@github.com`, `docker run --rm hello-world`, `uv --version`, `gh auth status`. Blockers get fixed on the spot, so bring them.
2. **Draw the Git model**: two blobs, a tree, two commits, `main`, `HEAD`, `origin/main`, and then what changes after `git commit`, `git switch -c feature`, and `git rebase main` from `feature`. Know what happens to the old commits and what the reflog records. Know what "ours" and "theirs" mean during a rebase.
3. **Walk through your** `git-conflict` **repo**: `git log --graph`, the reflog entries for the destructive reset and the recovery, and the `git bisect run` output. Be able to explain every decision you made while resolving the conflict, and what test would have caught the wrong resolution.
4. **Draw your Docker layers** in order, and say which are cached after editing `main.py`, after `uv add httpx`, and after changing `FROM`. Explain why the runtime stage has no `uv`.
5. **Draw the HTTP round trip** browser → reverse proxy (TLS) → container: which hops are encrypted, where CORS headers are added, where `X-Forwarded-Proto` is set, where a session cookie is checked. Know the status code for a bad JSON body, a missing token, a valid token from the wrong user, and a database that is down.
6. **Bring your retro**: what cost you the most time, what you want more or less of in week 2, and anything in this document that was wrong or unclear. Fixing course material is a valid PR.

**Friday demo (group, 60 minutes).** Each intern demos for 10 minutes (`docker compose up` from clean, hit `/db`, show the debugger pausing, answer one random "explain this line" question), then reviews one other PR live for 5 minutes. We finish by walking through the strongest week-1 PR line by line and discussing what made it good.

---



## Going deeper (optional)

- [Pro Git — 10.1 Plumbing and Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain): build a commit by hand with `hash-object`, `update-index`, `write-tree`, `commit-tree`. Ninety minutes that permanently demystify Git.
- [git-rebase reference](https://git-scm.com/docs/git-rebase): `--update-refs` for stacked branches, `--onto` for moving a branch between bases, `--exec` to run tests on every commit.
- [git-bisect reference](https://git-scm.com/docs/git-bisect): `bisect skip`, custom terms, bisecting with a script that returns 125 for "cannot test".
- [gitignore reference](https://git-scm.com/docs/gitignore) — precedence, negation, and global ignores (`core.excludesFile`) for editor junk.
- Learn Git Branching — finish the "Remote" world and "Moving Work Around".
- [uv — Concepts: Python versions](https://docs.astral.sh/uv/concepts/python-versions/) and [Managing dependencies](https://docs.astral.sh/uv/concepts/projects/dependencies/) — optional groups, sources (git/path deps), overriding constraints.
- [Ruff — rule catalog](https://docs.astral.sh/ruff/rules/): try enabling `RUF`, `PL`, `PERF` on `week1-py` and read every new warning.
- [Docker — Compose file reference](https://docs.docker.com/reference/compose-file/): `profiles`, `develop.watch` for live sync, `extends`, resource limits — you will use `profiles` and `watch` in week 9.
- [MDN — HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching): `Cache-Control` directives you will apply in week 10 (web) or to LLM response caching (AI).
- [DigitalOcean — Understanding Systemd Units and Unit Files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files): write a unit for a script, add a timer; useful when Compose is overkill.
- [Missing Semester — the rest of the 2020 lectures](https://missing.csail.mit.edu/2020/) (editors, debugging & profiling, metaprogramming, security).
- Julia Evans' zines *How Git Works*, *Bite Size Command Line*, *Bite Size Linux* — [wizardzines.com](https://wizardzines.com/zines/git/) — the fastest way to internalize the mental models in this document.

