# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Git is a save button that captures whatever my files look like right now, branches are copies of my project folder, `.gitignore` erases files from history, and managing dependencies is just running `pip install` whenever I need a package."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `git commit` captures the current state of all my files | `git commit` captures only what was **staged** via `git add` — the staging area is an independent snapshot |
| 2 | `.gitignore` removes already-tracked files from the repository | `.gitignore` only prevents **untracked** files from being staged; already-committed files remain tracked forever |
| 3 | A branch is a copy of the codebase; creating one doubles disk usage | A branch is a ~41-byte file containing a commit hash — a movable pointer, not a copy |
| 4 | Deleting a branch deletes the commits on it | Deleting a branch removes the pointer; commits reachable from other branches (e.g., via merge) survive intact |
| 5 | `pip install` and `uv add` are interchangeable inside a uv project | `pip install` doesn't update `pyproject.toml` or `uv.lock`; the dependency becomes invisible to the reproducibility pipeline |
| 6 | `uv.lock` is a manually-editable requirements file like `requirements.txt` | `uv.lock` is auto-generated from `pyproject.toml`; any manual edits are overwritten on the next `uv lock` or `uv add` |
| 7 | `python main.py` and `uv run python main.py` behave identically | `uv run` verifies the `.venv` is synced with `uv.lock` before executing; raw `python` uses whatever environment is active (or the system default) |

**The Prediction Gap:** When a student stages a file with `git add`, then edits the file *again* before committing, they expect `git commit` to capture the latest edit. The actual behavior is that the commit records only the snapshot that was staged — the post-staging edit remains uncommitted in the working directory. This immediately reveals whether the student has internalized the three-areas model or still thinks of `commit` as "save current state."

---

## The Main Purpose (Essence)

**Core Insight:** Git is a content-addressable snapshot DAG (directed acyclic graph) with cheap, movable pointer labels (branches), and uv creates a reproducible pipeline (`pyproject.toml` → `uv.lock` → `.venv`) that eliminates "works on my machine" by separating **intent** (what you need) from **resolution** (exact versions installed).

**The Problems Being Solved:**

| Tool | Without It | With It |
|------|-----------|---------|
| Git (staging area) | Every commit is all-or-nothing — entire working directory or nothing | Compose each commit deliberately: choose exactly which changes tell one story |
| Git (branches) | Experiment on the only copy; one mistake and everything breaks | Isolate experiments; discard failures or merge successes at zero risk |
| Git (remotes + PRs) | Email zip files; overwrite each other; no review | Synchronized collaboration with full history and a review gate |
| uv (pyproject.toml + uv.lock) | "Install these 47 packages manually at exact versions, good luck" | One command recreates the byte-identical environment on any machine |
| uv (uv run) | Activate the right venv and hope the packages are up to date | Always runs in the correct, synced environment — no activation, no guessing |

**The Key Distinctions:**
- **Staging area ≠ pre-commit holding pen.** It's an independent snapshot you're composing. It can diverge from both the working directory and the latest commit simultaneously.
- **`pyproject.toml` ≠ `uv.lock`.** `pyproject.toml` declares flexible intent ("I need httpx ≥ 0.28"); `uv.lock` records exact resolution ("httpx==0.28.1 + all transitive deps pinned"). One is the recipe card; the other is the exact shopping receipt.
- **`.gitignore` is a filter for `git add`, not a retroactive eraser.** It prevents future staging of untracked files. It does not untrack already-committed files.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "`git commit` saves current file state" | Editing after `git add` means commit has the edit | Commit records the **staged** version; the later edit is only in the working directory | Staging area holds its own snapshot, independent of the working directory |
| "`.gitignore` removes tracked files" | Adding `.env` to `.gitignore` hides it from Git history | `.env` remains tracked; old commits still contain it in full | `.gitignore` only filters **untracked** files from `git add` |
| "Deleting a branch deletes its commits" | `git branch -d feature` erases that line of work | Commits reachable via other branches (e.g., through a merge) are preserved | Branch = pointer; deleting a pointer ≠ deleting data |
| "Rebase = merge but cleaner" | Rebase on a shared branch is fine | Rebase rewrites commit hashes; pushing forces teammates to reconcile rewritten history | Rebase creates **new** commit objects; old hashes become orphaned |
| "`pip install` records the dependency" | Teammate can clone and run the project immediately | Teammate's `uv sync` doesn't install the pip-added package | pip doesn't modify `pyproject.toml` or `uv.lock` |
| "`uv run` is just `python` with extra steps" | `python main.py` and `uv run python main.py` are identical | `uv run` syncs `.venv` from `uv.lock` first; raw `python` uses whatever environment is active | `uv run` checks lockfile consistency before executing |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Phantom Edit (Staging Area ≠ Working Directory)

```bash
# Run these commands in order in a fresh directory
mkdir phantom && cd phantom
git init

echo 'print("Hello, World!")' > app.py
git add app.py

# *** MODIFY the file AFTER staging ***
echo 'print("Hello, Universe!")' > app.py

git commit -m "feat: add greeting"

# Now inspect:
cat app.py                 # Command A
git show HEAD:app.py       # Command B
git status                 # Command C
```

**Questions:**

1. **Predict the output** of Command A (`cat app.py`). What text is in the file on disk right now?

2. **Predict the output** of Command B (`git show HEAD:app.py`). What text was stored in the commit?

3. **Predict the output** of Command C (`git status`). Is the working directory clean? If not, what does Git report?

4. **Explain why** Commands A and B produce different results. A student claims: "I committed `app.py`, so the commit must have the latest version." Walk through the three-areas model to show exactly where each version lives after the commit.

5. **Mental Trace:** At each step below, state the contents of `app.py` in each of the three areas (Working Directory, Staging Area, Repository):
   - After `echo 'print("Hello, World!")' > app.py`
   - After `git add app.py`
   - After `echo 'print("Hello, Universe!")' > app.py`
   - After `git commit -m "feat: add greeting"`

---

## Exercise 1.2: The .gitignore That Came Too Late

```bash
mkdir secret-demo && cd secret-demo
git init

# Create a secrets file and commit it
echo "API_KEY=sk_live_abc123_super_secret" > .env
echo "DB_PASSWORD=p4ssw0rd_pr0d" >> .env
git add .
git commit -m "chore: initial project setup"

# Oh no! Realize we shouldn't have committed .env
# "Fix" it by adding .gitignore
echo ".env" > .gitignore
git add .gitignore
git commit -m "chore: add .gitignore to protect secrets"

# Verify our "fix" worked
git status
```

**Questions:**

1. **Predict:** After both commits, run `git status`. Does Git report `.env` as tracked, untracked, modified, or ignored? Why?

2. **Predict:** If you now run `echo "NEW_SECRET=xyz" >> .env` and then `git status`, does Git report the change to `.env`? Why?

3. **Prove the danger:** Write the exact Git command that would reveal the full contents of `.env` from commit history, even after `.gitignore` was added. (Hint: you can retrieve any file from any commit.)

4. **Explain:** A developer pushes this repository to GitHub and says, "The secrets are safe because `.env` is in `.gitignore`." Write a step-by-step argument for why they're wrong, referencing at least two different ways an attacker could access the secrets from the Git history.

5. **Fix (partial):** Write the command sequence that would:
   - Stop Git from tracking `.env` going forward (without deleting the file from disk)
   - Note: this still does NOT remove the secret from history. What additional tool or technique would be required?

---

## Exercise 1.3: The Branch Deletion Paradox

```bash
mkdir branch-demo && cd branch-demo
git init

echo "v1" > app.py
git add . && git commit -m "A: initial version"

echo "v2" > app.py
git add . && git commit -m "B: update to v2"

# Create feature branch and add work
git switch -c feature/experiment

echo "experiment code" > experiment.py
git add . && git commit -m "C: add experiment module"

echo "refined experiment" > experiment.py
git add . && git commit -m "D: refine experiment"

# Switch back, merge, then delete the branch
git switch main
git merge feature/experiment --no-ff -m "E: merge experiment into main"

git branch -d feature/experiment

# Inspect
git log --oneline --graph --all         # Command X
ls                                       # Command Y
```

**Explanation:**
- Without --no-ff (Fast-Forward): The history is flattened. You can't tell where the feature started or ended.
```
A---B---C---D---E (main)
```
- With --no-ff: The history preserves the branch structure.
```
      C---D---E (feature/experiment)
     /         \
A---B-----------F (main)  <-- F is the "Merge Commit"
```

**Questions:**

1. **Predict:** After `git branch -d feature/experiment`, how many commits exist in the repository? List each commit (A through E) and state whether it is still reachable.

2. **Predict:** What does `ls` (Command Y) show? Does `experiment.py` exist in the working directory? Why or why not?

3. **Predict:** Draw or describe the commit graph that `git log --oneline --graph --all` (Command X) would show. Include the position of the `main` pointer and `HEAD`.

4. **Alternative scenario:** Suppose instead of merging, you had run `git switch main` and then `git branch -D feature/experiment` (force delete, no merge). Would commits C and D survive? Why or why not? What Git mechanism eventually removes them?

5. **Explain:** A student says, "Branches are copies of the code, so deleting a branch deletes the code." Using the pointer model from the lecture, explain what *actually* happens when a branch is created, when it advances, and when it is deleted.

---

## Exercise 1.4: The Invisible Dependency

