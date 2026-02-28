# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, COMMANDS LAST                                   │
│  ────────────────────────────                                   │
│  Students must FEEL the chaos before learning the solution.     │
│  We'll show them a broken workflow. They'll cringe.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Git is abstract. We use a TIME MACHINE analogy throughout.     │
│  Dependencies are abstract. We use a RECIPE/KITCHEN analogy.    │
│  Every concept maps to something tangible.                      │
│                                                                 │
│  BUILD MENTAL MODEL BEFORE COMMANDS                             │
│  ──────────────────────────────────                             │
│  The snapshot model comes before git add.                       │
│  The three areas diagram comes before any terminal.             │
│  Understand the machine before driving it.                      │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Lecture 1–3 → You wrote code. Now learn to manage it.          │
│  Type hints → mypy runs inside your project via uv              │
│  Async code → The mini-project will require Git + uv            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEVELOPER WORKFLOW                           │
│                    (3.5–4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Disaster Scenario (Demonstration)                   │
│  ├─ 1.2 Two Problems, One Lecture                               │
│  └─ 1.3 The Time Machine Analogy                                │
│                                                                 │
│  PART 2: GIT — THE MENTAL MODEL (40 min)                        │
│  ├─ 2.1 Snapshots, Not Diffs                                    │
│  ├─ 2.2 The Three Areas                                         │
│  ├─ 2.3 Commits (Snapshots in Time)                             │
│  └─ 2.4 Branches (Parallel Timelines)                           │
│                                                                 │
│  ✦ PRACTICE CHECKPOINT 1 — Diagram Exercise (15 min)            │
│                                                                 │
│  PART 3: GIT — COMMANDS IN PRACTICE (50 min)                    │
│  ├─ Before You Start: Git Identity Configuration                │
│  ├─ 3.1 First Repository (git init, git status)                 │
│  ├─ 3.2 The Add-Commit Cycle                                    │
│  ├─ 3.2a Undoing Changes — Using the Time Machine               │
│  ├─ 3.2b Git File Operations                                    │
│  ├─ 3.3 .gitignore — What NOT to Track                          │
│  ├─ 3.4 Branching and Switching                                 │
│  ├─ 3.5 Merge (Combining Timelines)                             │
│  ├─ 3.6 Reading History (git log)                               │
│  ├─ 3.7 Git Tags — Marking Significant Points in History        │
│  [Optional: Rebase — Supplementary Reading]                     │
│  [Optional: Branching Strategies — Supplementary Reading]       │
│                                                                 │
│  ✦ PRACTICE CHECKPOINT 2 — Branch-Commit-Merge (15 min)         │
│                                                                 │
│  PART 4: COLLABORATION & CONVENTIONS (40 min)                   │
│  ├─ GitHub Authentication Setup (HTTPS / SSH)                   │
│  ├─ 4.1 Remotes — Connecting to GitHub                          │
│  ├─ 4.2 Push, Pull, Fetch (Syncing Timelines)                   │
│  ├─ 4.3 Pull Request Workflow (The Review Gate)                 │
│  ├─ 4.4 Conventional Commits (feat, fix, chore)                 │
│  ├─ 4.5 Code Review — A Worked Example                          │
│  └─ 4.6 IDE Git Integration & GUI Clients                       │
│                                                                 │
│  PART 5: PROJECT MANAGEMENT WITH UV (50 min)                    │
│  ├─ 5.1 The "It Works on My Machine" Problem                    │
│  ├─ 5.2 uv — The Modern Python Project Manager                  │
│  ├─ 5.3 Starting a Project (uv init)                            │
│  ├─ 5.4 Managing Dependencies (uv add, uv remove)               │
│  ├─ 5.5 pyproject.toml — The Single Source of Truth             │
│  ├─ 5.6 uv.lock and uv run — Reproducibility in Action          │
│  ├─ 5.7 Python Version Management                               │
│  ├─ 5.8 Legacy Context: venv, pip, requirements.txt             │
│  └─ 5.9 Common Mistakes and Misconceptions                      │
│                                                                 │
│  ✦ PRACTICE CHECKPOINT 3 — Alice-to-Bob Reproduction (15 min)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Disaster Scenario

**Start with a demonstration. Make them cringe.**

**Show this directory listing on screen:**

```
my_project/
├── app.py
├── app_v2.py
├── app_v2_final.py
├── app_v2_final_FIXED.py
├── app_v2_final_FIXED_real.py
├── app_old_DONT_DELETE.py
├── app_backup_jan15.py
├── app_backup_jan15_2.py
└── notes.txt          ← "i think the bug is in line 47 but idk"
```

**Now ask the class:**

> "Raise your hand if your projects folder has EVER looked like this."

*Wait. Let them laugh.*

> "Now the real horror story. You spent 3 hours building a feature. It worked. Then you 'refactored' — cleaned it up, reorganized. Now it's broken. You can't remember exactly what you changed. You can't go back. The old version? You overwrote it."

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE HORROR TIMELINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Monday:   app.py works! 🎉                                     │
│  Tuesday:  "Let me improve it..." (edit edit edit)              │
│  Tuesday:  app.py broken. 😰                                    │
│  Tuesday:  "Wait, what did I change?"                           │
│  Tuesday:  Ctrl+Z 47 times. Still broken. 😱                    │
│  Tuesday:  "I'll just start over." 😭                           │
│                                                                 │
│  Wednesday: Copy app.py → app_backup.py (just in case)          │
│  Thursday:  Copy app.py → app_v2.py (new feature)               │
│  Friday:    Copy app.py → app_v2_final.py (wait which is which) │
│  Saturday:  "WHICH FILE IS THE REAL ONE?"                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now show this conversation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE "IT WORKS ON MY MACHINE" CHAT                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice:  "Hey, can you test my code?"                           │
│  Bob:    "Sure, sent me the zip"                                │
│  Bob:    "ModuleNotFoundError: No module named 'httpx'"         │
│  Alice:  "Oh, pip install httpx"                                │
│  Bob:    "Done. Now: TypeError: AsyncClient.__init__() got      │
│           an unexpected keyword argument 'proxies'"             │
│  Alice:  "What version do you have?"                            │
│  Bob:    "0.23.0"                                               │
│  Alice:  "You need 0.27. Also install pytest and mypy"          │
│  Bob:    "Which versions?"                                      │
│  Alice:  "Uhh... whatever I have? Let me check..."              │
│  Bob:    "What Python version do you use?"                      │
│  Alice:  "3.12"                                                 │
│  Bob:    "I have 3.9..."                                        │
│  Alice:  "..."                                                  │
│  Bob:    "..."                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "The code Alice wrote in Lectures 1–3 — type hints, dataclasses, async functions — all of it lives in files. Files that change. Files that break. Files that need to be shared. Right now you have NO system for managing that. Today that changes."

---

## 1.2 Two Problems, One Lecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TWO PROBLEMS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: TRACKING CHANGES                                    │
│  ────────────────────────────                                   │
│  • How do I save snapshots of my project over time?             │
│  • How do I go back to a version that worked?                   │
│  • How do I experiment without breaking what works?             │
│  • How do I collaborate without overwriting each other?         │
│                                                                 │
│  SOLUTION: Git (version control)                                │
│                                                                 │
│                                                                 │
│  PROBLEM 2: REPRODUCING ENVIRONMENTS                            │
│  ───────────────────────────────────                            │
│  • How do I ensure everyone has the same packages?              │
│  • How do I ensure everyone has the same versions?              │
│  • How do I ensure everyone uses the same Python version?       │
│  • How do I add/remove packages without chaos?                  │
│                                                                 │
│  SOLUTION: uv (project and dependency management)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Time Machine Analogy

**This analogy will carry us through Parts 1–4.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE TIME MACHINE ANALOGY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine you're building a city (your project).                 │
│  You have a TIME MACHINE with these powers:                     │
│                                                                 │
│  📸 SNAPSHOT:                                                   │
│     At any moment, you can take a photograph of your entire     │
│     city exactly as it is right now. Date-stamped. Labeled.     │
│                                                                 │
│  ⏪ REWIND:                                                     │
│     You can go back to any photograph and see exactly           │
│     what the city looked like at that moment.                   │
│                                                                 │
│  🌐 PARALLEL TIMELINES:                                         │
│     You can create a parallel universe to try a crazy idea.     │
│     If it works → merge it into the real timeline.              │
│     If it fails → discard it. The real timeline is untouched.   │
│                                                                 │
│  ☁️  CLOUD BACKUP:                                              │
│     You can upload all your timelines to the cloud.             │
│     Others can download them, add their own snapshots,          │
│     and propose merging their changes into the main timeline.   │
│                                                                 │
│  That time machine is called GIT.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to Git:**

```
Time Machine             │  Git
─────────────────────────│─────────────────────────
Your city (project)      │  Working directory
Taking a photograph      │  Making a commit
The photograph album     │  Repository (.git/)
Choosing what to include │  Staging area (git add)
  in the photo           │
A parallel timeline      │  A branch
The official timeline    │  The main branch
Merging timelines        │  git merge
Cloud backup             │  Remote (GitHub)
Uploading photos         │  git push
Downloading photos       │  git pull
Proposing a merge        │  Pull request
```

---

# PART 2: GIT — THE MENTAL MODEL

## 2.1 Snapshots, Not Diffs

**The most common misconception about Git: people think it stores CHANGES. It doesn't. It stores SNAPSHOTS.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SNAPSHOTS, NOT DIFFS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT YOU MIGHT THINK (wrong):                                  │
│  ─────────────────────────────                                  │
│  "Git stores the differences between versions"                  │
│                                                                 │
│     Version 1 → Version 2: "Changed line 5, added line 20"      │
│     Version 2 → Version 3: "Deleted line 12, changed line 7"    │
│                                                                 │
│                                                                 │
│  WHAT GIT ACTUALLY DOES (correct):                              │
│  ─────────────────────────────────                              │
│  "Git takes a FULL SNAPSHOT of every tracked file"              │
│                                                                 │
│     Commit 1: [main.py v1] [utils.py v1] [config.py v1]         │
│     Commit 2: [main.py v2] [utils.py v1] [config.py v1]         │
│     Commit 3: [main.py v2] [utils.py v2] [config.py v1]         │
│                          ▲               ▲                      │
│                     changed          changed                    │
│                                                                 │
│  (If a file DIDN'T change, Git stores a pointer to the          │
│   previous version — it's smart about storage.)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does this matter?**

> "Because it means every commit is a COMPLETE picture of your project at that moment. You can jump to ANY commit and see the full project — not a pile of patches you have to reconstruct. Each photograph in your album is a complete photograph, not a list of edits since the last one."

---

## 2.2 The Three Areas

**This is the most important mental model in Git. Understand this, and everything else clicks.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE THREE AREAS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    git add    ┌──────────────┐   git commit   │
│  │   WORKING    │ ───────────▶  │   STAGING    │ ────────────▶  │
│  │  DIRECTORY   │               │    AREA      │                │
│  │              │               │  (Index)     │                │
│  │  Your files  │               │  "Selected   │                │
│  │  as they are │               │   for next   │                │
│  │  right now   │               │   photo"     │                │
│  └──────────────┘               └──────────────┘                │
│                                                                 │
│                                        │                        │
│                                        ▼                        │
│                                                                 │
│                                ┌──────────────┐                 │
│                                │  REPOSITORY  │                 │
│                                │   (.git/)    │                 │
│                                │              │                 │
│                                │  All your    │                 │
│                                │  saved       │                 │
│                                │  snapshots   │                 │
│                                └──────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Explain with the Time Machine analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE THREE AREAS — TIME MACHINE EDITION             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKING DIRECTORY = The city RIGHT NOW                         │
│  ──────────────────────────────────────                         │
│  You're building, demolishing, rearranging. This is the         │
│  live, messy, in-progress state. Nothing is saved yet.          │
│                                                                 │
│                                                                 │
│  STAGING AREA = Selecting what to photograph                    │
│  ───────────────────────────────────────────                    │
│  You don't have to photograph the ENTIRE city every time.       │
│  Maybe you only built a new bridge today. Stage just that.      │
│  "I want THIS change in my next snapshot, but not THAT one."    │
│                                                                 │
│                                                                 │
│  REPOSITORY = The photo album                                   │
│  ────────────────────────────                                   │
│  The permanent record. Every photograph you've ever taken,      │
│  date-stamped, labeled, in order. You can browse it anytime.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does the staging area exist?**

> "It lets you CHOOSE what goes into each commit. You changed 5 files, but only 2 are related to the bug fix? Stage those 2. Commit. Now stage the other 3 for a separate commit. Each snapshot tells a clean story."

---

## 2.3 Commits (Snapshots in Time)

**A commit is a saved snapshot with metadata.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      ANATOMY OF A COMMIT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     ┌────────────────────────────────────────┐                  │
│     │  Commit: a1b2c3d                       │                  │
│     │                                        │                  │
│     │  Author:  Alice <alice@example.com>    │                  │
│     │  Date:    2025-02-16 14:30:00          │                  │
│     │  Message: "Add weather fetch function" │                  │
│     │                                        │                  │
│     │  Parent:  f4e5d6c  ←────────────────── │──── Previous     │
│     │                                        │     commit       │
│     │  Snapshot:                              │                 │
│     │    main.py    → [contents at this time] │                 │
│     │    utils.py   → [contents at this time] │                 │
│     │    config.py  → [contents at this time] │                 │
│     └────────────────────────────────────────┘                  │
│                                                                 │
│                                                                 │
│  Every commit knows its PARENT (the commit before it).          │
│  This forms a CHAIN — a timeline of your project's history.     │
│                                                                 │
│     f4e5d6c ──▶ a1b2c3d ──▶ b7c8d9e ──▶ e1f2g3h                 │
│     "init"     "add       "add        "fix                      │
│                 weather"   news"       error"                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Each commit has a unique ID:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      COMMIT HASHES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Full hash:  a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0           │
│  Short hash: a1b2c3d  (first 7 characters — usually unique)     │
│                                                                 │
│  This ID is computed from:                                      │
│  • The snapshot content                                         │
│  • The author                                                   │
│  • The timestamp                                                │
│  • The parent commit                                            │
│                                                                 │
│  Change ANY of these → completely different hash.               │
│  This guarantees integrity. Nobody can tamper with history.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Branches (Parallel Timelines)

**A branch is a parallel timeline where you can experiment safely.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  BRANCHES = PARALLEL TIMELINES                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You're on the MAIN timeline. Everything works.                 │
│  You want to try adding a new feature — but what if it breaks?  │
│                                                                 │
│  WITHOUT BRANCHES:                                              │
│  ─────────────────                                              │
│  main: A ──▶ B ──▶ C ──▶ 💥 (broke it, can't easily undo)       │
│                                                                 │
│                                                                 │
│  WITH BRANCHES:                                                 │
│  ───────────────                                                │
│  main:    A ──▶ B ──▶ C                  (safe, untouched)      │
│                  │                                              │
│                  └──▶ D ──▶ E            (experiment here)      │
│                       feature branch                            │
│                                                                 │
│  Feature works? → Merge it back into main.                      │
│  Feature fails? → Delete the branch. Main is untouched.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight: a branch is just a movable POINTER to a commit.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   BRANCHES ARE POINTERS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A branch is NOT a copy of your code.                           │
│  A branch is a LABEL that points to a commit.                   │
│                                                                 │
│                                                                 │
│     A ──▶ B ──▶ C         ◀── main (pointer to C)               │
│                  │                                              │
│                  └──▶ D   ◀── feature (pointer to D)            │
│                                                                 │
│                       ▲                                         │
│                       │                                         │
│                      HEAD (which branch you're "on")            │
│                                                                 │
│                                                                 │
│  When you make a new commit on "feature":                       │
│                                                                 │
│     A ──▶ B ──▶ C         ◀── main (still points to C)          │
│                  │                                              │
│                  └──▶ D ──▶ E   ◀── feature (moved to E)        │
│                                                                 │
│  The "main" pointer didn't move. The "feature" pointer          │
│  advanced to the new commit. That's ALL a branch is.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**HEAD — "Which timeline am I on RIGHT NOW?"**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  HEAD = Your current position in the timeline.                  │
│                                                                 │
│  HEAD → main     means "I'm on the main branch"                 │
│  HEAD → feature  means "I'm on the feature branch"              │
│                                                                 │
│  When you commit, the branch HEAD points to moves forward.      │
│  HEAD itself follows.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Git maintains HEAD as an independent pointer — not merely a synonym for your current branch — because it must track your exact position in history even when you have left all named branches behind, such as when inspecting an old commit. Every reference you will later see, like `HEAD~1` (one commit before HEAD) or `HEAD^` (same thing), relies on this pointer existing independently of any branch.

---

```
┌─────────────────────────────────────────────────────────────────┐
│         ✦ PRACTICE CHECKPOINT 1 — DIAGRAM EXERCISE              │
│                    (15 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stop the demo. Close your notes. Work independently.           │
│                                                                 │
│  1. Draw the Three Areas diagram from memory and label each     │
│     area with its Time Machine analogy equivalent               │
│     (city / selecting for photo / photo album).                 │
│                                                                 │
│  2. Trace this exact sequence through your diagram:             │
│     a. You edit main.py                                         │
│     b. You run: git add main.py                                 │
│     c. You run: git commit -m "feat: add fetch function"        │
│     Where does main.py live after each step?                    │
│                                                                 │
│  3. Write one sentence explaining why the Staging Area          │
│     exists. Do not use the word "stage" in your answer.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: GIT — COMMANDS IN PRACTICE

## Before You Start — Git Identity Configuration

Git signs every commit with your identity — run these once per machine, not once per project. Git will refuse to create any commit without this information.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

> "Run these now, before anything else. You will only ever need to run them once per machine."

---

## 3.1 First Repository (git init, git status)

**Let's build the mental model into real commands.**

```bash
# Create a project folder
mkdir weather-cli
cd weather-cli

# Initialize Git — create the time machine
git init
```

Output:
```
Initialized empty Git repository in /home/user/weather-cli/.git/
```

**What just happened?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    git init                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before:                    After:                              │
│                                                                 │
│  weather-cli/               weather-cli/                        │
│  └── (empty)                ├── .git/  ← THE TIME MACHINE       │
│                             │   ├── objects/  (snapshots)       │
│                             │   ├── refs/     (branches)        │
│                             │   ├── HEAD      (current branch)  │
│                             │   └── ...                         │
│                             └── (your files go here)            │
│                                                                 │
│  The .git/ folder IS your repository.                           │
│  Delete it → all history is gone.                               │
│  Everything else is your working directory.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now create a file and check status.**

Create a file called `main.py` in your editor and add the following content:

```python
import asyncio

async def fetch_weather(city: str) -> dict:
    """Fetch weather for a given city."""
    print(f"Fetching weather for {city}...")
    await asyncio.sleep(1)
    return {"city": city, "temp": 20}

async def main():
    result = await fetch_weather("London")
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
# Ask Git: "What's going on?"
git status
```

Output:
```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        main.py

nothing added to commit but untracked files present (use "git add" to track it)
```

**Read the output together with the class:**

> "Git is telling you: 'I see main.py, but I'm not tracking it yet. It exists in your WORKING DIRECTORY but not in my STAGING AREA or REPOSITORY. You have to explicitly tell me to watch it.'"

```
┌─────────────────────────────────────────────────────────────────┐
│              AFTER git init + creating main.py                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKING DIR          STAGING AREA         REPOSITORY           │
│  ──────────           ────────────         ──────────           │
│  main.py ✨ (new)     (empty)              (empty)              │
│                                                                 │
│  Git sees the file but isn't tracking it yet.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 The Add-Commit Cycle

**This is the cycle you'll repeat hundreds of times a day.**

```bash
# Step 1: Stage the file (select it for the next snapshot)
git add main.py

# Check status again
git status
```

Output:
```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   main.py
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      AFTER git add                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKING DIR          STAGING AREA         REPOSITORY           │
│  ──────────           ────────────         ──────────           │
│  main.py              main.py ✅            (empty)             │
│                       "Ready for                                │
│                        the photo"                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Step 2: Commit — take the snapshot
git commit -m "Add initial weather fetch function"
```

Output:
```
[main (root-commit) a1b2c3d] Add initial weather fetch function
 1 file changed, 13 insertions(+)
 create mode 100644 main.py
```

```
┌─────────────────────────────────────────────────────────────────┐
│                     AFTER git commit                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKING DIR          STAGING AREA         REPOSITORY           │
│  ──────────           ────────────         ──────────           │
│  main.py              (clean)              Commit a1b2c3d       │
│                                            "Add initial         │
│                                             weather fetch       │
│                                             function"           │
│                                            └── main.py          │
│                                                                 │
│  Working directory matches the repository. Everything is clean. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The complete cycle visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE ADD-COMMIT CYCLE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│         ┌─────────┐   git add   ┌─────────┐  git commit         │
│    ┌──▶ │  Edit   │ ──────────▶ │  Stage  │ ──────────▶ 📸      │
│    │    │  files  │             │  files  │           Snapshot! │
│    │    └─────────┘             └─────────┘                     │
│    │                                                            │
│    └────────────────────────────────────────────────────────────┘
│                          Repeat forever                         │
│                                                                 │
│                                                                 │
│  COMMANDS:                                                      │
│                                                                 │
│    git add main.py         ← Stage one specific file            │
│    git add .               ← Stage ALL changed files            │
│                              (use only when every change        │
│                               belongs in this commit)           │
│                                                                 │
│    git commit -m "message" ← Commit with inline message         │
│                                                                 │
│    git status              ← "Where am I? What's staged?"       │
│    git diff                ← "What changed since last commit?"  │
│    git diff --staged       ← "What's staged for next commit?"   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> Early in your practice, prefer `git add <specific-file>` so you develop the habit of reviewing what you are committing. Reach for `git add .` only when you are certain every change in your working directory belongs in this commit.

**Let's do a second commit to see the chain.**

Add the following lines to the end of `main.py` in your editor:

```python

async def fetch_news(topic: str) -> dict:
    """Fetch news articles about a topic."""
    print(f"Fetching news about {topic}...")
    await asyncio.sleep(1)
    return {"topic": topic, "articles": 5}
```

```bash
# See what changed
git diff
```

Output:
```diff
diff --git a/main.py b/main.py
index 3a4b5c6..7d8e9f0 100644
--- a/main.py
+++ b/main.py
@@ -11,3 +11,9 @@ async def main():

 if __name__ == "__main__":
     asyncio.run(main())
+
+async def fetch_news(topic: str) -> dict:
+    """Fetch news articles about a topic."""
+    print(f"Fetching news about {topic}...")
+    await asyncio.sleep(1)
+    return {"topic": topic, "articles": 5}
```

> "Lines starting with `+` were ADDED. Lines with `-` were REMOVED. Git shows you exactly what changed."

```bash
git add main.py
git commit -m "Add news fetch function"
```

**Now we have a chain of two commits:**

```
    a1b2c3d ──────▶ b4c5d6e
    "Add initial     "Add news
     weather fetch    fetch
     function"        function"
```

---

## 3.2a Undoing Changes — Using the Time Machine

**The entire point of Git's time machine is to let you go back. Here are the commands that actually do it.**

> "The most important thing you'll learn today isn't how to commit — it's how to undo. Beginners who don't know undo commands are the ones who delete their `.git/` folder in a panic."

```
┌─────────────────────────────────────────────────────────────────┐
│            UNDOING CHANGES — THE FOUR SCENARIOS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO 1:                                                    │
│  "I edited a file — I want to throw away my changes and         │
│   restore it to what it looked like at the last commit."        │
│                                                                 │
│      git restore main.py                                        │
│      ⚠️  Permanent — your unsaved edits are gone.               │
│                                                                 │
│                                                                 │
│  SCENARIO 2:                                                    │
│  "I ran git add — I want to unstage the file but keep           │
│   my edits in the working directory."                           │
│                                                                 │
│      git restore --staged main.py                               │
│      ✅ Safe — your edits still exist, just un-queued.          │
│                                                                 │
│                                                                 │
│  SCENARIO 3:                                                    │
│  "I already committed — I want to undo that commit              │
│   without destroying history."                                  │
│                                                                 │
│      git revert a1b2c3d                                         │
│      ✅ Creates a NEW commit that undoes the target commit.     │
│         Safe for shared history. The old commit still exists.   │
│                                                                 │
│                                                                 │
│  SCENARIO 4 (DANGER):                                           │
│  "I want to erase the last commit and all its changes."         │
│                                                                 │
│      git reset --hard HEAD~1                                    │
│      ⚠️  DESTRUCTIVE — uncommitted changes are permanently      │
│          lost. Never use on commits already pushed to a         │
│          shared remote.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Going deeper: the three modes of `git reset`.**

> "SCENARIO 4 showed you `--hard`. But `git reset` has three modes and you **will** encounter all three in tutorials and Stack Overflow answers. The difference maps directly to the Three Areas you already know. This is worth understanding precisely — getting it wrong is how people lose work."

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE THREE RESET MODES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  All three modes move HEAD (and the branch pointer) to a        │
│  different commit. They differ in what they do to the           │
│  Three Areas.                                                   │
│                                                                 │
│                                                                 │
│  START STATE (all three modes begin here):                      │
│  ──────────────────────────────────────────                     │
│                                                                 │
│  WORKING DIR      STAGING AREA     REPOSITORY                   │
│  main.py (v3)     (clean)          HEAD → b4c  "bad commit"     │
│                                         └── main.py v3          │
│                                    a1b  "previous commit"       │
│                                         └── main.py v2          │
│                                                                 │
│  GOAL: Undo commit b4c, go back to a1b.                         │
│                                                                 │
│                                                                 │
│  git reset --soft HEAD~1                                        │
│  ────────────────────────                                       │
│  WORKING DIR      STAGING AREA       REPOSITORY                 │
│  main.py (v3)     main.py (v3) ✅    HEAD → a1b                 │
│                   (still staged)                                │
│                                                                 │
│  ✅ HEAD moved back. Changes are still STAGED.                  │
│  Use when: bad commit message, want to split a large commit     │
│  into smaller ones, or want to amend before recommitting.       │
│                                                                 │
│                                                                 │
│  git reset --mixed HEAD~1   ← DEFAULT (same as git reset HEAD~1)│
│  ───────────────────────────────────────────────────────────── │
│  WORKING DIR      STAGING AREA       REPOSITORY                 │
│  main.py (v3)     (empty)            HEAD → a1b                 │
│                                                                 │
│  ✅ HEAD moved back. Changes are in WORKING DIR but unstaged.   │
│  Use when: you want to reorganise what you committed — pick     │
│  and choose which changes to re-stage and recommit separately.  │
│                                                                 │
│                                                                 │
│  git reset --hard HEAD~1                                        │
│  ────────────────────────                                       │
│  WORKING DIR      STAGING AREA       REPOSITORY                 │
│  main.py (v2) ⚠️  (empty)            HEAD → a1b                 │
│  (reverted to                                                   │
│   v2 — v3 is                                                    │
│   GONE)                                                         │
│                                                                 │
│  ⚠️  DESTRUCTIVE. All three areas now match the target commit.  │
│  Your working directory changes are permanently lost.           │
│  Never use on commits already pushed to a shared remote.        │
│                                                                 │
│                                                                 │
│  QUICK LOOKUP:                                                  │
│  ─────────────                                                  │
│  --soft   → your changes land in:  STAGING AREA (staged)        │
│  --mixed  → your changes land in:  WORKING DIR (unstaged)       │
│  --hard   → your changes land in:  NOWHERE (destroyed)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "A practical rule: reach for `--soft` when you want to redo a commit. Reach for `--mixed` (the default) when you want to reorganise changes. Only reach for `--hard` when you are absolutely certain the work should not exist. And if you used `--hard` by mistake, `git reflog` is your last resort — it's covered in the appendix."

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT — RESET MODES                        │
│                    (10 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Work through the following in your weather-cli repo.           │
│  Do NOT look at the diagram above.                              │
│                                                                 │
│  1. Make a small change to main.py and commit it with a         │
│     deliberately bad message: git commit -m "oops"              │
│                                                                 │
│  2. Run git log --oneline. Note your new commit hash.           │
│                                                                 │
│  3. Run git reset --soft HEAD~1.                                │
│     Answer WITHOUT running git status yet:                      │
│     Where does your change live right now?                      │
│     Then run git status. Were you right?                        │
│                                                                 │
│  4. Now run git reset --mixed HEAD~1.                           │
│     Answer WITHOUT running git status yet:                      │
│     Where does your change live now?                            │
│     Then run git status. Were you right?                        │
│                                                                 │
│  5. Write one sentence each explaining when you would choose    │
│     --soft over --mixed, and when you would avoid --hard.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ SOLUTION — RESET MODES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  3. After git reset --soft HEAD~1:                              │
│     Your change lives in the STAGING AREA.                      │
│     git status shows: "Changes to be committed: modified main.py"│
│     The commit is gone from the log, but the change is staged   │
│     and ready for a new commit with a corrected message.        │
│                                                                 │
│  4. After git reset --mixed HEAD~1:                             │
│     Your change lives in the WORKING DIRECTORY, unstaged.       │
│     git status shows: "Changes not staged for commit: main.py"  │
│     You must git add it again before you can commit.            │
│                                                                 │
│  5. Sample answers:                                             │
│     --soft over --mixed: When I want to immediately recommit    │
│     with a better message or merge with another staged change.  │
│     Avoid --hard: When there is any chance I want to keep        │
│     the work — --hard offers zero recovery without git reflog.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The most common daily sequence:**

```bash
# You edited a file and regret it — restore it
git restore main.py

# You staged something accidentally — unstage it
git restore --staged main.py

# You committed something wrong — undo it safely
git revert b4c5d6e
# Git opens your editor for a commit message — save and close
# Result: a new commit that reverses b4c5d6e. History is intact.
```

> "Use `git restore` for working-directory and staging mistakes. Use `git revert` for committed mistakes. These two commands are your safety net. `git reset --hard` is the ejector seat — effective, but it throws away work irreversibly."

---

## 3.2b Git File Operations

**Tell Git when you move or delete things.**

> "Every time you use the system `mv` to rename a file, Git sees a deleted file and a brand-new untracked file. It has no idea they're connected. Over time, you also accumulate generated files, leftovers, and experiment scripts with no way to clear them. Here is how to do file operations the Git-aware way."

**The problem with system `mv`:**

```bash
# ❌ WRONG — using system rename
mv utils.py helpers.py
git status
```

Output:
```
Changes not staged for commit:
        deleted:    utils.py

Untracked files:
        helpers.py
```

```bash
# ✅ CORRECT — Git-aware rename
git mv utils.py helpers.py
git status
```

Output:
```
Changes to be committed:
        renamed:    utils.py -> helpers.py
```

> "Git understands this is a rename. The full commit history of `utils.py` stays attached to `helpers.py`. With the system `mv`, that history appears broken."

```
┌─────────────────────────────────────────────────────────────────┐
│                   GIT FILE OPERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RENAME / MOVE A FILE:                                          │
│  ──────────────────────                                         │
│  git mv old_name.py new_name.py                                 │
│  git mv utils.py src/utils.py      ← move to a subdirectory    │
│                                                                 │
│  Equivalent to: mv + git add (deletion) + git add (new file)    │
│  Git tracks it as a rename, preserving full file history.       │
│                                                                 │
│                                                                 │
│  DELETE A TRACKED FILE:                                         │
│  ──────────────────────                                         │
│  git rm obsolete.py                ← deletes file + stages      │
│                                      removal in one step        │
│                                                                 │
│  ❌ rm obsolete.py                 ← system delete: file gone,  │
│                                      but removal not staged;    │
│                                      need git add afterwards    │
│                                                                 │
│                                                                 │
│  REMOVE FROM GIT TRACKING BUT KEEP ON DISK:                     │
│  ──────────────────────────────────────────                     │
│  git rm --cached secret.py         ← removes from index only   │
│  git rm -r --cached .venv/         ← recursively for folders   │
│                                                                 │
│  WHEN TO USE: You forgot .gitignore, committed a file once,     │
│  now want Git to stop tracking it. The file stays on your       │
│  machine; it disappears from the repository.                    │
│                                                                 │
│                                                                 │
│  CLEAN UNTRACKED FILES:                                         │
│  ──────────────────────                                         │
│  git clean -n           ← DRY RUN: shows what WOULD be deleted  │
│  git clean -f           ← deletes untracked files               │
│  git clean -fd          ← deletes untracked files + directories  │
│  git clean -fdx         ← also deletes .gitignore'd files       │
│                            ⚠️  extreme caution — this removes    │
│                            .env, .venv/, and generated files    │
│                                                                 │
│  ⚠️  git clean is IRREVERSIBLE. Untracked files have no Git      │
│      history. Always run -n first to preview.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The most common real-world scenario — "I committed `.venv/` by accident":**

```bash
# You forgot .gitignore and committed .venv/ to the repo.

# Step 1: Remove it from tracking but keep the folder on disk
git rm -r --cached .venv/

# Step 2: Make sure .gitignore has .venv/ (it should already)
# If not: echo ".venv/" >> .gitignore

# Step 3: Commit the correction
git add .gitignore
git commit -m "chore: stop tracking .venv/, update .gitignore"

# Step 4: Push the fix
git push
```

> "After this commit, `.venv/` will no longer appear in `git status` and will no longer exist in the repository. Everyone who pulls this commit keeps their local `.venv/` folder untouched — only the repository copy is removed."

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT — GIT FILE OPERATIONS                 │
│                    (10 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In your weather-cli repo:                                      │
│                                                                 │
│  1. Create a file called scratch.py and write one print()       │
│     statement. DO NOT git add it.                               │
│                                                                 │
│  2. Run git clean -n. What does it tell you?                    │
│     Then run git clean -f. What happened to scratch.py?         │
│                                                                 │
│  3. Create a file called notes.txt and add it to the repo:      │
│     git add notes.txt && git commit -m "chore: add notes"       │
│                                                                 │
│  4. Now run git rm --cached notes.txt.                          │
│     Run git status. Where does notes.txt appear?                │
│     Does the file still exist on disk? (check with ls)          │
│                                                                 │
│  5. Complete the removal: add notes.txt to .gitignore, then     │
│     git add + git commit the fix. Confirm git status is clean.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ SOLUTION — GIT FILE OPERATIONS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2. git clean -n shows: "Would remove scratch.py"               │
│     git clean -f deletes it permanently. ls confirms it's gone. │
│     ⚠️  This is why -n first is always the rule.                │
│                                                                 │
│  4. After git rm --cached notes.txt:                            │
│     git status shows:                                           │
│       Changes to be committed: deleted: notes.txt               │
│       Untracked files: notes.txt                                │
│     The file exists on disk (ls shows it), but Git no longer    │
│     considers it tracked. It will appear untracked every time   │
│     until you add it to .gitignore.                             │
│                                                                 │
│  5. Final state after adding to .gitignore and committing:      │
│     git status shows nothing — clean working tree.              │
│     notes.txt still exists locally but is invisible to Git.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 .gitignore — What NOT to Track

**Before creating `.gitignore`, a quick note on one of its entries:**

`.venv/` is where uv stores all installed packages locally — you'll understand exactly what this is in Part 5. For now, know that it is large, machine-specific, and never committed. uv recreates it automatically from a lockfile. The same principle applies to the other entries: they are either generated automatically or contain secrets that must never be shared.

**Some files should NEVER be in your repository.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT NOT TO TRACK                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ NEVER COMMIT THESE:                                         │
│  ├─ .venv/              ← Virtual environment (hundreds of MBs) │
│  ├─ __pycache__/        ← Compiled Python bytecode              │
│  ├─ .env                ← Secrets (API keys, passwords!)        │
│  ├─ *.pyc               ← Compiled files                        │
│  ├─ .DS_Store           ← macOS junk files                      │
│  └─ node_modules/       ← (if you ever touch JavaScript)        │
│                                                                 │
│  WHY?                                                           │
│  ├─ Generated files can be recreated (no need to save them)     │
│  ├─ Secrets are DANGEROUS in a repository (public or not!)      │
│  ├─ Large files bloat the repo for everyone                     │
│  └─ OS-specific files are irrelevant to your code               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Create a `.gitignore` file in your editor and add the following content:**

```
# Virtual environments
.venv/

# Python bytecode
__pycache__/
*.pyc
*.pyo

# Environment variables / Secrets
.env
.env.local

# IDE settings (optional — some teams track these)
.vscode/
.idea/

# OS files
.DS_Store
Thumbs.db

# Distribution / packaging (Week 15 — ignore for now)
dist/
build/
*.egg-info/
```

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

> "Once a file pattern is in .gitignore, Git pretends it doesn't exist. It won't show up in `git status`, won't be staged by `git add .`, and won't end up in your repository. This is your first line of defense."

### .gitattributes — Cross-Platform Consistency

> "`.gitignore` tells Git what NOT to track. `.gitattributes` tells Git HOW to treat the files it does track — particularly for teams mixing Windows and macOS/Linux."

**The line-ending problem:**

Windows editors save files with CRLF (`\r\n`) line endings. macOS and Linux use LF (`\n`). When a Windows developer commits a file, every single line appears changed to teammates on other systems — not because the code changed, but because the line endings did. `git diff` becomes unreadable noise.

**Create a `.gitattributes` file in the project root:**

```
# Normalise all text files to LF on commit (the canonical form)
* text=auto

# Always use LF for Python files regardless of OS
*.py text eol=lf

# Tell Git to use Python-aware diff (shows function names in diffs)
*.py diff=python

# Mark binary files explicitly — never diff or merge them as text
*.png binary
*.jpg binary
*.pdf binary
*.db  binary
```

```bash
git add .gitattributes
git commit -m "chore: add .gitattributes for cross-platform consistency"
```

> "The `text=auto` line is the most important one. It ensures that regardless of what OS a developer uses, line endings are normalised to LF when committed to the repository. Each developer's OS gets the endings it prefers in the working directory, but the repository stays consistent."

---

### If You Accidentally Commit a Secret

**This happens. Here is the correct response, in order:**

```
┌─────────────────────────────────────────────────────────────────┐
│             ACCIDENTALLY COMMITTED A SECRET                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1 — ROTATE THE SECRET IMMEDIATELY (before anything else)  │
│  ─────────────────────────────────────────────────────────────  │
│  Go to the service (GitHub, AWS, Stripe, etc.) and invalidate   │
│  the key or token right now. Assume it has been seen.           │
│  Removing it from history does NOT guarantee nobody saw it.     │
│  Rotation is non-negotiable. Do this first.                     │
│                                                                 │
│                                                                 │
│  STEP 2 — REMOVE THE FILE FROM TRACKING                         │
│  ─────────────────────────────────────────────────────────────  │
│  git rm --cached .env                                           │
│  echo ".env" >> .gitignore                                      │
│  git add .gitignore                                             │
│  git commit -m "chore: remove .env from tracking"               │
│  git push                                                       │
│                                                                 │
│  This removes the file from future commits. But the secret      │
│  still exists in your Git history.                              │
│                                                                 │
│                                                                 │
│  STEP 3 — CONSIDER HISTORY REWRITING (advanced)                 │
│  ─────────────────────────────────────────────────────────────  │
│  If the repo is public or the secret is high-value, you may     │
│  need to rewrite history to scrub the secret from all past      │
│  commits. Tools: BFG Repo-Cleaner or git filter-repo.           │
│  This is a complex operation. If this happens on a work         │
│  project, escalate to a senior engineer immediately.            │
│                                                                 │
│                                                                 │
│  PREVENT THIS WITH detect-secrets (pre-commit hook):            │
│  ─────────────────────────────────────────────────────────────  │
│  uv add --dev detect-secrets                                    │
│  uv run detect-secrets scan > .secrets.baseline                 │
│                                                                 │
│  Then in .pre-commit-config.yaml (introduced in Week 15):       │
│  - repo: https://github.com/Yelp/detect-secrets                 │
│    hooks:                                                       │
│    - id: detect-secrets                                         │
│                                                                 │
│  Pre-commit hooks and the full pre-commit framework are          │
│  covered in Week 15 (CI/CD). For now, the key habit is:         │
│  ALWAYS check git diff --staged before git commit.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Branching and Switching

**Now for the real power: parallel timelines.**

```bash
# See what branch you're on
git branch
```

Output:
```
* main
```

```bash
# Create a new branch AND switch to it
git switch -c feature/add-stock-fetch
```

Output:
```
Switched to a new branch 'feature/add-stock-fetch'
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   AFTER git switch -c                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  a1b2c3d ──▶ b4c5d6e ──▶ c7d8e9f                                │
│                           ▲                                     │
│                           │                                     │
│                     ┌─────┴─────┐                               │
│                     │   main    │                               │
│                     └───────────┘                               │
│                     ┌───────────────────────────┐               │
│                     │ feature/add-stock-fetch   │               │
│                     └───────────────────────────┘               │
│                           ▲                                     │
│                          HEAD                                   │
│                                                                 │
│  Both branches point to the SAME commit.                        │
│  HEAD now follows feature/add-stock-fetch.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now make changes on the feature branch.**

Add the following lines to the end of `main.py` in your editor:

```python

async def fetch_stock(symbol: str) -> dict:
    """Fetch stock price for a given symbol."""
    print(f"Fetching stock price for {symbol}...")
    await asyncio.sleep(1)
    return {"symbol": symbol, "price": 150.0}
```

```bash
git add main.py
git commit -m "Add stock price fetch function"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 AFTER COMMITTING ON FEATURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  a1b ──▶ b4c ──▶ c7d                  ◀── main                  │
│                    │                                            │
│                    └──▶ d1e            ◀── feature/add-stock    │
│                                            ▲                    │
│                                           HEAD                  │
│                                                                 │
│  main didn't move. feature has a new commit.                    │
│  The "official" timeline is safe.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Switch back to main:**

```bash
git switch main

# Look at main.py — the stock function is GONE!
cat main.py
```

> "The stock function disappeared! Don't panic. It's not deleted — it's on the other timeline. Switch back to `feature/add-stock-fetch` and it reappears. Your working directory changes to match whichever branch HEAD points to."

**But what if you have uncommitted changes when you need to switch?**

```
┌─────────────────────────────────────────────────────────────────┐
│                       GIT STASH                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO: You're mid-work on feature/add-stock-fetch.          │
│  Something urgent comes up — you need to switch to main.        │
│  But you have uncommitted changes that conflict.                │
│                                                                 │
│  BASIC WORKFLOW:                                                 │
│                                                                 │
│  git stash              ← Shelve your uncommitted changes       │
│                           (working directory becomes clean)     │
│                                                                 │
│  git switch main        ← Now succeeds                          │
│  (do whatever you needed to do on main)                         │
│                                                                 │
│  git switch feature/add-stock-fetch                             │
│  git stash pop          ← Restore shelved changes               │
│                           (your work-in-progress comes back)    │
│                                                                 │
│  Think of stash as a temporary, global clipboard for your       │
│  working directory — park your work, do something else,         │
│  come back. Stashed changes can be applied on any branch.       │
│                                                                 │
│                                                                 │
│  ADVANCED STASH OPERATIONS:                                     │
│  ──────────────────────────                                     │
│                                                                 │
│  git stash push -m "half-done auth refactor"                    │
│                         ← Named stash. The -m message makes     │
│                           the stash list readable.              │
│                                                                 │
│  git stash list         ← View ALL stashes (not just the top)   │
│                                                                 │
│    stash@{0}: On feature/auth: half-done auth refactor          │
│    stash@{1}: WIP on feature/news: partial news fetch           │
│    stash@{2}: On main: quick hotfix attempt                     │
│                                                                 │
│  git stash apply stash@{1}                                      │
│                         ← Restore a specific stash WITHOUT      │
│                           removing it from the list.            │
│                           Use when you want to apply the        │
│                           same stash to multiple branches.      │
│                                                                 │
│  git stash pop          ← apply stash@{0} + drop it             │
│                           (the shorthand for apply + drop)      │
│                                                                 │
│  git stash drop stash@{1}                                       │
│                         ← Delete a specific stash entry         │
│                           without applying it.                  │
│                                                                 │
│  ⚠️  git stash pop on a stash that conflicts with your current   │
│      working directory will produce a merge conflict. Resolve   │
│      it the same way as a branch conflict (section 3.5).        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Stash with a name (always prefer this over anonymous stash)
git stash push -m "half-done auth refactor"

# See all stashes
git stash list
# stash@{0}: On feature/auth: half-done auth refactor
# stash@{1}: WIP on feature/news: partial news fetch

# Apply a specific stash without removing it
git stash apply stash@{1}

# Remove a stash you no longer need
git stash drop stash@{1}

# Confirm remaining stashes
git stash list
# stash@{0}: On feature/auth: half-done auth refactor
```

**Practical branch naming conventions:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   BRANCH NAMING CONVENTIONS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  feature/add-user-auth      ← New feature                       │
│  fix/null-pointer-crash     ← Bug fix                           │
│  refactor/split-services    ← Code restructuring                │
│  docs/update-readme         ← Documentation changes             │
│  test/add-api-tests         ← Adding tests                      │
│                                                                 │
│  RULES:                                                         │
│  ├─ Use lowercase                                               │
│  ├─ Use hyphens, not spaces or underscores                      │
│  ├─ Prefix with category (feature/, fix/, etc.)                 │
│  └─ Be descriptive but concise                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Merge (Combining Timelines)

**Your feature branch is done. How do you bring it back into main?**

**Fast-forward vs merge commit — understand this first.**

When the branch being merged *into* has no new commits since the feature branch was created, Git can "fast-forward" — it simply moves the pointer forward without creating a merge commit, leaving a perfectly linear history. When both branches have diverged and each has new commits, Git creates a merge commit with two parents to record the convergence.

### Merge — Combining timelines with a record

```bash
# Make sure you're on the branch you want to merge INTO
git switch main

# Merge the feature branch into main
git merge feature/add-stock-fetch
```

```
┌─────────────────────────────────────────────────────────────────┐
│                        GIT MERGE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE MERGE:                                                  │
│                                                                 │
│  main:    A ──▶ B ──▶ C                                         │
│                        │                                        │
│  feature:              └──▶ D ──▶ E                             │
│                                                                 │
│                                                                 │
│  AFTER git merge feature (while on main):                       │
│                                                                 │
│  main:    A ──▶ B ──▶ C ──────────▶ M   (merge commit)          │
│                        │            ▲                           │
│  feature:              └──▶ D ──▶ E─┘                           │
│                                                                 │
│                                                                 │
│  M is a "merge commit" — it has TWO parents (C and E).          │
│  History shows exactly when and where the branch existed.       │
│                                                                 │
│  TIME MACHINE: "Two timelines were combined on this date.       │
│  Here's the record of the merge."                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> Rebase is another strategy for combining branches — it rewrites commits to produce a linear history instead of a merge commit. It is covered in the **Optional Supplementary Reading** at the end of Part 3. Recommended once you are fully comfortable with merge.

### Conflicts — When timelines contradict each other

```
┌─────────────────────────────────────────────────────────────────┐
│                    MERGE CONFLICTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What happens when TWO branches changed the SAME LINE?          │
│                                                                 │
│  main:     changed line 5 to "return 42"                        │
│  feature:  changed line 5 to "return 99"                        │
│                                                                 │
│  Git: "I don't know which one you want. YOU decide."            │
│                                                                 │
│  Git marks the file:                                            │
│  ┌──────────────────────────────────┐                           │
│  │  <<<<<<< HEAD                    │                           │
│  │      return 42                   │  ← What's on main         │
│  │  =======                         │                           │
│  │      return 99                   │  ← What's on feature      │
│  │  >>>>>>> feature/my-branch       │                           │
│  └──────────────────────────────────┘                           │
│                                                                 │
│  TO RESOLVE:                                                    │
│  1. Open the file                                               │
│  2. Pick the correct code (or combine both)                     │
│  3. Delete the <<<<<<, =======, >>>>>> markers                  │
│  4. git add the fixed file                                      │
│  5. git commit (finishes the merge)                             │
│                                                                 │
│  Conflicts are NORMAL. They're not errors. They're Git          │
│  asking for human judgment.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Resolving Conflicts — Tooling

**Manual marker editing works, but your editor can do this visually.**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONFLICT RESOLUTION TOOLS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VS CODE MERGE EDITOR (recommended for beginners):              │
│  ─────────────────────────────────────────────────              │
│  When a conflict exists, VS Code shows inline action buttons    │
│  above each conflicted block:                                   │
│                                                                 │
│  ● Accept Current Change    ← keep your version (HEAD)          │
│  ● Accept Incoming Change   ← keep the branch being merged in   │
│  ● Accept Both Changes      ← keep both, one after the other    │
│  ● Compare Changes          ← side-by-side diff view            │
│                                                                 │
│  Click the appropriate button for each conflict block.          │
│  Save the file. Then:                                           │
│    git add <resolved-file>                                      │
│    git commit                                                   │
│                                                                 │
│                                                                 │
│  TERMINAL: TAKING AN ENTIRE FILE FROM ONE SIDE                  │
│  ──────────────────────────────────────────────                 │
│  When you know you want ALL of one side for an entire file:     │
│                                                                 │
│  git checkout --ours main.py     ← keep the file as it is       │
│                                    on your current branch       │
│  git checkout --theirs main.py   ← take the file exactly as     │
│                                    it is on the merging branch  │
│                                                                 │
│  After either command:                                          │
│    git add main.py                                              │
│    git commit                                                   │
│                                                                 │
│  Use these when you are certain one side is completely right     │
│  and the other is completely wrong for a given file.            │
│                                                                 │
│                                                                 │
│  TERMINAL: git mergetool                                        │
│  ─────────────────────────                                      │
│  git mergetool        ← opens your configured merge tool        │
│                          for each conflicted file in sequence   │
│                                                                 │
│  Configure your preferred tool:                                 │
│  git config --global merge.tool vscode                          │
│  git config --global mergetool.vscode.cmd \                     │
│    'code --wait $MERGED'                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Reading History (git log)

When a bug appears and you need to know which commit introduced it, `git log` is your starting point. When you inherit a codebase, it tells you who changed what and why. When you're about to open a pull request, it tells you what your branch actually contains.

```bash
git log
```

Output:
```
commit e1f2a3b (HEAD -> main)
Author: Alice <alice@example.com>
Date:   Sun Feb 16 15:00:00 2025

    Add .gitignore

commit b4c5d6e
Author: Alice <alice@example.com>
Date:   Sun Feb 16 14:45:00 2025

    Add news fetch function

commit a1b2c3d
Author: Alice <alice@example.com>
Date:   Sun Feb 16 14:30:00 2025

    Add initial weather fetch function
```

**More useful formats:**

```bash
# Compact one-line view
git log --oneline
```

```
e1f2a3b (HEAD -> main) Add .gitignore
b4c5d6e Add news fetch function
a1b2c3d Add initial weather fetch function
```

```bash
# With branch visualization (the one you'll use most)
git log --oneline --graph --all
```

```
* f4e5d6c (feature/add-stock-fetch) Add stock price fetch function
| * e1f2a3b (HEAD -> main) Add .gitignore
|/
* b4c5d6e Add news fetch function
* a1b2c3d Add initial weather fetch function
```

> "This is your timeline map. The `*` marks commits. The lines show how branches diverge and merge. `HEAD -> main` tells you where you are right now. Get comfortable reading this — it's your project's story."

```
┌─────────────────────────────────────────────────────────────────┐
│            ⚠️  DETACHED HEAD — DON'T PANIC                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If you run git checkout <commit-hash> to inspect an old        │
│  version of your project, Git will warn you:                    │
│                                                                 │
│  "You are in 'detached HEAD' state"                             │
│                                                                 │
│  This means you are not on any named branch. Do not commit      │
│  while in this state — your commits will be orphaned and        │
│  difficult to recover. To return to safety, run:                │
│                                                                 │
│      git switch main                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### git blame — Who Wrote This Line?

When you inherit a codebase and encounter a function you don't understand, `git blame` tells you who last changed each line and in which commit. That commit message often explains the intent.

```bash
git blame main.py
```

Output:
```
a1b2c3d (Alice  2025-02-16 14:30:00 +0000  1) import asyncio
a1b2c3d (Alice  2025-02-16 14:30:00 +0000  2)
a1b2c3d (Alice  2025-02-16 14:30:00 +0000  3) async def fetch_weather(city: str) -> dict:
b4c5d6e (Bob    2025-02-17 09:15:00 +0000  4)     """Fetch weather for a given city."""
a1b2c3d (Alice  2025-02-16 14:30:00 +0000  5)     print(f"Fetching weather for {city}...")
c7d8e9f (Alice  2025-02-18 11:45:00 +0000  6)     await asyncio.sleep(1)
```

> "Column by column: commit hash, author, date, line number, content. Bob changed line 4 in commit `b4c5d6e`. Run `git show b4c5d6e` to find out exactly what he was doing."

```bash
# Blame only specific lines (useful for large files)
git blame -L 10,25 main.py    ← show lines 10 to 25 only

# Blame ignoring whitespace changes
git blame -w main.py
```

---

### git show — Inspect Any Commit in Full

`git log` tells you a commit exists. `git show` tells you exactly what that commit contained.

```bash
# Show the full diff of a specific commit
git show a1b2c3d
```

Output:
```
commit a1b2c3d4e5f6...
Author: Alice <alice@example.com>
Date:   Sun Feb 16 14:30:00 2025

    feat(weather): add initial weather fetch function

diff --git a/main.py b/main.py
new file mode 100644
index 0000000..3a4b5c6
--- /dev/null
+++ b/main.py
@@ -0,0 +1,13 @@
+import asyncio
+
+async def fetch_weather(city: str) -> dict:
...
```

```bash
# Show only a specific file as it existed at that commit
git show a1b2c3d:main.py

# Show the current HEAD's changes (last commit)
git show HEAD

# Show what main.py looked like three commits ago
git show HEAD~3:main.py
```

> "The `git show <hash>:<path>` syntax is especially useful when you want to compare a file's current state with a past version without checking out that commit. You can copy the output directly or redirect it: `git show HEAD~3:main.py > main_old.py`."

---

## 3.7 Git Tags — Marking Significant Points in History

**A branch pointer moves with every new commit. A tag is a permanent label on a specific commit — it never moves.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       GIT TAGS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT A TAG IS:                                                 │
│  ──────────────                                                 │
│  A named, permanent pointer to a specific commit.               │
│  Used to mark releases, milestones, or known-good states.       │
│                                                                 │
│  a1b ──▶ b4c ──▶ c7d ──▶ d1e ──▶ e2f                           │
│                    ▲              ▲                             │
│                  v1.0.0         v1.1.0                          │
│                 (tag, fixed)   (tag, fixed)                     │
│                                                                 │
│                                           ▲                    │
│                                         main                    │
│                                        (moves with commits)     │
│                                                                 │
│  Tags stay on their commit forever. main keeps advancing.       │
│                                                                 │
│                                                                 │
│  TWO TYPES OF TAGS:                                             │
│  ──────────────────                                             │
│                                                                 │
│  ANNOTATED TAG (recommended for releases):                      │
│  git tag -a v1.0.0 -m "First stable release"                   │
│  └─ Stores: tagger, date, message, optional GPG signature       │
│  └─ Shows up in: git show v1.0.0 with full metadata             │
│                                                                 │
│  LIGHTWEIGHT TAG (for temporary or local markers):              │
│  git tag v1.0.0-beta                                            │
│  └─ Just a name pointing to a commit. No metadata stored.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create an annotated release tag on the current commit
git tag -a v1.0.0 -m "Release 1.0.0: async weather + news fetch"

# Create a tag on a specific past commit
git tag -a v0.9.0 -m "Beta release" a1b2c3d

# List all tags
git tag

# Show tag details (annotated tags include the message and tagger)
git show v1.0.0

# Tags are NOT pushed automatically — you must push them explicitly
git push origin v1.0.0          ← push one specific tag
git push --tags                 ← push ALL local tags (use sparingly)

# Delete a local tag
git tag -d v1.0.0-beta

# Delete a remote tag
git push origin --delete v1.0.0-beta
```

**Checking out a tag — and the Detached HEAD connection:**

```bash
git checkout v1.0.0
# WARNING: You are in 'detached HEAD' state
```

> "Checking out a tag puts you in detached HEAD state — which you learned about in section 3.6. You're looking at a historical snapshot. Don't commit here. If you need to make a hotfix against a specific version, create a branch from the tag: `git switch -c hotfix/v1.0.1 v1.0.0`."

**Connection to this course:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 TAGS AND THIS COURSE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  This week: create tags manually to mark project milestones.    │
│                                                                 │
│  Week 15 (CI/CD): Conventional commits drive automated          │
│  version bumping and tagging via tools like python-semantic-    │
│  release. The feat:, fix:, and BREAKING CHANGE: patterns        │
│  you are practising now will automatically generate the         │
│  correct tag (v1.0.0, v1.1.0, v2.0.0) based on commit history. │
│                                                                 │
│  For now: tag your first working version of every project.      │
│  git tag -a v0.1.0 -m "Initial working version"                 │
│  git push --tags                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Optional Supplementary Reading: Rebase

> **This section is not required for this week's project.** Read it once the basic merge workflow is automatic. Understanding rebase before merge is solid tends to cause confusion, not clarity.

### Rebase — Replaying your work on a new foundation

```bash
# While on the feature branch:
git switch feature/add-stock-fetch
git rebase main
```

```
┌─────────────────────────────────────────────────────────────────┐
│                        GIT REBASE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE REBASE:                                                 │
│                                                                 │
│  main:    A ──▶ B ──▶ C                                         │
│                        │                                        │
│  feature:              └──▶ D ──▶ E                             │
│                                                                 │
│                                                                 │
│  AFTER git rebase main (while on feature):                      │
│                                                                 │
│  main:    A ──▶ B ──▶ C                                         │
│                        │                                        │
│  feature:              └──▶ D' ──▶ E'                           │
│                                                                 │
│  D' and E' carry the same code changes as D and E, but are      │
│  new commit objects with new hashes — the original commits      │
│  were not moved, they were recreated on a new base.             │
│                                                                 │
│  Then fast-forward main:                                        │
│  git switch main                                                │
│  git merge feature   ← Fast-forward: no merge commit needed,   │
│                         just moves the pointer forward          │
│                                                                 │
│  main:    A ──▶ B ──▶ C ──▶ D' ──▶ E'                           │
│                                                                 │
│  History is a straight line. Cleaner, but rewrites commits.     │
│                                                                 │
│  TIME MACHINE: "These events always happened in this order."    │
│  (The original timeline was erased and rewritten.)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Golden Rule — with the mechanism that makes it matter:**

Rebasing rewrites commit hashes — D becomes D′, a completely different object with a different ID. If a teammate already pulled commit D and built work on top of it, their local history and the rebased remote history now permanently diverge — reconciling them requires a force-push and is a team emergency. This is why: **never rebase any commit another person has already seen.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   MERGE VS REBASE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MERGE                              REBASE                      │
│  ─────                              ──────                      │
│  Preserves full history             Rewrites history            │
│  Creates merge commits              Linear history, no merges   │
│  Safe for shared branches           NEVER rebase shared branches│
│  Default recommendation             For cleaning local work     │
│                                                                 │
│  RULE FOR THIS COURSE:                                          │
│  ─────────────────────                                          │
│  Use MERGE when combining branches.                             │
│  Use REBASE only to update YOUR local feature branch            │
│  with changes from main BEFORE opening a pull request.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Optional Supplementary Reading: Branching Strategies

> **This section is not required for this week's project.** Read it when the basic commit/branch/merge workflow feels automatic. It gives you a framework for deciding when and how to branch in real teams.

### Why a strategy matters

Without an agreed-upon strategy, teams produce branches called `fix-final-FINAL`, accumulate year-old branches on the remote, and push directly to main. A strategy is a shared agreement about what branches mean and when code moves between them.

```
┌─────────────────────────────────────────────────────────────────┐
│                   BRANCHING STRATEGIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GITHUB FLOW (recommended for this course and most teams):      │
│  ─────────────────────────────────────────────────────────────  │
│  One permanent branch: main (always deployable).               │
│  All work happens on short-lived feature branches.              │
│                                                                 │
│  1. Branch from main                                            │
│  2. Commit on your branch                                       │
│  3. Open a Pull Request                                         │
│  4. Review + merge to main                                      │
│  5. Delete the branch                                           │
│  6. Deploy main                                                 │
│                                                                 │
│  ✅ Simple, fast, suits continuous deployment.                   │
│  ✅ What you are practising in this course.                      │
│  ❌ Not ideal for managing multiple supported release versions.  │
│                                                                 │
│                                                                 │
│  TRUNK-BASED DEVELOPMENT (advanced teams, high CI discipline):  │
│  ─────────────────────────────────────────────────────────────  │
│  All developers commit directly to main (the "trunk")           │
│  or use very short-lived branches (< 1 day).                    │
│  Requires feature flags to hide incomplete work.                │
│                                                                 │
│  ✅ Maximum integration, minimum merge conflicts.                │
│  ❌ Demands rigorous automated testing and code review.          │
│                                                                 │
│                                                                 │
│  GIT FLOW (versioned software with scheduled releases):         │
│  ─────────────────────────────────────────────────────────────  │
│  Multiple permanent branches:                                   │
│  main (tagged releases), develop (integration),                 │
│  feature/*, release/*, hotfix/*                                 │
│                                                                 │
│  ✅ Good for desktop apps, libraries with version support.       │
│  ❌ Heavy overhead — most web services don't need it.            │
│  ℹ️  You will encounter it in legacy codebases.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Hotfix workflow — emergency patch to production

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOTFIX WORKFLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO: A critical bug is in production (main, tagged        │
│  v1.0.0). Your feature branch is mid-development and not        │
│  ready to ship. You cannot release the feature branch.          │
│                                                                 │
│  GITHUB FLOW HOTFIX:                                            │
│                                                                 │
│  1. Branch from main (the known-good version):                  │
│     git switch main && git pull                                 │
│     git switch -c hotfix/v1.0.1                                 │
│                                                                 │
│  2. Fix the bug with a minimal change. Commit.                  │
│     git commit -m "fix(api): handle null city in weather fetch" │
│                                                                 │
│  3. Open a PR against main. Expedited review.                   │
│     Merge to main.                                              │
│                                                                 │
│  4. Tag the hotfix release:                                     │
│     git switch main && git pull                                 │
│     git tag -a v1.0.1 -m "Hotfix: null city crash"             │
│     git push --tags                                             │
│                                                                 │
│  5. If you are using Git Flow: also merge the hotfix back        │
│     into develop to keep both branches consistent.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT 2 — BRANCH-COMMIT-MERGE               │
│                    (15 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without guidance, complete the following steps in the          │
│  weather-cli repo:                                              │
│                                                                 │
│  1. Create a new branch called feature/greet                    │
│  2. Add a greet(name: str) -> str function to main.py           │
│  3. Commit it with a valid conventional commit message           │
│  4. Switch back to main — confirm greet() is gone               │
│  5. Merge feature/greet into main                               │
│  6. Run git log --oneline --graph --all                         │
│     Sketch the shape you see on paper                           │
│                                                                 │
│  Discussion: Does your log show a merge commit? Why or why not? │
│  What would the log look like if main had a new commit between  │
│  steps 1 and 5?                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: COLLABORATION & CONVENTIONS

## GitHub Authentication Setup

Before you can push to GitHub, Git needs to prove to GitHub's servers that you are who you say you are. Choose **one** of the two methods below and complete it now — you will need it for section 4.2.

```
┌─────────────────────────────────────────────────────────────────┐
│              GITHUB AUTHENTICATION SETUP                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: HTTPS with a Personal Access Token (PAT)             │
│  ──────────────────────────────────────────────────             │
│  1. GitHub → Settings → Developer Settings                      │
│     → Personal access tokens → Tokens (classic)                 │
│  2. Generate new token → check the "repo" scope → Generate      │
│  3. Copy the token (it is shown only once — save it now)        │
│  4. On your first git push, enter:                              │
│       Username: your GitHub username                            │
│       Password: paste the token (not your account password)     │
│  5. Let your OS credential manager save it —                    │
│     you will not be asked again on this machine                 │
│                                                                 │
│  OPTION B: SSH Key (recommended for long-term use)              │
│  ─────────────────────────────────────────────────              │
│  1. Generate a key pair (run once per machine):                 │
│       ssh-keygen -t ed25519 -C "you@example.com"                │
│       Accept the default file path. Add a passphrase if         │
│       you want extra security.                                  │
│  2. Print your public key and copy it:                          │
│       cat ~/.ssh/id_ed25519.pub                                 │
│  3. GitHub → Settings → SSH and GPG keys → New SSH key          │
│     → paste the public key → Save                               │
│  4. Verify the connection:                                      │
│       ssh -T git@github.com                                     │
│       Expected: "Hi username! You've successfully authenticated" │
│  5. Use SSH URLs when adding remotes:                           │
│       git@github.com:username/repo.git                          │
│     instead of: https://github.com/username/repo.git            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.1 Remotes — Connecting to GitHub

**Right now your repository is LOCAL — only on your machine. Remotes let you share it.**

**Cloning an existing repository:**

Before creating your own remote, you need to know how to download one. To start working on an existing repository, use `git clone`:

```bash
git clone https://github.com/username/weather-cli.git
```

This creates a local copy of the entire repository — all commits, all branches, all history — with the remote already configured as `origin`. The result is a directory identical to what `git init` + `git push` would produce, already wired up and ready for `git pull` and `git push`.

```
┌─────────────────────────────────────────────────────────────────┐
│                AFTER git clone                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  weather-cli/          ← directory created by clone             │
│  ├── .git/             ← full repository history inside         │
│  ├── main.py           ← all tracked files checked out          │
│  └── ...                                                        │
│                                                                 │
│  Remote "origin" is pre-configured — no git remote add needed.  │
│  git pull and git push work immediately.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connecting YOUR local repository to a new GitHub remote:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      REMOTES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A REMOTE is a copy of your repository on another machine.      │
│  Usually a service like GitHub, GitLab, or Bitbucket.           │
│                                                                 │
│  YOUR MACHINE                        GITHUB                     │
│  ────────────                        ──────                     │
│  .git/ (local repo)    ◀──────▶     repo (remote repo)          │
│                          push                                   │
│                          pull                                   │
│                          fetch                                  │
│                                                                 │
│  The default remote name is "origin" (just a convention).       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Creating a repository on GitHub:**

Go to `github.com` → click **New** → enter the repository name → **keep all checkboxes unchecked** — do NOT initialize with a README, `.gitignore`, or license, because we already have those locally → click **Create repository**. GitHub will display a "Quick setup" page — copy the HTTPS or SSH URL.

> ⚠️ If you check any initialization checkbox, your first `git push` will fail because GitHub and your local repository will have divergent histories. Start empty.

```bash
# Connect your local repo to GitHub
git remote add origin https://github.com/yourusername/weather-cli.git

git remote -v  # -v = verbose: shows the fetch and push URL for each configured remote
```

Output:
```
origin  https://github.com/yourusername/weather-cli.git (fetch)
origin  https://github.com/yourusername/weather-cli.git (push)
```

### Fork Workflow — Contributing Without Push Access

**The previous section described adding a remote to a repository you own. But what if you want to contribute to a project you don't own?**

> "In open-source development, you cannot push directly to someone else's repository. The standard model is: copy the repository to your own GitHub account (a **fork**), work there, then propose your changes back via a pull request. This workflow uses two remotes simultaneously — `origin` (your fork) and `upstream` (the original)."

```
┌─────────────────────────────────────────────────────────────────┐
│               THE FORK-AND-PULL WORKFLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│       ORIGINAL REPO              YOUR FORK                      │
│   (you have no push access)   (you own this copy)               │
│   github.com/alice/weather    github.com/you/weather            │
│                                    ▲        │                   │
│              fork on GitHub ───────┘        │                   │
│                                             │                   │
│                                        git clone                │
│                                             │                   │
│                                             ▼                   │
│                                      YOUR MACHINE               │
│                                    (local repository)           │
│                                             │                   │
│             git push ──────────────────────▶│── to origin       │
│             git pull ◀──────────────────────│── from origin     │
│                                             │                   │
│             git fetch upstream ◀────────────│── from original   │
│                                                                 │
│  STEP BY STEP:                                                  │
│  ─────────────                                                  │
│  1. On GitHub: click Fork on the original repo                  │
│     → creates github.com/you/weather (your copy)                │
│                                                                 │
│  2. Clone YOUR fork (not the original):                         │
│     git clone https://github.com/you/weather.git                │
│     (this automatically sets up origin = your fork)             │
│                                                                 │
│  3. Add the original as "upstream":                             │
│     git remote add upstream https://github.com/alice/weather.git│
│     git remote -v                                               │
│       origin   https://github.com/you/weather.git (fetch/push)  │
│       upstream https://github.com/alice/weather.git (fetch)     │
│                                                                 │
│  4. Do your work on a feature branch, push to YOUR fork:        │
│     git switch -c feature/my-improvement                        │
│     (work, commit...)                                           │
│     git push -u origin feature/my-improvement                   │
│                                                                 │
│  5. Open a PR on GitHub: base = alice/weather:main,             │
│     compare = you/weather:feature/my-improvement                │
│                                                                 │
│  6. Keep your fork in sync with the original as others merge:   │
│     git fetch upstream                                          │
│     git switch main                                             │
│     git merge upstream/main                                     │
│     git push origin main     ← keep your fork's main updated   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Full setup sequence from scratch
git clone https://github.com/you/weather.git
cd weather

# Connect to the original ("upstream")
git remote add upstream https://github.com/alice/weather.git

# Verify both remotes
git remote -v
# origin    https://github.com/you/weather.git (fetch)
# origin    https://github.com/you/weather.git (push)
# upstream  https://github.com/alice/weather.git (fetch)
# upstream  https://github.com/alice/weather.git (push)

# Sync your fork when the original has new commits
git fetch upstream                  # download upstream changes (no merge yet)
git switch main
git merge upstream/main             # incorporate them
git push origin main                # update your fork on GitHub
```

> "The names `origin` and `upstream` are just conventions — Git doesn't enforce them. But they are so universally used that deviating will confuse your teammates. `origin` always means the remote you push to. `upstream` always means the authoritative original you pull from."

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT — FORK WORKFLOW                       │
│                    (15 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Work through the following WITHOUT guidance:                   │
│                                                                 │
│  1. Answer from memory: in a fork workflow, what does           │
│     "origin" point to? What does "upstream" point to?           │
│                                                                 │
│  2. Fork your partner's weather-cli repository on GitHub        │
│     (or fork any public repo if working solo).                  │
│     Clone YOUR fork locally. Confirm with: git remote -v        │
│                                                                 │
│  3. Add the original repo as upstream. Verify with              │
│     git remote -v that both origin and upstream are listed.     │
│                                                                 │
│  4. Create a branch, make a small change, push to your fork.    │
│     Open a PR targeting your partner's main branch.             │
│                                                                 │
│  5. Your partner makes a new commit to their main branch.       │
│     Sync your fork: fetch upstream, merge into your local       │
│     main, push to origin. Run git log --oneline --graph --all   │
│     and describe what you see.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ SOLUTION — FORK WORKFLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. "origin" points to YOUR fork on GitHub (the copy you        │
│     own and can push to). "upstream" points to the ORIGINAL     │
│     repository that you forked from (which you cannot push to   │
│     directly without approval).                                 │
│                                                                 │
│  2-3. After git remote add upstream <url>:                      │
│     git remote -v shows four lines:                             │
│       origin    https://...you/weather.git (fetch)              │
│       origin    https://...you/weather.git (push)               │
│       upstream  https://...alice/weather.git (fetch)            │
│       upstream  https://...alice/weather.git (push)             │
│                                                                 │
│  5. After sync:                                                 │
│     git log --oneline --graph --all shows your partner's new    │
│     commit on upstream/main merging into your local main.       │
│     Your fork on GitHub is now identical to the original.       │
│     Your feature branch is unchanged.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Push, Pull, Fetch (Syncing Timelines)

```
┌─────────────────────────────────────────────────────────────────┐
│                 PUSH, PULL, FETCH                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  git push                                                       │
│  ─────────                                                      │
│  Upload YOUR commits to the remote.                             │
│  "Here's what I've done. Add it to the shared timeline."        │
│                                                                 │
│     LOCAL: A → B → C → D                                        │
│     REMOTE: A → B → C         ← needs D                         │
│     git push → REMOTE: A → B → C → D  ✅                        │
│                                                                 │
│                                                                 │
│  git fetch                                                      │
│  ──────────                                                     │
│  Download commits from the remote WITHOUT changing your files.  │
│  "Let me see what others have done. Don't touch my work yet."   │
│                                                                 │
│                                                                 │
│  git pull                                                       │
│  ─────────                                                      │
│  git fetch + git merge in one step.                             │
│  "Download what others did AND merge it into my branch."        │
│                                                                 │
│     REMOTE: A → B → C → D → E   ← teammate added E              │
│     LOCAL:  A → B → C → D       ← you're behind                 │
│     git pull → LOCAL: A → B → C → D → E  ✅                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# First push — set upstream tracking
git push -u origin main
# -u means "remember that local main tracks remote main"

# After the first push, just:
git push

# Get updates from the team:
git pull
```

---

## 4.3 Pull Request Workflow (The Review Gate)

**In professional development, you NEVER push directly to main.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE PULL REQUEST WORKFLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Create a feature branch locally                             │
│     git switch -c feature/add-stock-fetch                       │
│                                                                 │
│  2. Make commits on the feature branch                          │
│     (do your work, commit often)                                │
│                                                                 │
│  3. Push the feature branch to the remote                       │
│     git push -u origin feature/add-stock-fetch                  │
│                                                                 │
│  4. Open a Pull Request (PR) on GitHub                          │
│     "I'd like to merge feature/add-stock-fetch into main"       │
│                                                                 │
│  5. Team reviews the code                                       │
│     Comments, suggestions, requests for changes                 │
│                                                                 │
│  6. Address feedback (more commits on the same branch)          │
│     git add, git commit, git push                               │
│                                                                 │
│  7. PR approved → Merge into main (via GitHub UI)               │
│                                                                 │
│  8. Delete the feature branch (it served its purpose)           │
│     git switch main                                             │
│     git pull                                                    │
│     git branch -d feature/add-stock-fetch                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the complete flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  LOCAL                            GITHUB                        │
│  ─────                            ──────                        │
│                                                                 │
│  1. git switch -c feature/xyz                                   │
│  2. (write code, git add, commit)                               │
│  3. git push -u origin feature/xyz ──────▶ Branch appears       │
│                                            on GitHub            │
│                                                                 │
│                                    4. Open Pull Request         │
│                                       (on GitHub web UI)        │
│                                                                 │
│                                    5. Reviewer reads code       │
│                                       leaves comments           │
│                                                                 │
│  6. Fix issues, commit, push ────────────▶ PR updates           │
│                                            automatically        │
│                                                                 │
│                                    7. Reviewer approves ✅      │
│                                       → Merge to main           │
│                                                                 │
│  8. git switch main                                             │
│     git pull ◀─────────────────── main now has your code        │
│     git branch -d feature/xyz                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Linking Commits and PRs to GitHub Issues

> "GitHub Issues are tickets that track bugs, features, and tasks. Your commits and PRs can reference and close issues automatically using specific keywords."

```
┌─────────────────────────────────────────────────────────────────┐
│              GITHUB ISSUES INTEGRATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  KEYWORDS THAT AUTO-CLOSE AN ISSUE ON MERGE:                    │
│  ─────────────────────────────────────────────                  │
│  closes #42       fixes #42       resolves #42                  │
│                                                                 │
│  IN A COMMIT MESSAGE:                                           │
│  git commit -m "fix(weather): handle null API response          │
│                                                                 │
│  closes #42"                                                    │
│                                                                 │
│  → Issue #42 closes automatically when this commit lands        │
│    on the default branch.                                       │
│                                                                 │
│  IN A PR DESCRIPTION:                                           │
│  "This PR fixes #42 and relates to #38."                        │
│  → Issue #42 closes when the PR is merged.                      │
│  → Issue #38 gets a cross-reference link (not closed).          │
│                                                                 │
│  CROSS-REFERENCING (without closing):                           │
│  relates to #38   see #38   part of #38                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "A pull request isn't just a merge mechanism. It's a CONVERSATION about code. It's where you explain WHAT you did, WHY you did it, and others verify it's correct. This is how professional teams work."

---

## 4.4 Conventional Commits (feat, fix, chore)

**How do you write a commit message? Not like this:**

```
❌ "fixed stuff"
❌ "update"
❌ "wip"
❌ "asdlfkjasdlf"
❌ "final commit I promise"
```

**Conventional Commits are a STANDARD FORMAT for commit messages.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CONVENTIONAL COMMIT FORMAT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  <type>(<scope>): <short description>                           │
│                                                                 │
│  Optional body (longer explanation, wrap at 72 chars)           │
│                                                                 │
│  Optional footer (breaking changes, issue references)           │
│                                                                 │
│                                                                 │
│  TYPES (core three — full taxonomy in Week 15):                 │
│  ───────────────────────────────────────────────                │
│  feat:      A new feature                                       │
│  fix:       A bug fix                                           │
│  chore:     Maintenance (dependencies, configs, tooling)        │
│                                                                 │
│                                                                 │
│  EXAMPLES:                                                      │
│  ─────────                                                      │
│  feat(weather): add weather fetch function                      │
│  fix(api): handle null response from weather service            │
│  chore(deps): update httpx to 0.27.2                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does this matter? Semantic Versioning.**

These commit type conventions map directly to version numbers — `fix:` increments the patch, `feat:` increments the minor version, breaking changes increment the major — and tools in Week 15 will automate this for you.

**Let's rewrite our earlier commits:**

```bash
# ❌ Before (what we wrote earlier):
# "Add initial weather fetch function"
# "Add news fetch function"
# "Add .gitignore"

# ✅ Proper conventional commits:
# "feat(weather): add initial weather fetch function"
# "feat(news): add news article fetch function"
# "chore: add .gitignore with Python defaults"
```

---

## 4.5 Code Review — A Worked Example

Code review is how professional teams verify each other's work. You'll practice it in Week 3 when you begin working on the Task Manager API with a partner.

For now, study this single worked example — it shows the difference between a review that helps and one that damages the person's confidence without improving the code.

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKED EXAMPLE: CODE REVIEW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PR TITLE: feat(weather): add async weather and news fetch      │
│                                                                 │
│  PR DESCRIPTION:                                                │
│  ───────────────                                                │
│  ## What                                                        │
│  Added fetch_weather() and fetch_news() with full async         │
│  support and type hints throughout.                             │
│                                                                 │
│  ## Why                                                         │
│  The CLI needs concurrent data fetching for the Week 2          │
│  aggregator project.                                            │
│                                                                 │
│  ## How to test                                                 │
│  uv run python main.py                                          │
│  Expected: both fetches complete within ~1 second (concurrent). │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REVIEW COMMENT 1 — Exemplary:                                  │
│  ──────────────────────────────                                 │
│  "Using asyncio.gather() here is the right call — both fetches  │
│   run concurrently. Have you considered adding a timeout        │
│   parameter in case the external API is slow or unreachable?"   │
│                                                                 │
│  → Specific. Explains the reasoning. Asks a question rather     │
│    than issuing a demand. Invites the author to think.          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REVIEW COMMENT 2 — Destructive:                                │
│  ─────────────────────────────────                              │
│  "This is wrong. You should have used a class for this."        │
│                                                                 │
│  → No reason given. Criticises the choice, not the code.        │
│    Offers no path forward. Author learns nothing.               │
│                                                                 │
│  REVIEW COMMENT 2 — Improved:                                   │
│  ─────────────────────────────                                  │
│  "Consider wrapping these in a class — it would make the        │
│   shared httpx.AsyncClient easy to reuse across calls. What     │
│   do you think? Happy to discuss if you're not sure."           │
│                                                                 │
│  → Explains the reasoning. Frames it as a suggestion.           │
│    Leaves the decision to the author. Opens a dialogue.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 IDE Git Integration & GUI Clients

**Git is not CLI-only. Knowing both gives you options; knowing only one limits you.**

### VS Code Source Control Panel

> "Every command you have learned in the terminal maps directly to a UI action in VS Code. The panel is not a replacement for understanding the CLI — it is a faster interface for operations you already understand."

```
┌─────────────────────────────────────────────────────────────────┐
│              VS CODE SOURCE CONTROL PANEL                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPEN IT:  Ctrl+Shift+G  (Windows/Linux)                        │
│            Cmd+Shift+G   (macOS)                                │
│            Or click the branch icon in the left sidebar.        │
│                                                                 │
│  WHAT YOU CAN DO WITHOUT LEAVING THE EDITOR:                    │
│  ──────────────────────────────────────────                     │
│  Stage a file        ← click the + icon next to the filename    │
│  Unstage a file      ← click the − icon                         │
│  Stage a single line ← open file, right-click a changed line,  │
│                        "Stage Selected Ranges"                  │
│  Commit              ← type message in the text box, Ctrl+Enter │
│  View diff inline    ← click a changed file → side-by-side view │
│  Switch branch       ← click branch name in the status bar      │
│  Push / Pull         ← "…" menu at the top of the panel         │
│                                                                 │
│  MERGE CONFLICT EDITOR:                                         │
│  ───────────────────────                                        │
│  When a conflict exists, VS Code shows above each conflict:     │
│  [Accept Current] [Accept Incoming] [Accept Both] [Compare]     │
│  Click to resolve. Save. Stage the file. Commit.                │
│                                                                 │
│  This is the same workflow as manual marker editing but         │
│  with visual feedback — you see exactly which code wins.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The VS Code source control panel is particularly useful for staging individual lines within a file — something that requires `git add -p` on the command line (see Going Further appendix). Click the file, find the specific change, right-click, and stage just that hunk."

### GUI Git Clients — When and Why

```
┌─────────────────────────────────────────────────────────────────┐
│                 GUI CLIENTS                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GitHub Desktop   ← simplest, great for beginners, free         │
│  GitKraken        ← visual branch graphs, team features         │
│  Sourcetree       ← feature-complete, free (Atlassian)          │
│  Fork             ← fast, clean macOS/Windows UI                │
│                                                                 │
│  PROS:                                                          │
│  ├─ Visual branch graphs are genuinely clearer than             │
│     git log --oneline --graph                                   │
│  ├─ Staging hunks / individual lines is more intuitive          │
│  └─ Merge conflict resolution is easier to visualise            │
│                                                                 │
│  CONS:                                                          │
│  ├─ CLI knowledge is universal — GUI is tool-specific           │
│  ├─ Remote servers (CI, Docker, SSH) have no GUI                │
│  └─ Complex operations (rebase, reflog) are harder in UI        │
│                                                                 │
│  RECOMMENDATION:                                                │
│  Learn the CLI thoroughly first (this course). Use the VS Code  │
│  panel for daily staging/committing. Try a GUI client if you    │
│  find visualising branch graphs helpful.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: PROJECT MANAGEMENT WITH UV

## 5.1 The "It Works on My Machine" Problem

**Remember the Alice-and-Bob disaster from the opening?**

The core issue was: Alice's project DEPENDS on specific packages at specific versions, running on a specific Python version, and NONE of that information was captured anywhere.

```
┌─────────────────────────────────────────────────────────────────┐
│               THE DEPENDENCY PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR CODE NEEDS:                                               │
│  ────────────────                                               │
│  1. A specific Python version (3.12, not 3.9)                   │
│  2. External packages (httpx, pytest, mypy)                     │
│  3. Specific versions of those packages (httpx 0.27, not 0.23)  │
│  4. Those packages' dependencies (httpx needs httpcore, etc.)   │
│  5. All of this ISOLATED from other projects on your machine    │
│                                                                 │
│  WITHOUT A TOOL:                                                │
│  ────────────────                                               │
│  You'd have to tell every teammate:                             │
│  "Install Python 3.12, then pip install httpx==0.27.2,          │
│   pytest==8.3.4, mypy==1.13.0, oh and also httpcore==1.0.7,     │
│   anyio==4.8.0, certifi==2025.1.31, idna==3.10,                 │
│   sniffio==1.3.1, h11==0.14.0..."                               │
│                                                                 │
│  That's 10+ packages for a simple project. A real project       │
│  can have HUNDREDS. This is madness.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

A virtual environment is a self-contained directory — `.venv/` — that contains its own copy of the Python interpreter and its own package installation folder. When you run code inside it, Python sees only the packages in that environment, not anything installed globally on your machine.

**The Kitchen Analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE KITCHEN ANALOGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your project is a RECIPE.                                      │
│                                                                 │
│  pyproject.toml  = The recipe card                              │
│                    "You need flour, sugar, eggs"                │
│                    (WHAT you need, loosely)                     │
│                                                                 │
│  uv.lock         = The exact shopping list                      │
│                    "King Arthur flour 5lb, Domino sugar 2lb,    │
│                     Pete and Gerry's eggs dozen, bought at      │
│                     Trader Joe's on Feb 16"                     │
│                    (EXACT brands, versions, sources)            │
│                                                                 │
│  .venv/          = Your kitchen, stocked with ingredients       │
│                    (The actual packages installed on disk)      │
│                                                                 │
│  uv              = Your smart kitchen assistant                 │
│                    Reads the recipe, makes the shopping list,   │
│                    goes shopping, stocks the kitchen.           │
│                    All in seconds.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 uv — The Modern Python Project Manager

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT IS UV?                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  uv is an all-in-one Python project manager.                    │
│  Operations that take pip 20–30 seconds typically complete      │
│  in under 2 seconds with uv.                                    │
│                                                                 │
│  IT REPLACES:                                                   │
│  ├─ pip            (installing packages)                        │
│  ├─ venv           (creating virtual environments)              │
│  ├─ pip-tools      (locking dependencies)                       │
│  ├─ pyenv          (managing Python versions)                   │
│  └─ pipx           (running tools)                              │
│                                                                 │
│  ONE TOOL instead of five.                                      │
│                                                                 │
│  Install: https://docs.astral.sh/uv/                            │
│                                                                 │
│  macOS/Linux:  curl -LsSf https://astral.sh/uv/install.sh | sh  │
│  Windows:      powershell -c "irm https://astral.sh/uv/         │
│                install.ps1 | iex"                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Starting a Project (uv init)

**Let's create a proper project from scratch.**

```bash
# Create a new project
uv init weather-cli
cd weather-cli
```

**What did uv create?**

```bash
ls -la
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   uv init OUTPUT                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  weather-cli/                                                   │
│  ├── .git/              ← Git repo (uv initializes Git!)        │
│  ├── .gitignore         ← Pre-configured for Python             │
│  ├── .python-version    ← Pinned Python version                 │
│  ├── README.md          ← Empty readme                          │
│  ├── main.py            ← Sample Python file                    │
│  └── pyproject.toml     ← Project configuration                 │
│                                                                 │
│  Notice: uv already ran git init for you AND created            │
│  a sensible .gitignore. Everything we learned in Parts 1–4      │
│  applies immediately.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Look at the generated pyproject.toml:**

```toml
[project]
name = "weather-cli"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []
```

> "This is the RECIPE CARD for your project. Right now it's empty — no ingredients (dependencies) yet. Let's add some."

**Look at .python-version:**

```
3.12
```

> "This file tells uv (and other tools) exactly which Python version this project requires. Anyone who clones the project will use the same version."

---

## 5.4 Managing Dependencies (uv add, uv remove)

**Adding packages is one command:**

```bash
# Add httpx — you'll use this in the Week 2 mini-project to fetch API data.
# We'll cover httpx's full API in Week 8 (HTTP Client Fundamentals).
# For the mini-project, you'll receive a usage template.
uv add httpx
```

Output:
```
Resolved 7 packages in 120ms
Prepared 7 packages in 50ms
Installed 7 packages in 10ms
 + anyio==4.8.0
 + certifi==2025.1.31
 + h11==0.14.0
 + httpcore==1.0.7
 + httpx==0.28.1
 + idna==3.10
 + sniffio==1.3.1
```

**What happened behind the scenes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    uv add httpx                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Added "httpx>=0.28.1" to pyproject.toml [dependencies]      │
│  2. Resolved ALL dependencies (httpx + everything httpx needs)  │
│  3. Created uv.lock with exact versions                         │
│  4. Created .venv/ (virtual environment) if it didn't exist     │
│  5. Installed everything into .venv/                            │
│                                                                 │
│  All in under 200 milliseconds. ⚡                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now check pyproject.toml:**

```toml
[project]
name = "weather-cli"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]
```

**Adding DEVELOPMENT dependencies (not shipped with your app):**

```bash
# pytest and mypy are dev tools — users of your code don't need them
uv add --dev pytest mypy
```

**Now pyproject.toml has a new section:**

```toml
[project]
name = "weather-cli"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[dependency-groups]
dev = [
    "mypy>=1.14.1",
    "pytest>=8.3.4",
]
```

```
┌─────────────────────────────────────────────────────────────────┐
│            DEPENDENCIES VS DEV-DEPENDENCIES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [project] dependencies        [dependency-groups] dev          │
│  ─────────────────────         ───────────────────────          │
│  Packages YOUR CODE needs      Packages YOU (the developer)     │
│  to run.                       need to develop.                 │
│                                                                 │
│  httpx    — making HTTP calls  pytest — running tests           │
│  pydantic — data validation    mypy   — type checking           │
│  fastapi  — web framework      ruff   — linting                 │
│                                                                 │
│  Ship to production? ✅ YES    Ship to production? ❌ NO         │
│                                                                 │
│  uv add httpx                  uv add --dev pytest              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Multiple Dependency Groups

> "The `dev` group is one of many groups you can define. Real projects typically separate concerns further — tools for documentation, tools only needed in CI, tools needed for deployment."

```toml
[project]
name = "weather-cli"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[dependency-groups]
dev = [
    "mypy>=1.14.1",
    "pytest>=8.3.4",
    "pytest-asyncio>=0.25.0",
    "ruff>=0.9.0",
]
docs = [
    "mkdocs>=1.6.0",
    "mkdocs-material>=9.5.0",
]
test = [
    "pytest>=8.3.4",
    "pytest-asyncio>=0.25.0",
    "pytest-cov>=6.0.0",
    "httpx>=0.28.1",          # for async test client
]
```

```bash
# Install ALL groups (default — what uv sync does)
uv sync

# Install only production + one specific group
uv sync --group docs          # install docs tools only
uv sync --only-group test     # install ONLY test tools (not dev)

# Install production dependencies only (no dev tools)
uv sync --no-dev
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHEN TO USE EACH GROUP                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  dev   → tools every developer needs every day                  │
│          (mypy, ruff, pytest, debugger extensions)              │
│                                                                 │
│  docs  → tools only needed when building documentation          │
│          (mkdocs, sphinx) — most devs never run these           │
│                                                                 │
│  test  → test-only dependencies for CI environments             │
│          where you want a minimal, reproducible test install    │
│                                                                 │
│  In CI (Week 15), your pipeline will run:                       │
│  uv sync --only-group test                                      │
│  uv run pytest                                                  │
│  This installs only what tests need, making CI faster.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Removing packages:**

```bash
# Remove a package
uv remove httpx

# Remove a dev dependency
uv remove --dev mypy
```

---

## 5.5 pyproject.toml — The Single Source of Truth

**Connection to Lectures 1–3: everything comes together here.**

```
┌─────────────────────────────────────────────────────────────────┐
│            pyproject.toml — THE SINGLE SOURCE OF TRUTH          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  pyproject.toml is ONE FILE that holds:                         │
│                                                                 │
│  1. Project metadata (name, version, description)               │
│  2. Python version requirement                                  │
│  3. Production dependencies                                     │
│  4. Development dependencies                                    │
│  5. Tool configuration (mypy, pytest, ruff settings)            │
│                                                                 │
│  Before pyproject.toml, you needed:                             │
│  ├─ setup.py            (project metadata)                      │
│  ├─ setup.cfg           (more metadata)                         │
│  ├─ requirements.txt    (production deps)                       │
│  ├─ requirements-dev.txt (dev deps)                             │
│  ├─ mypy.ini            (mypy config)                           │
│  ├─ pytest.ini          (pytest config)                         │
│  └─ .flake8             (linter config)                         │
│                                                                 │
│  That's 7 files. Now it's 1.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A complete pyproject.toml for this course:**

```toml
[project]
name = "weather-cli"
version = "0.1.0"
description = "Async CLI tool for fetching weather, news, and stock data"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[dependency-groups]
dev = [
    "mypy>=1.14.1",
    "pytest>=8.3.4",
    # pytest-asyncio and asyncio_mode configuration will be added in Week 2
]

# Tool configurations — remember mypy from Lecture 1?
[tool.mypy]
strict = true
```

> "Remember `mypy` from Lecture 1, when we learned type hints? Instead of a separate mypy.ini file, the configuration lives right here. ONE file, ONE source of truth for everything about your project."

---

## 5.6 uv.lock and uv run — Reproducibility in Action

### uv.lock — The Exact Recipe

```
┌─────────────────────────────────────────────────────────────────┐
│              pyproject.toml vs uv.lock                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  pyproject.toml says:           uv.lock says:                   │
│  ─────────────────              ────────────                    │
│  "httpx >= 0.28"                "httpx == 0.28.1"               │
│  (flexible — any version        "httpcore == 1.0.7"             │
│   0.28 or higher is fine)       "anyio == 4.8.0"                │
│                                 "certifi == 2025.1.31"          │
│                                 "h11 == 0.14.0"                 │
│                                 "idna == 3.10"                  │
│                                 "sniffio == 1.3.1"              │
│                                 (exact — every package,         │
│                                  every version, pinned)         │
│                                                                 │
│                                                                 │
│  KITCHEN ANALOGY:                                               │
│  ────────────────                                               │
│  pyproject.toml = "I need flour"                                │
│  uv.lock        = "King Arthur All-Purpose, 5lb bag, $6.99"     │
│                                                                 │
│                                                                 │
│  ⚠️ COMMIT uv.lock TO GIT.                                      │
│  This is how your teammates get the EXACT same versions.        │
│                                                                 │
│  ❌ DO NOT COMMIT .venv/ to Git.                                │
│  That's the actual installed packages — large, OS-specific.     │
│  uv recreates it from uv.lock.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

`uv sync` installs all dependencies from `uv.lock` into `.venv` without running any script — use it after cloning a project or after a teammate updates `uv.lock` and you need to refresh your environment.

### Virtual Environment Activation — What `uv run` Hides from You

> "Every time you run `uv run python main.py`, uv is silently activating your virtual environment, running the command inside it, and deactivating it. You never see this. But your IDE needs to see the virtual environment explicitly, and some workflows require manual activation. You need to understand what's happening under the hood."

```
┌─────────────────────────────────────────────────────────────────┐
│            WHAT IS A VIRTUAL ENVIRONMENT?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  .venv/ is a self-contained directory with:                     │
│  ├─ A copy of (or symlink to) the Python interpreter            │
│  ├─ Its own site-packages/ folder for installed packages        │
│  └─ Activation scripts that modify PATH                         │
│                                                                 │
│  When ACTIVATED:                                                │
│  "python" → runs .venv/bin/python (not system Python)           │
│  "pip"    → installs into .venv/site-packages/ (not globally)  │
│  Packages installed elsewhere are invisible                     │
│                                                                 │
│  When NOT ACTIVATED:                                            │
│  "python" → runs whatever Python is on your system PATH         │
│  Your project's packages are not accessible                     │
│                                                                 │
│  uv run handles activation transparently per command.           │
│  Manual activation affects your entire terminal session.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Manual activation:**

```bash
# macOS / Linux
source .venv/bin/activate

# Windows (Command Prompt)
.venv\Scripts\activate.bat

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# After activation, your shell prompt changes:
# (.venv) user@machine:~/weather-cli$

# Now python and pip point into the virtual environment:
which python          # → /home/user/weather-cli/.venv/bin/python
pip list              # → shows only packages installed in .venv

# Deactivate (return to system Python)
deactivate
```

> "Notice the `(.venv)` prefix in your terminal prompt after activation — this tells you which environment is active. Without it, you are using your system Python and your project packages are not accessible."

```
┌─────────────────────────────────────────────────────────────────┐
│            uv run vs MANUAL ACTIVATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  uv run python main.py          MANUAL ACTIVATION               │
│  ───────────────────────        ──────────────────              │
│  ✅ Activates for ONE command   ✅ Active for entire session     │
│  ✅ Auto-syncs packages first   ❌ No auto-sync                  │
│  ✅ Works everywhere            ❌ OS-specific command           │
│  ❌ New shell per command        ✅ REPL, debugger, scripts work │
│  ❌ IDE can't introspect it      ✅ IDE can use it for Intellisense│
│                                                                 │
│  USE uv run FOR:                USE MANUAL ACTIVATION FOR:      │
│  ───────────────                ──────────────────────          │
│  Running scripts                IDE Python interpreter path     │
│  Running pytest / mypy          Interactive debugging sessions  │
│  CI pipelines                   Running custom shell scripts    │
│  One-off commands                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Configuring VS Code to use your virtual environment:**

```bash
# Open the Command Palette: Ctrl+Shift+P (or Cmd+Shift+P on macOS)
# Type: Python: Select Interpreter
# Choose: .venv (the one in your project folder)
```

> "After selecting the interpreter, VS Code's IntelliSense, type checking, and import resolution will all use the packages inside `.venv`. If VS Code shows import errors even after `uv add httpx`, this is almost always the fix — VS Code is pointed at the wrong Python."

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT — VIRTUAL ENVIRONMENT                 │
│                    (10 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Do NOT use uv run for any step in this exercise.               │
│                                                                 │
│  1. Activate the virtual environment manually using the         │
│     command appropriate for your OS. Confirm the (.venv)        │
│     prefix appears in your shell prompt.                        │
│                                                                 │
│  2. Run: python --version                                       │
│     Where is this Python interpreter coming from?               │
│     (Hint: run "which python" on macOS/Linux or                 │
│     "where python" on Windows to find the full path.)           │
│                                                                 │
│  3. Run: python -c "import httpx; print(httpx.__version__)"     │
│     Does this work? Why?                                        │
│                                                                 │
│  4. Run: deactivate                                             │
│     Run the same command from step 3 again. What happens?       │
│     Why?                                                        │
│                                                                 │
│  5. In VS Code, open the Command Palette and set the Python     │
│     interpreter to your project's .venv. Confirm IntelliSense   │
│     recognises httpx (hover over an httpx import).              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ SOLUTION — VIRTUAL ENVIRONMENT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2. "which python" returns a path ending in .venv/bin/python,   │
│     NOT /usr/bin/python or /usr/local/bin/python.               │
│     The virtual environment's Python is at the front of PATH.   │
│                                                                 │
│  3. httpx imports and prints its version. It works because the  │
│     activated .venv has httpx installed in its site-packages.   │
│                                                                 │
│  4. After deactivate, the import fails:                         │
│     ModuleNotFoundError: No module named 'httpx'                │
│     The system Python does not have httpx. The packages live    │
│     exclusively inside .venv and are only accessible when       │
│     that environment is active.                                 │
│                                                                 │
│  This is exactly the "it works on my machine" problem that uv   │
│  solves — but now you understand WHY the isolation exists.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### uv run — Running Code

**`uv run` is how you execute code in your project:**

```bash
# Run your Python script
uv run python main.py

# Run pytest
uv run pytest

# Run mypy (type checking from Lecture 1!)
uv run mypy main.py
```

**What `uv run` does behind the scenes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      uv run                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Checks: Is pyproject.toml newer than uv.lock?               │
│     If yes → Updates the lockfile                               │
│                                                                 │
│  2. Checks: Is .venv/ in sync with uv.lock?                     │
│     If no → Installs/updates packages                           │
│                                                                 │
│  3. Runs your command inside the virtual environment            │
│                                                                 │
│  This means: uv run ALWAYS uses the correct packages            │
│  at the correct versions. No "forgot to pip install".           │
│  No "wrong version". It just works.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The complete teammate workflow — the Alice-Bob problem, solved:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE "IT WORKS ON MY MACHINE" FIX                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ALICE (creates the project):                                   │
│  ──────────────────────────────                                 │
│  uv init weather-cli                                            │
│  cd weather-cli                                                 │
│  uv add httpx                                                   │
│  uv add --dev pytest                                            │
│  # write code...                                                │
│  git add .                                                      │
│  git commit -m "feat: initial project setup"                    │
│  git push                                                       │
│                                                                 │
│  BOB (joins the project):                                       │
│  ────────────────────────                                       │
│  git clone https://github.com/alice/weather-cli.git             │
│  cd weather-cli                                                 │
│  uv run python main.py        ← THAT'S IT. ONE COMMAND.         │
│                                                                 │
│  uv reads pyproject.toml and uv.lock, creates .venv,            │
│  installs the EXACT same packages Alice has, runs the code.     │
│  No "which version?", no "pip install what?".                   │
│  It. Just. Works.                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.7 Python Version Management

**uv can also manage Python versions:**

```bash
# See available Python versions
uv python list

# Install a specific version
uv python install 3.12

# Pin your project to a specific version
uv python pin 3.12
# This writes "3.12" to .python-version
```

```
┌─────────────────────────────────────────────────────────────────┐
│               PYTHON VERSION PINNING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  .python-version file says: "3.12"                              │
│                                                                 │
│  When uv creates the virtual environment, it uses 3.12.         │
│  When Bob clones and runs uv run, it uses 3.12.                 │
│  If Bob doesn't have 3.12, uv tells him to install it.          │
│                                                                 │
│  No more "what Python version are you using?" conversations.    │
│                                                                 │
│  COMMIT .python-version TO GIT. ✅                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.8 Legacy Context: venv, pip, requirements.txt

You will encounter `pip install`, `python -m venv`, and `requirements.txt` in existing codebases and older tutorials. Know that uv replaces all of these. When you encounter them, you will understand what they are doing — we will revisit them briefly when you first encounter legacy code in practice.

---

## 5.9 Common Mistakes and Misconceptions

### Mistake 1: Committing .venv/ or __pycache__/

```bash
# ❌ WRONG: Adding the virtual environment to Git
git add .venv/  # This is HUNDREDS of megabytes of installed packages!

# ✅ CORRECT: These should be in .gitignore (uv does this for you)
# .gitignore:
# .venv/
# __pycache__/
```

> "The .venv/ folder is your local kitchen — it's rebuilt from the recipe (uv.lock) on each machine. Don't ship the whole kitchen, ship the recipe."

---

### Mistake 2: Giant commits

```bash
# ❌ WRONG: 3 days of work in one commit
git add .
git commit -m "finished everything"

# ✅ CORRECT: Small, focused commits — each tells one story
git add main.py
git commit -m "feat(weather): add async weather fetch function"

git add utils.py
git commit -m "refactor(utils): extract retry logic into helper"

git add tests/test_weather.py
git commit -m "test(weather): add unit tests for weather fetch"
```

> "Each commit should do ONE thing. If you can't describe it in one line, it's too big. Small commits are easier to review, easier to revert, and easier to understand."

---

### Mistake 3: Working directly on main

```bash
# ❌ WRONG: Making changes directly on main
git switch main
# edit files...
git commit -m "feat: new stuff"
git push

# ✅ CORRECT: Always use a feature branch
git switch -c feature/add-stock-fetch
# edit files...
git commit -m "feat(stock): add stock price fetcher"
git push -u origin feature/add-stock-fetch
# Open PR on GitHub → review → merge
```

---

### Mistake 4: Forgetting to commit uv.lock

```bash
# ❌ WRONG: .gitignore includes uv.lock
# Your teammates get DIFFERENT versions!

# ✅ CORRECT: Always commit uv.lock
git add pyproject.toml uv.lock
git commit -m "chore(deps): add httpx dependency"
```

---

### Mistake 5: Using time.sleep() in async code

> "Connection to Lecture 3: Remember, `time.sleep()` blocks the event loop. Same applies when you're running async code through `uv run`. The tools don't fix your code logic — they manage your project's environment."

---

### Mistake 6: Not pulling before starting new work

```bash
# ❌ WRONG: Start coding without checking for updates
git switch main
# immediately start new branch...

# ✅ CORRECT: Always pull first
git switch main
git pull                    # Get latest changes from the team
git switch -c feature/new-thing  # NOW branch from up-to-date main
```

---

## What to Commit, What to Ignore — Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMIT vs IGNORE — THE FINAL WORD                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ COMMIT THESE:                  ❌ IGNORE THESE:              │
│  ─────────────────                 ────────────────             │
│  pyproject.toml                    .venv/                       │
│  uv.lock                          __pycache__/                  │
│  .python-version                   *.pyc                        │
│  .gitignore                        .env                         │
│  README.md                         .DS_Store                    │
│  All source code (.py)             Thumbs.db                    │
│  Tests (tests/)                    dist/                        │
│  Configuration files               build/                       │
│                                    *.egg-info/                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│     ✦ PRACTICE CHECKPOINT 3 — ALICE-TO-BOB REPRODUCTION         │
│                    (15 minutes)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Work in pairs, or solo by playing both roles yourself.         │
│                                                                 │
│  PLAY ALICE:                                                    │
│  1. uv init my-api-project                                      │
│  2. cd my-api-project                                           │
│  3. uv add httpx                                                │
│  4. Write three lines of Python in main.py (print anything)     │
│  5. git add . && git commit -m "feat: initial project"          │
│  6. Push to a new empty GitHub repository                       │
│                                                                 │
│  PLAY BOB (or swap with a partner):                             │
│  1. git clone <your-repo-url>                                   │
│  2. cd my-api-project                                           │
│  3. uv run python main.py                                       │
│                                                                 │
│  GOAL: Bob's output is identical to Alice's with exactly        │
│  one command after cloning — no pip install, no version         │
│  negotiation, no "which Python do you have?"                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Cards

```
┌─────────────────────────────────────────────────────────────────┐
│                    GIT QUICK REFERENCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IDENTITY (once per machine):                                   │
│      git config --global user.name "Your Name"                  │
│      git config --global user.email "you@example.com"           │
│                                                                 │
│  START A REPO:                                                  │
│      git init                                                   │
│      git clone <URL>            (download existing repo)        │
│                                                                 │
│  STAGE + COMMIT:                                                │
│      git add <file>             (stage specific file)           │
│      git add .                  (stage everything — use with    │
│                                  care, see section 3.2)         │
│      git commit -m "message"    (commit staged changes)         │
│                                                                 │
│  FILE OPERATIONS:                                               │
│      git mv old.py new.py       (rename — preserves history)    │
│      git rm file.py             (delete + stage removal)        │
│      git rm --cached file.py    (stop tracking, keep on disk)   │
│      git clean -n               (preview untracked deletions)   │
│      git clean -fd              (delete untracked files/dirs)   │
│                                                                 │
│  UNDO:                                                          │
│      git restore <file>         (discard working dir changes)   │
│      git restore --staged <file>(unstage a file)                │
│      git revert <hash>          (create new undo commit)        │
│      git reset --soft HEAD~1    (undo commit, keep staged)      │
│      git reset --mixed HEAD~1   (undo commit, keep unstaged)    │
│      git reset --hard HEAD~1    (⚠️  destroy commit + changes)  │
│                                                                 │
│  INSPECT:                                                       │
│      git status                 (what's staged/changed?)        │
│      git diff                   (what changed?)                 │
│      git log --oneline --graph  (timeline overview)             │
│      git show <hash>            (full diff of a commit)         │
│      git show <hash>:<file>     (file as it was at that commit) │
│      git blame <file>           (who last changed each line?)   │
│                                                                 │
│  BRANCHING:                                                     │
│      git branch                 (list branches)                 │
│      git switch -c name         (create + switch)               │
│      git switch main            (switch to main)                │
│      git merge feature-branch   (merge into current branch)     │
│      git checkout --ours <file> (take entire file from HEAD)    │
│      git checkout --theirs <file>(take entire file from merge)  │
│                                                                 │
│  TAGS:                                                          │
│      git tag -a v1.0.0 -m "msg" (create annotated tag)         │
│      git tag                    (list all tags)                 │
│      git show v1.0.0            (inspect a tag)                 │
│      git push origin v1.0.0    (push a specific tag)           │
│      git push --tags            (push all tags)                 │
│                                                                 │
│  STASH:                                                         │
│      git stash push -m "name"   (stash with label)              │
│      git stash list             (view all stashes)              │
│      git stash pop              (restore latest + remove it)    │
│      git stash apply stash@{N}  (restore specific, keep it)     │
│      git stash drop stash@{N}   (delete specific stash)         │
│                                                                 │
│  COLLABORATION:                                                 │
│      git remote add origin URL  (connect to GitHub)             │
│      git remote add upstream URL(add original for fork workflow)│
│      git remote -v              (list all remotes)              │
│      git push -u origin main    (first push)                    │
│      git push                   (subsequent pushes)             │
│      git pull                   (download + merge updates)      │
│      git fetch upstream         (download without merging)      │
│                                                                 │
│  USEFUL ALIASES (run once per machine):                         │
│      git config --global alias.st "status"                      │
│      git config --global alias.lg                               │
│        "log --oneline --graph --all"                            │
│      git config --global alias.sw "switch"                      │
│                                                                 │
│  CONVENTIONAL COMMIT FORMAT:                                    │
│      <type>(<scope>): <description>                             │
│      feat, fix, chore  (full taxonomy: Week 15)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    UV QUICK REFERENCE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CREATE A PROJECT:                                              │
│      uv init project-name                                       │
│                                                                 │
│  ADD DEPENDENCIES:                                              │
│      uv add httpx               (production dependency)         │
│      uv add --dev pytest        (development dependency)        │
│                                                                 │
│  REMOVE DEPENDENCIES:                                           │
│      uv remove httpx                                            │
│      uv remove --dev pytest                                     │
│                                                                 │
│  RUN CODE:                                                      │
│      uv run python main.py     (run script)                     │
│      uv run pytest             (run tests)                      │
│      uv run mypy main.py      (type check)                      │
│                                                                 │
│  SYNC ENVIRONMENT:                                              │
│      uv sync                   (install deps from lockfile)     │
│      uv lock                   (update lockfile)                │
│                                                                 │
│  PYTHON VERSIONS:                                               │
│      uv python install 3.12   (install Python)                  │
│      uv python pin 3.12       (pin for project)                 │
│                                                                 │
│  KEY FILES:                                                     │
│      pyproject.toml            (project config — commit ✅)     │
│      uv.lock                   (exact versions — commit ✅)     │
│      .python-version           (Python pin — commit ✅)         │
│      .venv/                    (local env — DO NOT commit ❌)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Models

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  GIT = TIME MACHINE FOR YOUR CODE                               │
│                                                                 │
│  Save snapshots. Travel back in time. Experiment in parallel    │
│  timelines. Merge the good ideas. Collaborate via the cloud.    │
│                                                                 │
│  ┌──────────┐  git add  ┌──────────┐  git commit  ┌─────────┐   │
│  │ Working  │ ────────▶ │ Staging  │ ──────────▶  │  Repo   │   │
│  │   Dir    │           │  Area    │              │ (.git/) │   │
│  └──────────┘           └──────────┘              └─────────┘   │
│                                                                 │
│  Branch = Parallel timeline.     Merge = Combine timelines.     │
│  Remote = Cloud backup.          PR = Propose a merge.          │
│  Stash = Temporary clipboard.    Revert = Safe undo.            │
│                                                                 │
│                                                                 │
│  UV = SMART KITCHEN ASSISTANT FOR YOUR PROJECT                  │
│                                                                 │
│  Reads the recipe (pyproject.toml). Makes the exact shopping    │
│  list (uv.lock). Stocks the kitchen (.venv). All in seconds.    │
│                                                                 │
│  ┌──────────────┐     ┌──────────┐     ┌──────────┐             │
│  │ pyproject    │ ──▶ │ uv.lock  │ ──▶ │  .venv/  │             │
│  │ .toml        │     │ (exact   │     │ (actual  │             │
│  │ (what you    │     │  versions│     │  packages│             │
│  │  need)       │     │  pinned) │     │  on disk)│             │
│  └──────────────┘     └──────────┘     └──────────┘             │
│       commit ✅          commit ✅        ignore ❌              │
│                                                                 │
│                                                                 │
│  TOGETHER:                                                      │
│  Git tracks your code AND your pyproject.toml + uv.lock.        │
│  Anyone who clones your repo gets your code + your exact        │
│  environment. No surprises.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 2, LECTURE 1 (Debugging):                                 │
│  └─ git diff and git log help isolate when bugs were            │
│     introduced. "It worked in commit X, broke in commit Y."     │
│                                                                 │
│  WEEK 2, LECTURE 2 (Testing):                                   │
│  └─ uv run pytest becomes your daily command.                   │
│     pytest-asyncio and asyncio_mode are added to pyproject.toml │
│     at this point. Tests live in version control alongside code. │
│                                                                 │
│  WEEK 2 MINI-PROJECT (Async CLI Aggregator):                    │
│  └─ You'll build it using uv for dependencies (httpx,           │
│     pytest-asyncio) and Git for version control.                │
│     Everything from Lectures 1–4 comes together.                │
│                                                                 │
│  WEEK 3 (FastAPI):                                              │
│  └─ FastAPI projects live in Git repos, managed by uv.          │
│     Multiple files, proper structure, team collaboration.        │
│     Code review is practiced for the first time in pairs.       │
│                                                                 │
│  WEEK 15 (CI/CD):                                               │
│  └─ GitHub Actions will run uv run pytest, uv run mypy,         │
│     uv run ruff automatically on every push.                    │
│     Full conventional commits taxonomy introduced.              │
│     Conventional commits drive automated releases.              │
│                                                                 │
│  EVERY SINGLE PROJECT from now until Week 16:                   │
│  └─ Uses Git + uv. These are not optional tools.                │
│     They are the foundation of your professional workflow.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Going Further (Appendix)

**`git add -p` — Interactive Patch Mode**

Once the basic add-commit cycle is automatic, `git add -p` (patch mode) lets you stage individual *hunks* within a single file — so if you changed ten lines in `main.py` but only five belong in this commit, you can select exactly those five. Learn this once `git add <file>` feels completely natural.

---

**`git cherry-pick` — Apply a Specific Commit to Another Branch**

```bash
# Apply one specific commit to your current branch
git cherry-pick a1b2c3d
```

```
┌─────────────────────────────────────────────────────────────────┐
│                     git cherry-pick                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE CASE: A critical bug fix is on a feature branch.           │
│  You need it on main before the feature is ready to ship.       │
│                                                                 │
│  main:     A ──▶ B ──▶ C                                        │
│                         │                                       │
│  feature:               └──▶ D(fix) ──▶ E ──▶ F                │
│                                                                 │
│  git switch main                                                │
│  git cherry-pick D                                              │
│                                                                 │
│  main:     A ──▶ B ──▶ C ──▶ D'(fix)                           │
│                                                                 │
│  D' is a NEW commit with a NEW hash containing the same         │
│  change as D. The original D still exists on feature.           │
│                                                                 │
│  ⚠️  If feature is later merged into main, D and D' represent   │
│      the same change. Git usually resolves this cleanly, but    │
│      it can produce subtle conflicts. Use sparingly.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**`git reflog` — Recovery After Destructive Operations**

```bash
git reflog
```

`git reflog` records every position HEAD has ever pointed to — including after `git reset --hard`, deleted branches, and detached HEAD states. If you run `git reset --hard` and immediately regret it, `git reflog` shows you the commit hash of what you lost. You can then `git checkout <hash>` or `git switch -c recovery-branch <hash>` to get it back.

```
# After an accidental git reset --hard:
git reflog
# HEAD@{0}: reset: moving to HEAD~1
# HEAD@{1}: commit: feat(stock): add stock price fetch function  ← this is the lost commit
# HEAD@{2}: commit: Add .gitignore

git switch -c recovery/stock-fetch HEAD@{1}
# The lost commit is now on a branch. Merge or cherry-pick it back.
```

---

**Git Internals — Blobs, Trees, and Commits**

This is optional depth. Understanding it explains why Git is fast, why `git add` is a separate step, and why rebasing rewrites history.

```
┌─────────────────────────────────────────────────────────────────┐
│                  GIT OBJECT MODEL                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Git stores four types of objects in .git/objects/:             │
│                                                                 │
│  BLOB (binary large object):                                    │
│  └─ The compressed contents of a single file.                   │
│     No filename, no path — just the content.                    │
│     Same content in two files → same blob (stored once).        │
│                                                                 │
│  TREE:                                                          │
│  └─ A directory listing: names + permissions + pointers         │
│     to blobs (files) and other trees (subdirectories).          │
│                                                                 │
│  COMMIT:                                                        │
│  └─ Pointer to ONE root tree (the full project snapshot)        │
│     + author + timestamp + message + parent commit hash(es)     │
│                                                                 │
│  TAG:                                                           │
│  └─ Annotated tags are also objects: name + message             │
│     + pointer to a commit.                                      │
│                                                                 │
│                                                                 │
│  WHY THIS MATTERS:                                              │
│  ──────────────────                                             │
│                                                                 │
│  Why git add is needed:                                         │
│  git add creates the blob objects. git commit creates the tree  │
│  and commit objects. Without git add, there is nothing to       │
│  commit — the objects don't exist yet.                          │
│                                                                 │
│  Why rebasing rewrites history:                                 │
│  A commit's hash is computed from its content + parent hash.    │
│  Rebase changes the parent → new hash → "new" commit.           │
│  The original still exists until garbage collected.             │
│                                                                 │
│  Why Git is fast:                                               │
│  Unchanged files share the same blob across commits.            │
│  No storage is wasted on identical content.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# See this for yourself — inspect raw git objects
git cat-file -t HEAD             # type of the HEAD object (commit)
git cat-file -p HEAD             # contents of the commit object
git cat-file -p HEAD^{tree}      # the root tree object
git cat-file -p main.py:HEAD     # the blob for main.py
```

---

**Git Hooks — Running Code Before or After Git Events**

Git hooks are scripts in `.git/hooks/` that run automatically at specific points in the Git workflow. The most common: `pre-commit` runs before every commit, letting you fail a commit if linting or type-checking fails.

```bash
# Create a simple pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
uv run ruff check .
if [ $? -ne 0 ]; then
    echo "Ruff found lint errors. Fix them before committing."
    exit 1
fi
EOF

chmod +x .git/hooks/pre-commit
```

> "This hook prevents commits that fail linting. The problem: `.git/hooks/` is not tracked by Git — every developer must set it up manually. In Week 15 (CI/CD), the `pre-commit` framework is introduced, which manages hooks declaratively via a `.pre-commit-config.yaml` file that IS committed to the repository."

---

**`.gitattributes` — Extended Reference**

Section 3.3 covers the essential `.gitattributes` patterns. Additional useful patterns:

```
# Prevent merge conflicts in auto-generated lock files
# by always taking the version being merged in
uv.lock merge=union

# Force specific files to always show diffs (even binary-looking)
*.json diff
*.svg  diff

# Export-ignore: exclude files from git archive (for releases)
tests/       export-ignore
.github/     export-ignore
```

---

**Signed Commits — Verified Identity**

When you push a commit to GitHub without signing, GitHub shows the author name you configured — but has no way to verify it's really you. Anyone can set `git config user.email alice@example.com` and commit as Alice.

GPG signing attaches a cryptographic signature to each commit. GitHub marks signed commits with a green "Verified" badge, and organisations can enforce signature requirements for protected branches.

```bash
# Generate a GPG key
gpg --full-generate-key      # choose RSA 4096, no expiry

# List your key and copy the key ID
gpg --list-secret-keys --keyid-format=long

# Configure Git to use it
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true

# Export the public key and add it to GitHub
gpg --armor --export <KEY_ID>
# paste output into GitHub → Settings → SSH and GPG keys → New GPG key
```

---

**Git LFS — Large File Storage**

Git history stores everything forever. A 50MB model file committed once remains in the repository forever, making every `git clone` slow for all future developers. Git LFS replaces large files in the repository with small pointer files, storing the actual content on a separate LFS server.

```bash
# Install LFS (once per machine)
git lfs install

# Track large file types
git lfs track "*.psd"
git lfs track "*.pkl"       # ML model files
git lfs track "*.parquet"   # large datasets

# .gitattributes is updated automatically — commit it
git add .gitattributes
git commit -m "chore: configure Git LFS for binary assets"
```

> "LFS is needed when your repository contains binary files that change frequently: design assets, ML model checkpoints, large datasets. Python source code projects almost never need it."

---

**Advanced Clone Options**

```bash
# Shallow clone — download only the latest commit, not full history
# Use case: CI pipelines where history is unnecessary and speed matters
git clone --depth 1 <url>

# Convert a shallow clone to a full clone later
git fetch --unshallow

# Sparse checkout — clone a repo but only materialise specific directories
# Use case: large monorepos where you only need one subdirectory
git clone --filter=blob:none --sparse <url>
cd repo
git sparse-checkout set path/to/subdirectory
```

---

**Repository Maintenance**

Git performs most maintenance automatically. For large repositories or after heavy history rewriting (rebases, filter-repo runs), you may want to trigger it manually:

```bash
git gc              # garbage collect: compress loose objects,
                    # remove unreachable objects, pack refs

git prune           # remove objects with no references
                    # (usually called by git gc automatically)

git count-objects -vH   # see how much storage the repo uses
```

> "You will almost never need to run these manually. If your `.git/` folder grows unexpectedly large (gigabytes), run `git gc --aggressive` and investigate what you have been committing."

---

**Merge Strategy Options**

When a merge conflict occurs and you know you want one side to win for all conflicts in a given merge (not just one file), you can pass a merge strategy option:

```bash
git merge -X ours feature-branch    # our version wins all conflicts
git merge -X theirs feature-branch  # their version wins all conflicts
```

> "This is not a substitute for reading conflicts — it is for cases like automated dependency updates where you know with certainty that the incoming version is correct. Use with caution. These flags do NOT apply to `git checkout --ours` and `--theirs`, which operate on individual files."

---

# Tech Debt Register

The following topics were **introduced, referenced, or forward-linked** in this lecture but are not fully taught here. Each item has a scheduled home in the roadmap.

**Deferred from this lecture (scope cuts):**

- **Full Conventional Commits taxonomy** (`docs:`, `style:`, `refactor:`, `test:`) → Week 15 (CI/CD), where automated release tooling makes the full taxonomy immediately actionable
- **Semantic versioning automation** (tools that read commit messages and bump version numbers) → Week 15 (CI/CD)
- **Code review behavioural norms** (full reviewer and author checklists) → Week 3–4, when students collaborate on the Task Manager API in pairs and the norms have a live context
- **Git rebase workflow** (optional supplementary reading this week) → revisited formally when team-based pull request hygiene is taught in Week 3–4
- **Python packaging artifacts in `.gitignore`** (`dist/`, `build/`, `*.egg-info/`) → Week 15, when publishing a package to PyPI becomes relevant
- **Merge strategy options `-X ours / -X theirs`** (introduced in Going Further) → covered more thoroughly in Week 3–4 when team conflicts are a daily occurrence
- **Platform-specific dependency markers** (`sys_platform == 'win32'`) → Week 15 (Packaging), where cross-platform distribution becomes relevant
- **Optional dependencies** (`[project.optional-dependencies]` extras syntax) → Week 15 (Packaging), when building installable packages with optional feature sets
- **Signed commits** (GPG key generation and enforcement) → introduced in Going Further; when to enforce is an organisational decision outside this course's scope

**Forward-referenced in this lecture:**

- **httpx API usage** (making actual GET/POST requests, reading responses, handling errors) → Week 8 (HTTP Client Fundamentals); a usage template is provided for the Week 2 mini-project
- **pytest configuration** (`[tool.pytest.ini_options]` in `pyproject.toml`) → Week 2 Lecture 2 (Testing Fundamentals); added to the project file at that point
- **pytest-asyncio and `asyncio_mode = "auto"`** → Week 2 Lecture 2; required for testing async functions and introduced alongside the first test suite
- **Full `pip` / `venv` / `requirements.txt` workflow** → revisited briefly when students first encounter a legacy codebase or tutorial during the course
- **GitHub Actions CI pipeline** (runs `uv run pytest`, `uv run mypy`, `uv run ruff` on every push) → Week 15 (CI/CD)
- **pre-commit framework** (`.pre-commit-config.yaml`, managing hooks as committed configuration) → Week 15 (CI/CD); introduced in Going Further this week at the conceptual level only
- **detect-secrets** (full pre-commit integration for secrets scanning) → Week 15 (CI/CD); introduced in section 3.3 at the concept level
- **BFG Repo-Cleaner and `git filter-repo`** (history rewriting to scrub committed secrets) → Week 15 (CI/CD); mentioned as emergency tools in section 3.3
- **Fork workflow and open-source contribution** (introduced in section 4.1) → students will use this outside the course when contributing to real open-source projects; the branching strategy supplementary reading reinforces it
- **`uv sync --only-group test`** (installing specific dependency groups in CI) → Week 15 (CI/CD); introduced in section 5.4, used in practice when building pipelines

**Skills requiring follow-up practice:**

- **`git add -p` (interactive patch mode)** → see Going Further appendix; learn once the basic add-commit cycle is automatic
- **`git reflog`** → introduced in Going Further; the safety net beneath the undo commands taught in section 3.2a; students should read this section before attempting any destructive `reset --hard`
- **`git bisect`** → binary-search through commit history to find which commit introduced a bug; Week 2 Debugging Methodology lecture
- **Branch protection rules on GitHub** (require PR review before merging to `main`) → Week 3–4, when the PR workflow is practised in teams
- **SSH agent forwarding** (using your local SSH key on remote servers) → Week 15, when deployment to cloud infrastructure is introduced
- **`git cherry-pick` in practice** (introduced in Going Further) → students will apply it naturally during Week 3–4 when hotfixes are needed across branches
- **Advanced `git stash` operations** (`list`, `apply`, `drop`, named stashes) → now covered in section 3.4; revisit once the basic workflow is solid