```bash
# === DEVELOPER ALICE (Day 1) ===

uv init weather-app && cd weather-app
uv add httpx

# Alice also wants 'rich' for formatted console output.
# She remembers pip from an old tutorial:
source .venv/bin/activate
pip install rich
python -c "from rich.console import Console; Console().print('[green]rich works![/green]')"
# Output: "rich works!"  (in green text)
deactivate

# Alice writes her application
cat > main.py << 'PYEOF'
import httpx
from rich.console import Console

console = Console()

def fetch_and_display(url: str) -> None:
    response = httpx.get(url)
    console.print(f"[green]Status:[/green] {response.status_code}")
    console.print(f"[blue]Length:[/blue] {len(response.content)} bytes")

if __name__ == "__main__":
    fetch_and_display("https://httpbin.org/get")
PYEOF

# Alice tests it — it works!
source .venv/bin/activate
python main.py
# Output:
#   Status: 200
#   Length: 407 bytes
deactivate

# Alice commits and pushes
git add .
git commit -m "feat: add fetch and display"
git push origin main
```

```bash
# === DEVELOPER BOB (Day 2) ===

git clone https://github.com/alice/weather-app.git
cd weather-app
uv sync
uv run python main.py       # <-- What happens?
```

**Questions:**

1. **Predict:** When Alice runs `python -c "from rich.console import Console; ..."` inside the activated `.venv`, does it work? Why?

2. **Inspect:** After Alice's `pip install rich`, examine `pyproject.toml`. Is `rich` listed under `[project] dependencies`? Examine `uv.lock`. Is `rich` included anywhere?

3. **Predict:** When Bob runs `uv sync`, what packages get installed into his `.venv`?

4. **Predict:** When Bob runs `uv run python main.py`, what is the exact error? On which line does it fail?

5. **Explain:** Alice protests: "But it works on my machine! I installed rich!" Explain the difference between what `pip install` does and what `uv add` does, referencing the `pyproject.toml → uv.lock → .venv` pipeline.

6. **Fix:** What single command should Alice have used instead of `pip install rich`?

---

# Level 1 Solutions

## Solution 1.1: The Phantom Edit

**1. Command A: `cat app.py`**
```
print("Hello, Universe!")
```
The working directory contains the *latest* edit.

**2. Command B: `git show HEAD:app.py`**
```
print("Hello, World!")
```
The commit recorded the *staged* version, not the working directory version.

**3. Command C: `git status`**
```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   app.py
```
Git reports `app.py` as **modified but not staged** — because the working directory version (`"Hello, Universe!"`) differs from the committed version (`"Hello, World!"`).

**4. Explain why:**
The naive model treats `git commit` as "save current files." The correct model is:

- `git add app.py` **copies** the file's contents at that moment into the staging area. This is a snapshot — a separate copy, independent of the working directory.
- When the file is overwritten with `"Hello, Universe!"`, only the **working directory** changes. The staging area still holds `"Hello, World!"`.
- `git commit` takes whatever is in the **staging area** (not the working directory) and stores it as a permanent snapshot in the repository.

After the commit:
- Working Directory: `"Hello, Universe!"`
- Staging Area: `"Hello, World!"` (now matches the commit)
- Repository (HEAD): `"Hello, World!"`

The working directory diverges from the repository because the post-staging edit was never staged.

**5. Mental Trace:**

| Step | Working Dir | Staging Area | Repository |
|------|------------|--------------|------------|
| After first `echo` | `"Hello, World!"` | (empty) | (empty) |
| After `git add app.py` | `"Hello, World!"` | `"Hello, World!"` | (empty) |
| After second `echo` | `"Hello, Universe!"` | `"Hello, World!"` | (empty) |
| After `git commit` | `"Hello, Universe!"` | `"Hello, World!"` | Commit 1: `"Hello, World!"` |

**The Key Insight:** The staging area is not a "pointer" to the working directory. It's a separate, independent snapshot. `git add` copies INTO it; `git commit` reads FROM it. The working directory can change freely without affecting what the next commit will contain.

---

## Solution 1.2: The .gitignore That Came Too Late

**1. `git status` shows a clean working tree:**
```
On branch main
nothing to commit, working tree clean
```
`.env` is **still tracked.** `.gitignore` only prevents *untracked* files from being staged. Since `.env` was already committed (it's in the repository), `.gitignore` has no effect on it. Git continues to track changes to `.env`.

**2. Yes, Git reports the change:**
```
On branch main
Changes not staged for commit:
        modified:   .env
```
Because `.env` is still tracked, Git notices the modification. `.gitignore` did nothing to stop this.

**3. Reveal the secret from history:**
```bash
git show HEAD~1:.env
```
Output:
```
API_KEY=sk_live_abc123_super_secret
DB_PASSWORD=p4ssw0rd_pr0d
```
The first commit (`HEAD~1`) contains the full `.env` file. Alternative commands: `git log -p -- .env` shows every change to `.env` across all commits.

**4. Why the developer is wrong:**

The secrets are exposed in at least three ways:
1. **Any commit in history** that includes `.env` can be inspected with `git show <hash>:.env`. Cloning the repo gives access to ALL history.
2. **`git log -p`** displays the full diff, showing exactly when `.env` was added and what it contained.
3. Even if `.env` were removed from the latest commit, **git reflog** and **dangling objects** may retain the data until garbage collection.

The fundamental issue: Git is an *append-only* history system. Adding `.gitignore` only affects future `git add` operations. It cannot reach back in time to erase committed content.

**5. Fix (partial):**
```bash
# Stop tracking .env (keep the file on disk)
git rm --cached .env
git commit -m "chore: remove .env from tracking"

# .env is now untracked AND in .gitignore → future git add . won't stage it
```

**But the secret is still in commits 1 and 2.** To purge it from history entirely, you would need a history-rewriting tool like `git filter-repo` or BFG Repo Cleaner — and you must treat the secret as **already compromised** (rotate the API key immediately).

---

## Solution 1.3: The Branch Deletion Paradox

**1. Five commits exist (A, B, C, D, E). All are reachable.**

The merge commit E has **two parents**: B (from main) and D (from feature). This means the entire chain A → B → C → D is reachable from E, and E is reachable from the `main` pointer. Deleting the `feature/experiment` pointer removed a label, not the commits.

**2. `ls` shows:**
```
app.py          experiment.py
```
`experiment.py` exists because commit E (the merge) incorporated all changes from the feature branch into main. The working directory reflects the `HEAD` commit, which is E.

**3. Commit graph:**
```
*   E: merge experiment into main  (HEAD -> main)
|\
| * D: refine experiment
| * C: add experiment module
|/
* B: update to v2
* A: initial version
```
`main` points to E. `HEAD` points to `main`. The `feature/experiment` label is gone, but the fork-and-merge shape is permanently recorded.

**4. Without merging, force delete:**
```bash
git switch main
git branch -D feature/experiment
```
Commits C and D become **unreachable** — no branch or tag points to them, and no merge commit links to them. They are "dangling objects." They still physically exist in `.git/objects/` temporarily, but `git log` won't show them. Git's garbage collector (`git gc`, which runs periodically) will eventually delete them permanently. Until then, you could recover them via `git reflog` or `git fsck --lost-found`.

**5. Explain — The Pointer Model:**

- **Creating a branch** (`git switch -c feature`): Creates a new file in `.git/refs/heads/` containing the current commit's hash. No code is copied. Cost: 41 bytes.
- **Advancing a branch** (committing on feature): The file in `.git/refs/heads/feature` is updated to contain the new commit's hash. The pointer moves forward.
- **Deleting a branch** (`git branch -d feature`): The file `.git/refs/heads/feature` is deleted. The commits it pointed to still exist in the object database. They're only at risk of garbage collection if **no other ref** (branch, tag, or merge) can reach them.

The student's mental model ("branches are copies") would predict that creating a branch doubles disk usage and deleting a branch halves it. In reality, creating a branch costs ~41 bytes, and deleting it saves ~41 bytes. The commits are shared.

---

## Solution 1.4: The Invisible Dependency

**1. Yes, it works.**
When Alice activates `.venv` and runs `pip install rich`, pip installs the `rich` package directly into `.venv/lib/python3.12/site-packages/`. Since the Python interpreter inside `.venv` looks for packages in that directory, the import succeeds.

**2. No, on both counts.**
- `pyproject.toml` still shows only `httpx` under dependencies. `pip install` does not modify `pyproject.toml`.
- `uv.lock` still shows only `httpx` and its transitive dependencies. `pip install` does not modify `uv.lock`.

`pip` is completely unaware of uv's project management files. It operates at the level of the environment, not the project.

**3. Bob's `uv sync` installs only what's in `uv.lock`:**
- httpx (and its transitive dependencies: httpcore, anyio, certifi, h11, idna, sniffio)
- That's it. `rich` is not in `uv.lock`, so it is not installed.

**4. Bob gets:**
```
Traceback (most recent call last):
  File "main.py", line 2, in <module>
    from rich.console import Console
ModuleNotFoundError: No module named 'rich'
```
It fails on **line 2** — the import of `rich`.

**5. The difference:**

| Action | Updates pyproject.toml? | Updates uv.lock? | Installs in .venv? | Teammate gets it? |
|--------|:---:|:---:|:---:|:---:|
| `pip install rich` | ❌ | ❌ | ✅ (current machine only) | ❌ |
| `uv add rich` | ✅ | ✅ | ✅ | ✅ (via `uv sync`) |

`pip install` operates on the *environment* — it puts packages in `.venv` but tells nobody about it. Since `.venv` is in `.gitignore` (correctly), this installation is invisible to Git, invisible to teammates, and will be overwritten if `uv sync` runs.

`uv add` operates on the *project* — it updates `pyproject.toml` (the recipe card), resolves all dependencies into `uv.lock` (the exact shopping list), and then installs into `.venv`. Since `pyproject.toml` and `uv.lock` are committed to Git, any teammate who runs `uv sync` gets the identical environment.

**6. The fix:**
```bash
uv add rich
```
One command. It updates `pyproject.toml`, resolves dependencies, updates `uv.lock`, and installs into `.venv`.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Secret That Won't Die

**Context:** A junior developer committed API credentials, realized the mistake, and attempted to fix it. The project runs fine and `git status` is clean. But there is a critical security flaw.

```bash
# What the developer did (reconstruct this to follow along):
mkdir api-project && cd api-project
git init

# Commit 1: Project with secrets (the mistake)
cat > config.py << 'EOF'
OPENAI_API_KEY = "sk-proj-abc123def456ghi789jkl012mno345"
DATABASE_URL = "postgresql://admin:Sup3rS3cret!@prod-db.company.com:5432/maindb"
DEBUG = True
EOF

cat > main.py << 'EOF'
from config import OPENAI_API_KEY, DATABASE_URL

def connect():
    print(f"Connecting to database...")
    # ... uses DATABASE_URL
    
def call_api():
    print(f"Calling OpenAI API...")
    # ... uses OPENAI_API_KEY

if __name__ == "__main__":
    connect()
    call_api()
EOF

git add .
git commit -m "feat: initial project with API integration"

# Commit 2: The "fix" — move secrets to .env
cat > .env << 'EOF'
OPENAI_API_KEY=sk-proj-abc123def456ghi789jkl012mno345
DATABASE_URL=postgresql://admin:Sup3rS3cret!@prod-db.company.com:5432/maindb
EOF

cat > config.py << 'EOF'
import os
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")
DATABASE_URL = os.getenv("DATABASE_URL", "")
DEBUG = True
EOF

echo ".env" > .gitignore

git add .
git commit -m "fix: move secrets to environment variables"

# Commit 3: Remove .env from tracking
git rm --cached .env
git commit -m "chore: stop tracking .env"

# Developer thinks: "All clean now!"
git status
# Output: nothing to commit, working tree clean
```

---

**Task 2a: Identify the Flaw**

The developer claims the secrets are safe. `git status` is clean, `.env` is in `.gitignore`, and the code reads from environment variables. The project runs correctly.

1. How many commits contain the plain-text secrets?
2. In which files and which commits?
3. Is this a runtime bug or a different category of flaw?

Do NOT fix the code yet. Just identify every location where secrets are exposed.

---

**Task 2b: Explain the Mechanism**

1. Why does adding `.env` to `.gitignore` in Commit 2 NOT prevent `.env` from being committed in that same commit?
2. Why does `git rm --cached .env` in Commit 3 NOT erase `.env` from Commits 1 and 2?
3. Explain what Git's append-only history model means for sensitive data that has ever been committed.

---

**Task 2c: Instrument to Prove Diagnosis**

Write the exact Git commands that prove the secrets are still accessible. For each command, show what output you expect:

```bash
# Command to show config.py from commit 1 (contains hardcoded secrets):
???

# Command to show .env from commit 2 (contains secrets in env file):
???

# Command to search entire history for the API key string:
???
```

---

**Task 2d: Fix the Code**

The secrets are already compromised in Git history. Describe a complete remediation plan:

1. **Immediate action** (not Git-related): What must be done before ANY Git cleanup?
2. **Git cleanup**: What command or tool removes the files from ALL historical commits? (Name the tool; implementing history rewriting is beyond this exercise.)
3. **Prevention**: Show the correct project setup that would have prevented this from the start — write the complete sequence of commands for initializing the project safely.

---

**Task 2e: Write Verification**

Write a shell script that a CI pipeline could run to verify that no secrets exist in the current Git history. The script should fail if any commit contains a string matching the pattern `sk-proj-` or `Sup3rS3cret`.

```bash
#!/bin/bash
# verify_no_secrets.sh
# Should exit 0 if clean, exit 1 if secrets found

# Your implementation here
???
```

---

## Exercise 2.2: The Dependency Ghost

**Context:** A developer's project runs perfectly on their machine. Every teammate who clones it gets a crash. The developer insists: "The code is correct — I just ran it five minutes ago."

```toml
# pyproject.toml
[project]
name = "news-aggregator"
version = "0.1.0"
description = "Fetch and display news from multiple sources"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[dependency-groups]
dev = [
    "pytest>=8.3.4",
]
```

```python
# main.py
import asyncio
import httpx
from rich.console import Console       # Line 3
from rich.table import Table           # Line 4

console = Console()

async def fetch_headlines(url: str) -> list[dict]:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json().get("articles", [])

async def display_news():
    urls = [
        "https://newsapi.example.com/v1/top",
        "https://newsapi.example.com/v1/tech",
    ]
    tasks = [fetch_headlines(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    table = Table(title="Headlines")
    table.add_column("Source", style="cyan")
    table.add_column("Title", style="green")
    
    for articles in results:
        if isinstance(articles, Exception):
            console.print(f"[red]Error: {articles}[/red]")
            continue
        for article in articles:
            table.add_row(article.get("source", "?"), article.get("title", "?"))
    
    console.print(table)

if __name__ == "__main__":
    asyncio.run(display_news())
```

```python
# tests/test_main.py
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_fetch_headlines_success():
    mock_response = AsyncMock()
    mock_response.json.return_value = {
        "articles": [{"source": "TechNews", "title": "Python 3.13 Released"}]
    }
    
    with patch("httpx.AsyncClient.get", return_value=mock_response):
        from main import fetch_headlines
        result = await fetch_headlines("https://fake.api/news")
        assert len(result) == 1
        assert result[0]["source"] == "TechNews"
```

**The developer runs on their machine:**
```bash
uv run python main.py     # Works! Beautiful table with headlines.
uv run pytest              # Works! 1 test passed.
```

**A teammate clones and runs:**
```bash
git clone <url> news-aggregator
cd news-aggregator
uv sync
uv run python main.py     # CRASH!
```

---

**Task 2a: Identify the Flaw**

1. The project runs and passes tests on the developer's machine. What specific error does the teammate see?
2. Which line of `main.py` causes the crash?
3. Compare the imports in `main.py` with the `[project] dependencies` in `pyproject.toml`. What's missing?
4. Why does it work on the developer's machine but not the teammate's?

---

**Task 2b: Explain the Mechanism**

1. How did `rich` end up importable on the developer's machine even though it's not in `pyproject.toml`? List at least two possible ways this could happen.
2. Why does `uv sync` on the teammate's machine NOT install `rich`?
3. Why did `uv run pytest` pass on the developer's machine even though the dependency is missing from the project?

---

**Task 2c: Instrument to Prove Diagnosis**

Show the commands that would reveal the problem WITHOUT running `main.py`:

```bash
# Command 1: List all packages declared in the project
???

# Command 2: Check if 'rich' is installed in the current .venv
???

# Command 3: Search all .py files for imports not covered by pyproject.toml
# (This could be a short Python script or shell command)
???
```

---

**Task 2d: Fix the Code**

1. Write the single command that properly adds the missing dependency.
2. After your fix, show what `pyproject.toml` should look like.
3. Verify that `uv.lock` was updated (describe what to look for, don't paste the full lockfile).

---

**Task 2e: Write Verification Test**

Write a script that verifies every `import` in the project is satisfied by packages declared in `pyproject.toml` or `uv.lock`:

```python
# verify_imports.py
"""
Verify that all imported third-party packages in .py files
are declared in pyproject.toml.

Run with: uv run python verify_imports.py
Should exit 0 if all imports are declared, exit 1 otherwise.
"""
import ast
import sys
import tomllib
from pathlib import Path

# Standard library modules to ignore (not exhaustive — add as needed)
STDLIB = {
    "asyncio", "os", "sys", "json", "pathlib", "typing",
    "unittest", "collections", "functools", "itertools",
    "dataclasses", "abc", "re", "math", "datetime",
    "pytest",  # dev dependency, handled separately
}

def get_declared_packages() -> set[str]:
    """Read pyproject.toml and return set of declared package names."""
    # Your implementation here
    ???

def get_imported_packages(filepath: Path) -> set[str]:
    """Parse a .py file and return set of top-level imported package names."""
    # Your implementation here
    ???

def main():
    # Your implementation here
    ???

if __name__ == "__main__":
    main()
```

---

# Level 2 Solutions

## Solution 2.1: The Secret That Won't Die

**Task 2a: Identify the Flaw**

1. **Two commits** contain plain-text secrets.
2. Locations:
   - **Commit 1** (`feat: initial project with API integration`): `config.py` contains hardcoded `OPENAI_API_KEY` and `DATABASE_URL` in plain text.
   - **Commit 2** (`fix: move secrets to environment variables`): `.env` was staged and committed *in the same commit* as `.gitignore`. The `.env` file was added via `git add .` before `.gitignore` could take effect for future operations.
3. This is a **security flaw**, not a runtime bug. The code runs correctly. The danger is that anyone with read access to the Git repository (including all of history) can extract production credentials.

**Task 2b: Explain the Mechanism**

1. In Commit 2, the developer ran `git add .` *after* writing both `.env` and `.gitignore`. At that point, `.env` already existed as an untracked file, and `git add .` staged it. `.gitignore` was also staged in the same operation. The `.gitignore` file only takes effect for **future** `git add` commands — it doesn't retroactively unstage files that were included in the same `git add .` invocation. (Technically, `.gitignore` prevents untracked files from being matched by glob patterns in `git add`. Since `.env` was explicitly included in `git add .`'s directory scan before the ignore rules were loaded for subsequent operations, it was staged.)

   More precisely: the `git add .` command processed all files in the working directory. At the time of processing, `.env` was untracked and not yet covered by a `.gitignore` rule (since `.gitignore` was also just a new file being staged). Both got staged together.

2. `git rm --cached .env` creates a **new commit** where `.env` is removed from the tracked files going forward. But Git is append-only: commits 1 and 2 are immutable snapshots. They still contain `.env` exactly as it was at that time. No subsequent commit can alter a past commit's contents — only history-rewriting operations can do that.

3. Git's history model means that every commit is an immutable snapshot identified by a cryptographic hash. "Removing" a file only means the *next* snapshot doesn't include it. All *previous* snapshots are permanent records. Sensitive data in any commit — even an old one — is accessible to anyone who can clone the repository.

**Task 2c: Instrument to Prove Diagnosis**

```bash
# Show config.py from commit 1 (contains hardcoded secrets):
git show HEAD~2:config.py
# Output includes: OPENAI_API_KEY = "sk-proj-abc123def456ghi789jkl012mno345"

# Show .env from commit 2 (contains secrets in .env file):
git show HEAD~1:.env
# Output includes: OPENAI_API_KEY=sk-proj-abc123def456ghi789jkl012mno345

# Search entire history for the API key pattern:
git log -p --all -S "sk-proj-abc123" -- .
# Shows every commit that added or removed a line containing that string
```

**Task 2d: Fix the Code**

1. **Immediate action: ROTATE ALL CREDENTIALS.** Generate a new OpenAI API key and a new database password. The old ones must be treated as publicly known. This must happen *before* any Git cleanup, because the cleanup takes time and the secrets may already be cached by Git hosting services.

2. **Git cleanup:** Use `git filter-repo` (recommended) or BFG Repo Cleaner to rewrite all commits, removing `config.py`'s old contents and `.env` from every historical snapshot. This rewrites commit hashes, so all collaborators must re-clone.

   ```bash
   # Using git-filter-repo (install separately):
   git filter-repo --path .env --invert-paths
   git filter-repo --blob-callback '
     if b".env" in blob.data or b"sk-proj-" in blob.data:
       blob.data = b""
   '
   ```

3. **Prevention — the correct sequence from the start:**
   ```bash
   uv init api-project && cd api-project
   
   # FIRST: Set up .gitignore BEFORE creating any secret files
   cat >> .gitignore << 'EOF'
   .env
   .env.local
   .env.production
   EOF
   
   git add .gitignore
   git commit -m "chore: add .gitignore with secret file patterns"
   
   # NOW create .env (it won't be staged because .gitignore already excludes it)
   cat > .env << 'EOF'
   OPENAI_API_KEY=sk-proj-your-key-here
   DATABASE_URL=postgresql://user:pass@localhost:5432/devdb
   EOF
   
   # Create .env.example (safe to commit — no real secrets)
   cat > .env.example << 'EOF'
   OPENAI_API_KEY=sk-proj-REPLACE_ME
   DATABASE_URL=postgresql://user:password@localhost:5432/dbname
   EOF
   
   git add .env.example
   git commit -m "docs: add .env.example template"
   
   # Verify .env is NOT tracked:
   git status  # .env should not appear at all
   ```

**Task 2e: Verification Script**

```bash
#!/bin/bash
# verify_no_secrets.sh

FAILED=0

# Search all commits for secret patterns
PATTERNS=("sk-proj-" "Sup3rS3cret" "password" "secret" "api_key=sk")

for pattern in "${PATTERNS[@]}"; do
    RESULT=$(git log -p --all -S "$pattern" -- . 2>/dev/null)
    if [ -n "$RESULT" ]; then
        echo "FAIL: Found pattern '$pattern' in git history"
        FAILED=1
    fi
done

# Check that .env is not currently tracked
if git ls-files --error-unmatch .env 2>/dev/null; then
    echo "FAIL: .env is currently tracked by Git"
    FAILED=1
fi

# Check that .gitignore contains .env
if ! grep -q "^\.env$" .gitignore 2>/dev/null; then
    echo "FAIL: .gitignore does not contain '.env'"
    FAILED=1
fi

if [ $FAILED -eq 0 ]; then
    echo "PASS: No secrets found in repository history"
    exit 0
else
    exit 1
fi
```

---

## Solution 2.2: The Dependency Ghost

**Task 2a: Identify the Flaw**

1. The teammate sees:
   ```
   Traceback (most recent call last):
     File "main.py", line 3, in <module>
       from rich.console import Console
   ModuleNotFoundError: No module named 'rich'
   ```
2. **Line 3** (`from rich.console import Console`).
3. `main.py` imports `httpx` and `rich`. `pyproject.toml` declares only `httpx`. `rich` is missing from the project's dependency declaration.
4. The developer's `.venv` (or system Python) happened to have `rich` installed from a previous project or manual `pip install`. Since `.venv` is not committed (correctly), the teammate's freshly-created `.venv` only contains what `uv.lock` specifies.

**Task 2b: Explain the Mechanism**

1. `rich` could be importable on the developer's machine via several paths:
   - **Previous `pip install`:** The developer ran `pip install rich` (with `.venv` activated) for a previous experiment and never removed it.
   - **System-wide installation:** `rich` is installed in the system Python's site-packages from another project.
   - **Cross-project contamination:** If the developer accidentally used a different project's `.venv` or ran commands outside `uv run`.

2. `uv sync` reads `uv.lock` and installs exactly those packages into `.venv`. Since `rich` does not appear in `uv.lock` (because it was never added to `pyproject.toml` via `uv add`), `uv sync` does not install it.

3. `uv run pytest` passed because the test (`test_fetch_headlines_success`) only tests `fetch_headlines`, which imports `httpx` — not `rich`. The test file itself imports `pytest`, `unittest.mock`, and `main.fetch_headlines`. The test never exercises `display_news()` or any code path that uses `Console` or `Table`. The import at the **top of `main.py`** (`from rich.console import Console`) *does* execute when `from main import fetch_headlines` runs, but on the developer's machine, `rich` was available, so it didn't fail.

   On the teammate's machine, even `uv run pytest` would **also crash** — because importing `main` triggers line 3.

**Task 2c: Instrument to Prove Diagnosis**

```bash
# Command 1: List declared dependencies
cat pyproject.toml | grep -A 20 "dependencies"
# Shows: "httpx>=0.28.1" — no rich

# Command 2: Check if 'rich' is installed in .venv
uv run pip list | grep rich
# Output: (nothing)  — rich is not installed

# Or more directly:
uv run python -c "import rich" 2>&1
# Output: ModuleNotFoundError: No module named 'rich'

# Command 3: Find all third-party imports not in pyproject.toml
grep -rn "^import \|^from " *.py tests/*.py | grep -v "asyncio\|os\|sys\|typing\|pathlib\|unittest\|pytest\|__future__"
# This shows: httpx (declared ✅), rich (NOT declared ❌)
```

**Task 2d: Fix the Code**

1. The fix:
   ```bash
   uv add rich
   ```

2. Updated `pyproject.toml`:
   ```toml
   [project]
   name = "news-aggregator"
   version = "0.1.0"
   description = "Fetch and display news from multiple sources"
   readme = "README.md"
   requires-python = ">=3.12"
   dependencies = [
       "httpx>=0.28.1",
       "rich>=13.9.4",
   ]
   
   [dependency-groups]
   dev = [
       "pytest>=8.3.4",
   ]
   ```

3. Verify `uv.lock` was updated: Search for `rich` in `uv.lock`. You should find an entry with the exact resolved version of `rich` and its transitive dependencies (e.g., `markdown-it-py`, `pygments`, `mdurl`).

   ```bash
   grep "name = \"rich\"" uv.lock
   # Output: name = "rich"   ← Confirms it's now in the lockfile
   ```

**Task 2e: Verification Script**

```python
# verify_imports.py
import ast
import sys
import tomllib
from pathlib import Path

STDLIB = {
    "asyncio", "os", "sys", "json", "pathlib", "typing",
    "unittest", "collections", "functools", "itertools",
    "dataclasses", "abc", "re", "math", "datetime",
    "contextlib", "io", "time", "logging", "hashlib",
    "copy", "enum", "textwrap", "string", "struct",
}

# Mapping: import name → pyproject.toml package name (when they differ)
IMPORT_TO_PACKAGE = {
    "bs4": "beautifulsoup4",
    "cv2": "opencv-python",
    "PIL": "pillow",
    "yaml": "pyyaml",
    "sklearn": "scikit-learn",
    "attr": "attrs",
    "dotenv": "python-dotenv",
}

def get_declared_packages() -> set[str]:
    """Read pyproject.toml and return set of declared package names (lowercased)."""
    with open("pyproject.toml", "rb") as f:
        config = tomllib.load(f)

    packages = set()
    for dep in config.get("project", {}).get("dependencies", []):
        # Extract package name from version specifier (e.g., "httpx>=0.28" → "httpx")
        name = dep.split(">")[0].split("<")[0].split("=")[0].split("!")[0].strip()
        packages.add(name.lower().replace("-", "_"))

    for dep in config.get("dependency-groups", {}).get("dev", []):
        name = dep.split(">")[0].split("<")[0].split("=")[0].split("!")[0].strip()
        packages.add(name.lower().replace("-", "_"))

    return packages


def get_imported_packages(filepath: Path) -> set[str]:
    """Parse a .py file and return set of top-level imported package names."""
    source = filepath.read_text()
    try:
        tree = ast.parse(source)
    except SyntaxError:
        print(f"WARNING: Could not parse {filepath}")
        return set()

    imports = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                top_level = alias.name.split(".")[0]
                imports.add(top_level)
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                top_level = node.module.split(".")[0]
                imports.add(top_level)

    return imports


def main():
    declared = get_declared_packages()
    # Also treat the project itself as "declared"
    with open("pyproject.toml", "rb") as f:
        config = tomllib.load(f)
    project_name = config.get("project", {}).get("name", "").lower().replace("-", "_")
    declared.add(project_name)

    py_files = list(Path(".").rglob("*.py"))
    undeclared = {}

    for filepath in py_files:
        imports = get_imported_packages(filepath)
        for imp in imports:
            if imp in STDLIB:
                continue
            # Map import name to package name
            pkg_name = IMPORT_TO_PACKAGE.get(imp, imp).lower().replace("-", "_")
            if pkg_name not in declared and imp.lower() not in declared:
                if imp not in undeclared:
                    undeclared[imp] = []
                undeclared[imp].append(str(filepath))

    if undeclared:
        print("FAIL: The following imports are not declared in pyproject.toml:")
        for imp, files in sorted(undeclared.items()):
            print(f"  '{imp}' imported in: {', '.join(files)}")
        sys.exit(1)
    else:
        print("PASS: All imports are covered by declared dependencies.")
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

# LEVEL 2.5: EXTEND EXISTING CODE (The Integration)

## The Collaborative Feature Pipeline

**Given:** A working, correctly-set-up Python project with the following state.

```
weather-cli/
├── .git/                   (initialized, 2 commits on main)
├── .gitignore              (Python defaults, .env, .venv)
├── .python-version         (3.12)
├── pyproject.toml
├── uv.lock
├── main.py
└── tests/
    └── test_main.py
```

**pyproject.toml:**
```toml
[project]
name = "weather-cli"
version = "0.1.0"
description = "Async weather data fetcher"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[dependency-groups]
dev = [
    "pytest>=8.3.4",
    "pytest-asyncio>=0.25.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**main.py:**
```python
import asyncio
import httpx

async def fetch_weather(city: str) -> dict:
    """Fetch weather data for a given city."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.weatherapi.com/v1/current.json",
            params={"key": "demo", "q": city}
        )
        response.raise_for_status()
        return response.json()

async def fetch_multiple(cities: list[str]) -> list[dict]:
    """Fetch weather for multiple cities concurrently."""
    tasks = [fetch_weather(city) for city in cities]
    return await asyncio.gather(*tasks, return_exceptions=True)

if __name__ == "__main__":
    results = asyncio.run(fetch_multiple(["London", "Tokyo", "Sydney"]))
    for r in results:
        if isinstance(r, Exception):
            print(f"Error: {r}")
        else:
            print(f"{r.get('location', {}).get('name', '?')}: "
                  f"{r.get('current', {}).get('temp_c', '?')}°C")
```

**tests/test_main.py:**
```python
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from main import fetch_weather, fetch_multiple

@pytest.fixture
def mock_weather_response():
    mock_resp = MagicMock()
    mock_resp.json.return_value = {
        "location": {"name": "London"},
        "current": {"temp_c": 15.0}
    }
    mock_resp.raise_for_status = MagicMock()
    return mock_resp

@pytest.mark.asyncio
async def test_fetch_weather_returns_dict(mock_weather_response):
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock, return_value=mock_weather_response):
        result = await fetch_weather("London")
        assert result["location"]["name"] == "London"
        assert result["current"]["temp_c"] == 15.0

@pytest.mark.asyncio
async def test_fetch_multiple_returns_list(mock_weather_response):
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock, return_value=mock_weather_response):
        results = await fetch_multiple(["London", "Tokyo"])
        assert len(results) == 2
        assert all(isinstance(r, dict) for r in results)

@pytest.mark.asyncio
async def test_fetch_weather_handles_http_error():
    mock_resp = MagicMock()
    mock_resp.raise_for_status.side_effect = httpx.HTTPStatusError(
        "Not Found", request=MagicMock(), response=MagicMock(status_code=404)
    )
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock, return_value=mock_resp):
        results = await fetch_multiple(["InvalidCity"])
        assert len(results) == 1
        assert isinstance(results[0], Exception)

import httpx  # needed for the error class reference
```

**Git log:**
```
b4c5d6e (HEAD -> main) test: add unit tests for weather fetcher
a1b2c3d feat: implement async weather fetcher
```

**Setup instructions:** To create this starting state, run the following:
```bash
uv init weather-cli && cd weather-cli
uv add httpx
uv add --dev pytest pytest-asyncio
# (create the files above)
git add .
git commit -m "feat: implement async weather fetcher"
# (create tests/test_main.py)
git add .
git commit -m "test: add unit tests for weather fetcher"
```

**Verify starting state:**
```bash
uv run pytest       # All 3 tests pass
git status          # Clean working tree
```

---

### Extension 1: Add Formatted Output (Dependency Management + Conventional Commits)

**Task:** Add the `rich` library to produce a formatted table when displaying weather results.

**Requirements:**
1. Add `rich` as a **production** dependency (not dev) using the correct uv command.
2. Create a new function `display_results(results: list[dict]) -> None` in `main.py` that uses `rich.table.Table` to format output.
3. **The existing tests must still pass unchanged.** Your new function is additive — don't modify `fetch_weather` or `fetch_multiple`.
4. Commit with a proper conventional commit message.
5. After your commit, `pyproject.toml` should list `rich`, and `uv.lock` should include `rich` and its transitive dependencies.

**Verification:**
```bash
uv run pytest                    # All 3 original tests still pass
grep "rich" pyproject.toml       # rich appears in [project] dependencies
grep 'name = "rich"' uv.lock    # rich appears in lockfile
git log --oneline -1             # Latest commit follows conventional format
```

---

### Extension 2: Feature Branch Workflow (Branching + Merge)

**Task:** Add an environment-variable-based configuration system on a feature branch, then merge it.

**Requirements:**
1. Create and switch to a branch named `feature/env-config`.
2. Create a file `config.py` that reads `WEATHER_API_KEY` from the environment (using `os.getenv` with a default of `"demo"`).
3. Create a `.env.example` file showing the required variables.
4. **Ensure `.env` is in `.gitignore`** (verify it was already there from the starting state).
5. Modify `fetch_weather` to accept an `api_key` parameter instead of hardcoding `"demo"`.
6. Update the existing tests to pass the api_key parameter.
7. Make **at least two** focused conventional commits on the feature branch (e.g., one for `config.py`, one for the `main.py` modification).
8. Switch to main and merge with `--no-ff` to create an explicit merge commit.
9. All tests must pass after the merge.

**Verification:**
```bash
uv run pytest                                    # All tests pass
git log --oneline --graph | head -10             # Shows merge commit with branch shape
test -f config.py                                # config.py exists
test -f .env.example                             # .env.example exists
git ls-files .env | wc -l | grep "^0$"          # .env is NOT tracked
```

---

### Extension 3: Handle Merge Conflict (Conflict Resolution)

**Task:** Simulate and resolve a merge conflict.

**Requirements:**
1. From `main`, create a branch `feature/add-news-fetch`.
2. On this branch, add a function `fetch_news(topic: str) -> dict` to `main.py`. Add it directly **below** the `fetch_weather` function. Commit.
3. Switch back to `main`. **Without merging the feature branch**, edit the same area of `main.py` — add a docstring or comment directly below `fetch_weather`. Commit on `main`.
4. Now merge `feature/add-news-fetch` into `main`. A conflict will occur.
5. Resolve the conflict: keep BOTH changes (the comment/docstring AND the `fetch_news` function).
6. Complete the merge commit.
7. All previous tests must still pass. Add one new test for `fetch_news`.

**Verification:**
```bash
uv run pytest                                    # All tests pass (old + new)
git log --oneline --graph | head -15             # Shows merge with conflict resolution
grep "fetch_news" main.py                        # fetch_news function exists
grep "fetch_weather" main.py                     # fetch_weather still exists
```

---

# Level 2.5 Solutions / Guidelines

## Extension 1 Solution

```bash
# Step 1: Add rich as production dependency
uv add rich

# Step 2: Modify main.py — add display_results function (additive only)
```

Add to `main.py` (below existing code, above `if __name__`):
```python
from rich.table import Table
from rich.console import Console

def display_results(results: list[dict]) -> None:
    """Display weather results in a formatted table."""
    console = Console()
    table = Table(title="Weather Report")
    table.add_column("City", style="cyan")
    table.add_column("Temperature (°C)", style="green")

    for r in results:
        if isinstance(r, Exception):
            table.add_row("Error", f"[red]{r}[/red]")
        else:
            city = r.get("location", {}).get("name", "Unknown")
            temp = str(r.get("current", {}).get("temp_c", "N/A"))
            table.add_row(city, temp)

    console.print(table)
```

```bash
# Step 3: Verify tests still pass
uv run pytest

# Step 4: Commit
git add pyproject.toml uv.lock main.py
git commit -m "feat(display): add rich table output for weather results"
```

**Key lesson:** `uv add rich` updates three things atomically — `pyproject.toml`, `uv.lock`, and `.venv`. A `pip install` would have updated only `.venv`, making the dependency invisible to teammates.

---

## Extension 2 Solution

```bash
# Step 1: Create feature branch
git switch -c feature/env-config

# Step 2: Create config.py
cat > config.py << 'EOF'
import os

WEATHER_API_KEY: str = os.getenv("WEATHER_API_KEY", "demo")
EOF

# Step 3: Create .env.example
cat > .env.example << 'EOF'
# Weather API Configuration
# Get your key at: https://www.weatherapi.com/
WEATHER_API_KEY=your_api_key_here
EOF

# Step 4: First commit (config infrastructure)
git add config.py .env.example
git commit -m "feat(config): add environment-based API key configuration"

# Step 5: Modify main.py to use config
# Update fetch_weather signature and import config
# Step 6: Update tests to pass api_key

# Step 7: Second commit
git add main.py tests/test_main.py
git commit -m "refactor(weather): use configurable API key in fetch_weather"

# Step 8: Merge
git switch main
git merge feature/env-config --no-ff -m "feat: merge environment config support"

# Step 9: Verify
uv run pytest
```

**Key lesson:** The `--no-ff` flag forces a merge commit even when fast-forward is possible. This preserves the branch topology in the log, making it clear that these commits were developed as a unit on a feature branch.

---

## Extension 3 Solution

The conflict will look something like:
```
<<<<<<< HEAD
# Note: This function connects to the WeatherAPI service.
=======
async def fetch_news(topic: str) -> dict:
    """Fetch news articles about a topic."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://newsapi.example.com/v1/search",
            params={"q": topic}
        )
        response.raise_for_status()
        return response.json()
>>>>>>> feature/add-news-fetch
```

**Resolution:** Keep both — the comment AND the function. Delete the `<<<<<<<`, `=======`, and `>>>>>>>` markers.

```bash
# After editing the file to resolve the conflict:
git add main.py
git commit   # Git auto-populates the merge commit message

# Add test for fetch_news
# ... write test ...
git add tests/test_main.py
git commit -m "test(news): add unit test for fetch_news"
```

**Key lesson:** Conflicts are not errors. They are Git saying: "Two humans changed the same region, and I don't know which change to keep." The developer's job is to combine both changes meaningfully. Reading conflict markers is a fundamental skill.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS (The Application)

## The Team Project Bootstrap

**Scenario:** You are the tech lead bootstrapping a new Python backend project. Three developers (including you) will work on it simultaneously starting tomorrow. You must set up the project so that it meets strict requirements from day one.

**The Project:** An async data aggregator that fetches from multiple APIs and stores results. (The business logic is not the focus — the **project infrastructure** is.)

**Constraints:**

1. **One-Command Setup:** Any developer must be able to clone the repository and have a fully working environment with a SINGLE command (`uv sync` or `uv run`). No manual package installation, no "also install X," no activation steps.

2. **Python Version Pinned:** The project must enforce Python 3.12. If a developer has a different version active, the tooling must tell them.

3. **Zero Secrets in History:** The project requires an API key (stored as `DATA_API_KEY`). At no point in the Git history should any real or placeholder key appear in a tracked file. A `.env.example` template must exist.

4. **Dependency Segregation:** Production dependencies (`httpx`) and development dependencies (`pytest`, `pytest-asyncio`, `mypy`) must be separated. A production deployment should not need test tools.

5. **Branch Protection:** The `main` branch must contain only merge commits from feature branches. Direct commits to `main` are not allowed (enforced by convention in this exercise).

6. **Commit Discipline:** Every commit must follow the conventional commit format. The git log, read from top to bottom, should tell a coherent story of how the project was set up.

7. **Minimum Structure:** The project must contain at least:
   - `main.py` — entry point that imports from `config.py` and uses `httpx`
   - `config.py` — reads configuration from environment variables
   - `tests/test_main.py` — at least one passing test
   - Proper project metadata in `pyproject.toml`

---

### Task 3a: Bootstrap the Project (Pass basic verification)

Set up the project from scratch. Every command you run should be intentional. The verification script below must pass.

**Verification Script (save as `verify_project.py`, run with `uv run python verify_project.py`):**

```python
#!/usr/bin/env python3
"""Project setup verification script.
Run with: uv run python verify_project.py
All checks must pass."""

import subprocess
import tomllib
import sys
from pathlib import Path


def run(cmd: str) -> subprocess.CompletedProcess:
    return subprocess.run(cmd, shell=True, capture_output=True, text=True)


def check(condition: bool, msg: str) -> bool:
    status = "✅" if condition else "❌"
    print(f"  {status} {msg}")
    return condition


def main() -> None:
    passed = 0
    failed = 0

    print("\n=== PROJECT STRUCTURE ===")

    for f in ["pyproject.toml", "uv.lock", ".python-version",
              ".gitignore", ".env.example", "main.py", "config.py",
              "tests/test_main.py"]:
        if check(Path(f).exists(), f"File exists: {f}"):
            passed += 1
        else:
            failed += 1

    print("\n=== PYPROJECT.TOML ===")

    try:
        with open("pyproject.toml", "rb") as f:
            config = tomllib.load(f)

        deps = " ".join(config.get("project", {}).get("dependencies", []))
        if check("httpx" in deps, "httpx in production dependencies"):
            passed += 1
        else:
            failed += 1

        dev_deps = str(config.get("dependency-groups", {}).get("dev", []))
        for pkg in ["pytest", "mypy"]:
            if check(pkg in dev_deps, f"{pkg} in dev dependencies"):
                passed += 1
            else:
                failed += 1

        py_req = config.get("project", {}).get("requires-python", "")
        if check("3.12" in py_req, f"Python 3.12 required (got: {py_req})"):
            passed += 1
        else:
            failed += 1

    except Exception as e:
        print(f"  ❌ Could not parse pyproject.toml: {e}")
        failed += 1

    print("\n=== PYTHON VERSION ===")

    if Path(".python-version").exists():
        ver = Path(".python-version").read_text().strip()
        if check(ver.startswith("3.12"), f".python-version is {ver}"):
            passed += 1
        else:
            failed += 1
    else:
        failed += 1

    print("\n=== SECRETS SAFETY ===")

    gitignore = Path(".gitignore").read_text() if Path(".gitignore").exists() else ""
    if check(".env" in gitignore, ".env is in .gitignore"):
        passed += 1
    else:
        failed += 1

    tracked_env = run("git ls-files .env").stdout.strip()
    if check(tracked_env == "", ".env is NOT tracked by Git"):
        passed += 1
    else:
        failed += 1

    history_check = run('git log -p --all -S "DATA_API_KEY=" -- . | grep -c "DATA_API_KEY="').stdout.strip()
    # .env.example should have the variable name but no real value
    # We check that no commit introduces a line with a real-looking key
    suspicious = run('git log -p --all -- . | grep -E "DATA_API_KEY=.{10,}"').stdout.strip()
    if check(suspicious == "", "No real API key values in Git history"):
        passed += 1
    else:
        failed += 1

    if check(Path(".env.example").exists(), ".env.example exists"):
        passed += 1
    else:
        failed += 1

    print("\n=== DEPENDENCY SEGREGATION ===")

    prod_deps = " ".join(config.get("project", {}).get("dependencies", []))
    if check("pytest" not in prod_deps and "mypy" not in prod_deps,
             "pytest/mypy are NOT in production dependencies"):
        passed += 1
    else:
        failed += 1

    print("\n=== GIT WORKFLOW ===")

    log_output = run("git log --oneline --first-parent main").stdout.strip()
    commits = [line for line in log_output.split("\n") if line.strip()]

    if check(len(commits) >= 3, f"At least 3 commits on main (found {len(commits)})"):
        passed += 1
    else:
        failed += 1

    # Check conventional commit format
    prefixes = ("feat", "fix", "docs", "test", "refactor", "chore", "style")
    bad_commits = []
    for c in commits:
        parts = c.split(" ", 1)
        if len(parts) == 2:
            msg = parts[1]
            # Allow merge commits
            if not msg.startswith("Merge") and not any(msg.startswith(p) for p in prefixes):
                bad_commits.append(msg)

    if check(len(bad_commits) == 0,
             f"All commits follow conventional format (bad: {bad_commits[:3]})"):
        passed += 1
    else:
        failed += 1

    # Check for feature branch usage (merge commits exist)
    merge_log = run("git log --oneline --merges main").stdout.strip()
    has_merges = len(merge_log.strip()) > 0 if merge_log else False
    if check(has_merges, "At least one merge commit exists (feature branch workflow)"):
        passed += 1
    else:
        failed += 1

    print("\n=== TESTS ===")

    test_result = run("uv run pytest --tb=short -q")
    tests_pass = test_result.returncode == 0
    if check(tests_pass, f"All tests pass (uv run pytest)"):
        passed += 1
    else:
        failed += 1
        if test_result.stdout:
            print(f"    stdout: {test_result.stdout[:200]}")
        if test_result.stderr:
            print(f"    stderr: {test_result.stderr[:200]}")

    print("\n=== TYPE CHECK ===")

    mypy_result = run("uv run mypy main.py config.py")
    mypy_pass = mypy_result.returncode == 0
    if check(mypy_pass, "mypy passes on main.py and config.py"):
        passed += 1
    else:
        failed += 1
        if mypy_result.stdout:
            print(f"    {mypy_result.stdout[:200]}")

    print(f"\n{'='*50}")
    print(f"Results: {passed} passed, {failed} failed")
    if failed == 0:
        print("🎉 ALL CHECKS PASSED!")
        sys.exit(0)
    else:
        print("⚠️  Some checks failed. Review and fix.")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

### Task 3b: Implement Using Feature Branches

Your git history must demonstrate proper branch workflow. Here is the minimum expected flow:

1. Initialize the project on `main` (one initial commit with infrastructure: `pyproject.toml`, `.gitignore`, `.python-version`).
2. Feature branch `feature/core-config` → add `config.py`, `.env.example` → merge to main.
3. Feature branch `feature/data-fetcher` → add `main.py` with async fetch logic → merge to main.
4. Feature branch `feature/tests` → add `tests/test_main.py` → merge to main.

Each feature branch must have at least one focused commit before merging.

---

### Task 3c: Extend With a New Requirement

A fourth developer joins the team. They need to add `rich` for display formatting.

1. They clone the repo (simulate: `cd .. && git clone weather-aggregator weather-aggregator-dev4 && cd weather-aggregator-dev4`).
2. They run a single command to get the full environment.
3. They create a feature branch, add `rich` as a dependency, create `display.py`, and commit.
4. They push their branch and describe what their PR description would contain.
5. After merging, the verification script must still pass.

**Additional verification:**
```bash
# After the fourth developer's merge:
grep "rich" pyproject.toml       # rich is declared
uv run python -c "import rich"   # rich is importable
uv run pytest                    # all tests pass
```

---

# Level 3 Solutions

The exact code will vary by student, but the following command sequence represents a correct solution:

```bash
# === STEP 1: Initialize on main ===
uv init weather-aggregator && cd weather-aggregator
uv add httpx
uv add --dev pytest pytest-asyncio mypy
uv python pin 3.12

# Set up .gitignore properly FIRST
cat >> .gitignore << 'EOF'
.env
.env.local
.env.production
EOF

git add pyproject.toml uv.lock .python-version .gitignore README.md
git commit -m "chore: initialize project with dependencies and tooling"

# === STEP 2: Feature branch for config ===
git switch -c feature/core-config

cat > config.py << 'PYEOF'
import os

DATA_API_KEY: str = os.getenv("DATA_API_KEY", "")
BASE_URL: str = os.getenv("BASE_URL", "https://api.example.com")
PYEOF

cat > .env.example << 'EOF'
# Copy this file to .env and fill in real values
DATA_API_KEY=
BASE_URL=https://api.example.com
EOF

git add config.py .env.example
git commit -m "feat(config): add environment-based configuration"

git switch main
git merge feature/core-config --no-ff -m "feat: merge environment configuration"
git branch -d feature/core-config

# === STEP 3: Feature branch for data fetcher ===
git switch -c feature/data-fetcher

cat > main.py << 'PYEOF'
import asyncio
import httpx
from config import DATA_API_KEY, BASE_URL

async def fetch_data(endpoint: str) -> dict:
    """Fetch data from the configured API."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{BASE_URL}/{endpoint}",
            headers={"Authorization": f"Bearer {DATA_API_KEY}"}
        )
        response.raise_for_status()
        return response.json()

async def fetch_multiple(endpoints: list[str]) -> list[dict]:
    """Fetch from multiple endpoints concurrently."""
    tasks = [fetch_data(ep) for ep in endpoints]
    return await asyncio.gather(*tasks, return_exceptions=True)

if __name__ == "__main__":
    results = asyncio.run(fetch_multiple(["data/latest", "data/summary"]))
    for r in results:
        if isinstance(r, Exception):
            print(f"Error: {r}")
        else:
            print(r)
PYEOF

git add main.py
git commit -m "feat(fetcher): add async data fetcher with concurrent support"

git switch main
git merge feature/data-fetcher --no-ff -m "feat: merge async data fetcher"
git branch -d feature/data-fetcher

# === STEP 4: Feature branch for tests ===
git switch -c feature/tests

mkdir -p tests
cat > tests/__init__.py << 'EOF'
EOF

cat > tests/test_main.py << 'PYEOF'
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
import httpx
from main import fetch_data, fetch_multiple

@pytest.fixture
def mock_success_response():
    mock_resp = MagicMock()
    mock_resp.json.return_value = {"status": "ok", "data": [1, 2, 3]}
    mock_resp.raise_for_status = MagicMock()
    return mock_resp

@pytest.mark.asyncio
async def test_fetch_data_returns_dict(mock_success_response):
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock, 
               return_value=mock_success_response):
        result = await fetch_data("data/latest")
        assert isinstance(result, dict)
        assert result["status"] == "ok"

@pytest.mark.asyncio
async def test_fetch_multiple_concurrent(mock_success_response):
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock,
               return_value=mock_success_response):
        results = await fetch_multiple(["a", "b", "c"])
        assert len(results) == 3

@pytest.mark.asyncio
async def test_fetch_data_handles_error():
    mock_resp = MagicMock()
    mock_resp.raise_for_status.side_effect = httpx.HTTPStatusError(
        "Server Error", request=MagicMock(), response=MagicMock(status_code=500)
    )
    with patch("httpx.AsyncClient.get", new_callable=AsyncMock,
               return_value=mock_resp):
        results = await fetch_multiple(["failing-endpoint"])
        assert isinstance(results[0], Exception)
PYEOF

git add tests/
git commit -m "test(fetcher): add unit tests for data fetch functions"

git switch main
git merge feature/tests --no-ff -m "test: merge unit test suite"
git branch -d feature/tests

# === VERIFY ===
uv run pytest
uv run python verify_project.py
```

**The critical pattern students must internalize:**
1. `.gitignore` is set up BEFORE any secret files are created.
2. Every feature goes through a branch → commit → merge cycle.
3. `uv add` (not `pip install`) is used for every dependency.
4. Both `pyproject.toml` and `uv.lock` are committed.
5. `.venv` and `.env` are never committed.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE (The Distinction)

## When Git and uv Best Practices Become Overkill

**Task:** For each scenario below, argue that the "professional" workflow taught in this lecture is **worse** than the simpler alternative. Write a brief justification (3-5 sentences) and, where applicable, a code snippet or command sequence showing why the simple approach is superior.

---

**Scenario A: Branching**

You are writing a one-time data cleanup script (`cleanup.py`, ~30 lines) that you will run once, verify the output, and then delete. You are the sole developer. It will never be shared, deployed, or maintained.

**Question:** Is creating a feature branch, making conventional commits, opening a PR, and merging to main appropriate here? Argue why the strict workflow is counterproductive in this case. What would you do instead?

---

**Scenario B: Lock Files**

You are leading a 2-hour hackathon workshop where participants rapidly prototype ideas. They need to install, try, and discard 10-15 different packages in quick succession to explore what's possible. Speed of experimentation is the only metric that matters.

**Question:** Is managing dependencies through `uv add` and maintaining a `uv.lock` appropriate here? Argue why the reproducibility pipeline adds friction that harms the goal. What would you use instead?

---

**Scenario C: Conventional Commits**

You are in the middle of a time-critical production outage. The fix is a one-line change. Your server is losing money every minute it's down.

**Question:** Should you follow the branch-PR-review-merge workflow? Should you carefully format a conventional commit message? Argue when speed should override process. What would you do, and what would you do *afterward*?

---

**Scenario D: Version Pinning**

You maintain an open-source library (not an application) that other people install via `pip install yourlibrary`. Your `pyproject.toml` declares `dependencies = ["httpx>=0.28.1"]`.

**Question:** Should you commit a `uv.lock` file in this case? Explain the difference between an **application** (deployed to a specific environment) and a **library** (installed into someone else's environment). Why does strict pinning in a library cause problems for users?

---

# Level 4 Solutions

**Scenario A: Branching — When it's overkill**

For a throwaway script that will be deleted in an hour, the branching overhead (branch creation, conventional commit, merge) costs more time than it saves. The purpose of branches is to protect a stable mainline during uncertain experiments and enable collaboration. A solo, disposable script has no mainline to protect and no collaborators. Committing directly to `main` (or not using Git at all) is appropriate. The judgment is: if the cost of losing the code is zero, the cost of protecting it should also be zero.

**Scenario B: Lock Files — When reproducibility is friction**

In a rapid prototyping session, the goal is *exploration speed*, not *reproducibility*. Running `uv add` and `uv remove` 15 times generates 15 lockfile resolutions and 15 `pyproject.toml` edits — overhead that slows experimentation. A plain `pip install` into a throwaway virtual environment is faster because it skips resolution, lockfile generation, and project metadata updates. After the hackathon, if a prototype is worth keeping, you would *then* formalize it with `uv init` and `uv add` for the packages you actually need. The key distinction: **lock files are for code that will be shared or deployed**; throwaway exploration doesn't need them.

**Scenario C: Conventional Commits — When process must yield to urgency**

During a production outage, the priority is: *fix, verify, deploy*. Spending even 2 minutes on branch creation, PR description, and reviewer approval could cost significant money. The correct action is to commit directly to main (or the deployment branch), push, and deploy. The commit message can be brief: `fix: patch null pointer in payment handler`. **After** the fire is out, write a post-mortem, create a proper PR with the fix explained, add tests that prevent regression, and consider what process changes would prevent this in the future. Speed and process aren't permanently opposed — you borrow from process during the emergency and pay it back afterward.

**Scenario D: Version Pinning — Libraries vs. Applications**

A library should **NOT** commit `uv.lock` (or at least not use it for distribution). The reason: when a user does `pip install yourlibrary`, pip must resolve your dependencies alongside all their *other* dependencies. If your library pins `httpx==0.28.1` exactly, and the user's application needs `httpx==0.29.0`, pip cannot satisfy both — the installation fails. Libraries should declare **loose constraints** (`httpx>=0.28`) so that the user's resolver has room to find a compatible version. Applications, by contrast, should pin exact versions (via `uv.lock`) because they control the entire environment. The rule: **libraries specify floors (`>=`), applications pin exact versions (`==`)**. A `uv.lock` in a library repo is fine for the library's *own development and CI*, but it should not influence what gets installed for end users.

---

# LEVEL 5: THE TEACH BACK (Synthesis)

## Scenario 5.1: "Why Can't I Just Email Zip Files?"

A junior developer joins your team and says:

> "I don't understand why we need Git. It's so complicated with branches and staging and all that. Can't I just zip my code folder and email it to the team when I'm done? Or use Google Drive? That way everyone has the latest version."

**Task:** Write a response that:

1. **Acknowledges** the legitimate frustration (Git IS complex for beginners).
2. **Presents a concrete scenario** where the zip-file approach fails catastrophically (not a hypothetical — write it as a story with two developers and show the exact moment things go wrong).
3. **Explains the three specific Git features** that solve the problem, mapped to the scenario.
4. **Ends with a practical concession:** Name one situation where the zip/Drive approach is genuinely fine.

---

## Scenario 5.2: "pip install Works Fine, Why Do I Need uv?"

The same junior developer says:

> "I just `pip install` whatever I need and it works. `uv` is just another tool to learn. My projects run fine with pip. Why add complexity?"

**Task:** Write a response that:

1. Shows a **specific, step-by-step failure scenario** where `pip install` works perfectly on the junior's machine but causes the production server to crash at 2 AM. Include exact commands and error messages.
2. Explains the difference between **"works for me"** and **"works for everyone, every time, on every machine."**
3. Demonstrates how `uv add` + `uv.lock` prevents the exact failure from your scenario.
4. Addresses the objection honestly: acknowledge when `pip install` is genuinely sufficient (be specific).

---

## Scenario 5.3: "I Committed My API Key, But I Deleted It in the Next Commit, So It's Fine"

The junior developer pushes a commit and says:

> "Oops, I accidentally committed my `.env` file. But I deleted it in the next commit and added `.env` to `.gitignore`. The file doesn't show up in `git status` anymore, so the key is safe now, right?"

**Task:** Write a response that:

1. **Immediately** tells them the ONE action they must take before anything else (not a Git command — a security action).
2. Explains, using the **snapshot model** from this lecture, exactly why deleting a file in a later commit does NOT remove it from history.
3. Walks through the exact commands an attacker would use to extract the key from the repository.
4. Describes the complete remediation process (both the Git cleanup AND the non-Git steps).
5. Shows the correct setup they should have used from the start, as a series of commands.

---

# Level 5 Solutions

## Solution 5.1: Why Not Zip Files?

**1. Acknowledgment:**
"You're right that Git has a learning curve. The commands feel arcane at first, and the three-areas model is genuinely non-obvious. But here's why we use it despite that cost."

**2. The Scenario:**

Alice and Bob are both working on `app.py`. On Monday, Alice emails Bob a zip of the project. Bob starts working on the `login` function. Alice, meanwhile, rewrites the `database` function. On Wednesday, both finish. Alice emails her zip. Bob emails his zip. Now there are two versions of `app.py`. Alice's zip has the new `database` function but the OLD `login` function. Bob's zip has the new `login` function but the OLD `database` function. Someone must now manually open both files side by side and combine the changes line by line. With a 500-line file, this takes an hour and introduces new bugs.

Now multiply by 5 developers, 50 files, daily changes. It's unworkable.

**3. Three Git Features That Solve This:**

- **Merge:** Git can automatically combine Alice's changes and Bob's changes into a single file, as long as they edited different sections. The merge algorithm handles what a human would have to do manually.
- **Branches:** Alice works on `feature/database`, Bob works on `feature/login`. They don't interfere with each other. When done, each merges into main independently.
- **History:** If the merged result has a bug, `git log` shows exactly what changed and when. `git bisect` can find the exact commit that introduced the bug. With zip files, there's no trail.

**4. Concession:**
If you're writing a solo, one-off script that nobody else will ever touch or run, saving it to Google Drive is fine. Git's value is proportional to the number of collaborators, the frequency of changes, and the cost of losing work.

---

## Solution 5.2: Why Not Just pip?

**1. Failure Scenario:**

```
Friday, 5 PM — Junior's Machine:
$ pip install httpx
$ python main.py
"Server started! ✅"
# (httpx 0.28.1 installed, Python 3.12)
$ git push

Saturday, 2 AM — Production Server (auto-deploy):
$ pip install httpx
# (installs httpx 0.29.0 — released Friday evening)
# (0.29.0 removed a deprecated method the junior used)
$ python main.py
AttributeError: module 'httpx' has no attribute 'Response.stream_text'
# 💥 Server crashes. Customers see 502 errors.
```

The junior's code was correct on Friday. httpx released a new version Friday night. `pip install httpx` on the production server installed the *latest* version (0.29.0), not the version the junior tested with (0.28.1). A breaking change in the new version crashed the application.

**2. "Works for me" vs "Works everywhere":**

`pip install httpx` means "give me whatever version is newest right now." That version changes over time and differs between machines depending on when you run the command. "Works for me" means "it worked at 5 PM on Friday on my specific machine." "Works everywhere" means "it will produce identical behavior on any machine, at any time, with zero ambiguity."

**3. How uv prevents this:**

```
$ uv add httpx            # Installs httpx 0.28.1, pins it in uv.lock
$ git push                # uv.lock committed with exact version

# Production server:
$ uv sync                 # Reads uv.lock, installs httpx==0.28.1 EXACTLY
$ uv run python main.py   # Same version as the junior tested — works ✅
```

`uv.lock` is a contract: "These exact package versions, at these exact hashes, are known to work together." `uv sync` honors that contract byte-for-byte.

**4. When pip is sufficient:**
For one-off scripts, learning experiments, Jupyter notebooks, or any throwaway code where you don't care about reproducibility. If nobody else will run it and you're okay reinstalling packages if something breaks, `pip install` is perfectly fine. The tool's purpose is reproducibility — if you don't need that, you don't need the tool.

---

## Solution 5.3: The Committed API Key

**1. IMMEDIATE ACTION: Rotate the key.**

Before anything else — before any Git command, before any cleanup — go to the API provider's dashboard and generate a **new** API key. Revoke the old one. The old key must be treated as compromised from the moment it was pushed, even to a private repository. Don't wait. Do it now.

**2. Why deletion doesn't remove it (Snapshot Model):**

Remember the Time Machine analogy: every commit is a **photograph** of your entire project at that moment. Commit 1 is a photograph that includes `.env` with the API key in full view. Commit 2 is a new photograph where `.env` is gone. But Commit 1's photograph still exists in the album. It wasn't burned. It wasn't altered. Anyone who opens the album can flip back to that page.

In Git terms: each commit is an immutable snapshot identified by a SHA-1 hash. Creating a new commit that removes `.env` only means the *new* snapshot doesn't include it. The *old* snapshot is permanent. Git's entire purpose is to never lose history.

**3. How an attacker extracts the key:**

```bash
# Clone the repo (gets ALL history)
git clone https://github.com/victim/project.git
cd project

# Method 1: Browse commit history
git log --all --oneline
# Find the commit before .env was removed
git show <commit-hash>:.env
# Output: API_KEY=sk_live_abc123...

# Method 2: Search all diffs
git log -p --all -S "API_KEY"
# Shows every commit that added/removed lines containing "API_KEY"

# Method 3: List all files ever tracked
git log --all --diff-filter=D --name-only
# Shows .env was deleted — drawing attention to it
```

**4. Complete Remediation:**

1. **Rotate the key** (already done in step 1).
2. **Remove from Git history** using `git filter-repo`:
   ```bash
   pip install git-filter-repo
   git filter-repo --path .env --invert-paths
   ```
   This rewrites every commit to exclude `.env`. All commit hashes change.
3. **Force-push** the rewritten history: `git push --force --all`
4. **Notify all collaborators** to re-clone (their local copies have the old history).
5. **Check GitHub's cached data** — even after force-push, GitHub may cache old commits briefly. Contact support if the repo was public.

**5. The Correct Setup From the Start:**

```bash
uv init my-project && cd my-project

# Step 1: .gitignore FIRST, before creating ANY .env file
echo ".env" >> .gitignore
git add .gitignore
git commit -m "chore: add .env to .gitignore"

# Step 2: Create .env (Git now ignores it)
echo "API_KEY=sk_live_real_key_here" > .env

# Step 3: Create .env.example (safe — no real values)
echo "API_KEY=your_key_here" > .env.example
git add .env.example
git commit -m "docs: add .env.example template"

# Verify:
git status   # .env does not appear
```

The rule: **`.gitignore` must exist and contain `.env` BEFORE the `.env` file is ever created.** Order matters.