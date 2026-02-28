# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FRUSTRATION FIRST, TOOLS SECOND                                │
│  ───────────────────────────────                                │
│  Students must FEEL the pain of bad debugging habits before     │
│  we hand them proper tools. We'll break something. They'll      │
│  suffer. Then we'll show them the fast way.                     │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Debugging is abstract. We use a detective analogy throughout.  │
│  Every concept maps to criminal investigation.                  │
│                                                                 │
│  READING BEFORE DOING                                           │
│  ────────────────────                                           │
│  Before touching a debugger, students learn to READ.            │
│  Read the traceback. Read the variables. Read the flow.         │
│  Most bugs die the moment you actually look at the evidence.    │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Dataclasses (Lecture 2) → __repr__ is a debugging superpower   │
│  Custom exceptions (Lecture 2) → Chained tracebacks             │
│  Async (Lecture 3) → Async tracebacks look different            │
│  Type hints (Lecture 1) → mypy catches bugs before runtime      │
│                                                                 │
│  THE CENTRAL THESIS                                             │
│  ─────────────────                                              │
│  Every bug is a gap between what you THINK your code does       │
│  and what it ACTUALLY does. Debugging closes that gap.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   DEBUGGING METHODOLOGY                         │
│                     (3 Hour Lecture)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ Reading Unfamiliar Code First — The 5-Layer Strategy        │
│  ├─ 1.1 The Broken Program (Demonstration)                      │
│  ├─ 1.2 The Print Statement Trap                                │
│  └─ 1.3 The Detective Analogy                                   │
│                                                                 │
│  PART 2: READING THE EVIDENCE (50 min)                          │
│  ├─ The Execution Stack — Why Tracebacks Look the Way They Do   │
│  ├─ 2.1 Anatomy of a Traceback                                  │
│  │   └─ Recursive Tracebacks                                    │
│  ├─ ★ Practice Checkpoint 6                                     │
│  ├─ 2.2 Reading Bottom-Up (The Technique)                       │
│  ├─ ★ Practice Checkpoint 1                                     │
│  ├─ 2.3 Common Error Types and What They Tell You               │
│  ├─ ★ Practice Checkpoint 2                                     │
│  ├─ 2.4 Chained Exceptions (Connection to Lecture 2)            │
│  └─ 2.5 Async Tracebacks (Connection to Lecture 3)              │
│       └─ When the Bug IS in the Library                         │
│                                                                 │
│  PART 3: THE TOOLS (70 min)                                     │
│  ├─ 3.1 The IDE Debugger (Your Forensic Lab)                    │
│  ├─ 3.2 Breakpoints (Pausing Execution)                         │
│  ├─ 3.3 Stepping Through Code                                   │
│  ├─ ★ Practice Checkpoint 3                                     │
│  ├─ 3.4 Variable Inspection and Watch Expressions               │
│  ├─ 3.5 Conditional Breakpoints                                 │
│  ├─ 3.6 Debug Configurations                                    │
│  ├─ 3.7 pdb and breakpoint() (The Universal Fallback)           │
│  │   ├─ Post-Mortem Debugging (pdb.pm())                        │
│  │   └─ Attaching to a Running Process                          │
│  ├─ ★ Practice Checkpoint 7                                     │
│  ├─ 3.8 Logging — Debugging Without Stopping the Program        │
│  └─ ★ Practice Checkpoint 4                                     │
│                                                                 │
│  PART 4: THE METHOD                                             │
│  ├─ 4.1 Hypothesis-Driven Debugging                             │
│  ├─ 4.2 Binary Search Debugging                                 │
│  │   └─ Mutation Tracking                                       │
│  ├─ ★ Practice Checkpoint 5                                     │
│  ├─ 4.3 Rubber Duck Debugging                                   │
│  ├─ 4.4 When to Ask for Help (And How)                          │
│  ├─ 4.5 Assertions as Early Bug Detection                       │
│  ├─ ★ Practice Checkpoint 8                                     │
│  ├─ 4.6 Performance Debugging                                   │
│  ├─ ★ Practice Checkpoint 9                                     │
│  ├─ 4.7 Memory Debugging                                        │
│  ├─ ★ Practice Checkpoint 10                                    │
│  ├─ 4.8 Concurrency Bug Patterns                                │
│  ├─ ★ Practice Checkpoint 11                                    │
│  ├─ 4.9 Intermittent Bugs and Heisenbugs                        │
│  └─ 4.10 Environment Debugging                                  │
│                                                                 │
│  PART 5: READING OTHERS' CODE (5 min)                           │
│  └─ 5.1 You Already Did It                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## Reading Unfamiliar Code First — The 5-Layer Strategy

**Before touching a debugger — before running anything — spend 2 minutes reading the program. Here is how.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE 5-LAYER READING STRATEGY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1 — Structure (30 seconds)                               │
│  ─────────────────────────────────                              │
│  How many files? What are the folders?                          │
│  Look at filenames and the rough line count.                    │
│  Do NOT open every file yet.                                    │
│                                                                 │
│  LAYER 2 — Data (2 minutes)                                     │
│  ──────────────────────────                                     │
│  Find the classes or dataclasses first.                         │
│  "What are the NOUNS in this program?"                          │
│  Nouns = data structures = the foundation of everything else.   │
│                                                                 │
│  LAYER 3 — Entry Point (2 minutes)                              │
│  ─────────────────────────────────                              │
│  Find main() or the startup code.                               │
│  Read the top-level flow only.                                  │
│  Do NOT dive into function bodies yet.                          │
│                                                                 │
│  LAYER 4 — Signatures (3 minutes)                               │
│  ────────────────────────────────                               │
│  Scan every function: name, parameters, return type.            │
│  Still do NOT read bodies.                                      │
│  "What goes in, what comes out?"                                │
│                                                                 │
│  LAYER 5 — One Path (5 minutes)                                 │
│  ─────────────────────────────                                  │
│  Pick one realistic execution path.                             │
│  Trace it from start to finish through the signatures.          │
│  Follow only that one thread. Ignore everything else.           │
│                                                                 │
│  TOTAL: ~12 minutes to understand any reasonably-sized file.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Apply all five layers to `buggy_grades.py` before running it:**

```
Layer 1 — Structure:
  One file, ~80 lines. Manageable.

Layer 2 — Data:
  Student dataclass: name (str), scores (list[int]).
  It also has an average() method.

Layer 3 — Entry point:
  main() at the bottom. It calls four functions, labeled Stage 1–4.
  Looks like a pipeline: each stage hands output to the next.

Layer 4 — Signatures:
  load_raw_data()  → list[dict]           Stage 1: raw source
  parse_students() → list[Student]        Stage 2: typed objects
  apply_curve()    → list[Student]        Stage 3: transform scores
  build_grade_report() → list[dict]       Stage 4: compute grades

Layer 5 — One path:
  Alice: raw dict {"name": "Alice", "scores": [...]}
     → Student(name="Alice", scores=[...])
     → curved Student(name="Alice", scores=[...+5])
     → {"name": "Alice", "average": 91.2, "grade": "A"}
```

> "In about 5 minutes, you understand the structure, the data model, and the execution flow. Now you're ready to debug something specific — because you know what *should* happen. Most debugging failures begin with not knowing what the code is supposed to do."

---

## 1.1 The Broken Program

**Run the program. Say nothing.**

```python
# buggy_grades.py — Run this. It crashes. Find the bugs.
from dataclasses import dataclass

@dataclass
class Student:
    name: str
    scores: list[int]

    def average(self) -> float:
        return sum(self.scores) / len(self.scores)

def calculate_letter_grade(avg: float) -> str:
    if avg > 90:          # is this right?
        return "A"
    elif avg >= 80:
        return "B"
    elif avg >= 70:
        return "C"
    elif avg >= 60:
        return "D"
    return "F"

def load_raw_data() -> list[dict]:
    """Stage 1: Source data"""
    return [
        {"name": "Alice",   "scores": [85, 92, 78, 90]},
        {"name": "Bob",     "scores": [72, 68, 75, 80]},
        {"name": "Charlie", "scores": [95, 88, 91, 97]},
        {"name": "Diana",   "scores": []},
        {"name": "Eve",     "scores": [75, 85, 85, 95]},
    ]

def parse_students(raw_data: list[dict]) -> list[Student]:
    """Stage 2: Create Student objects"""
    return [Student(**data) for data in raw_data]

def apply_curve(students: list[Student], curve: int) -> list[Student]:
    """Stage 3: Add curve points to every score"""
    curved = []
    for student in students:
        new_scores = [s + curve for s in student.scores]
        curved.append(Student(name=student.name, scores=new_scores))
    return curved

def build_grade_report(students: list[Student]) -> list[dict]:
    """Stage 4: Compute average and letter grade for each student"""
    report = []
    for student in students:
        avg = student.average()
        grade = calculate_letter_grade(avg)
        report.append({
            "name": student.name,
            "average": avg,
            "grade": grade,
        })
    return report

def get_class_statistics(report: list[dict]) -> dict:
    averages = [entry["average"] for entry in report]
    return {
        "class_mean": sum(averages) / len(averages),
        "highest": max(averages),
        "lowest": min(averages),
        "passing": len([a for a in averages if a >= 60]),
    }

def main():
    raw_data = load_raw_data()              # Stage 1
    students  = parse_students(raw_data)    # Stage 2
    curved    = apply_curve(students, 5)    # Stage 3
    report    = build_grade_report(curved)  # Stage 4
    stats     = get_class_statistics(report)

    print("=== GRADE REPORT ===")
    for entry in report:
        print(f"  {entry['name']}: {entry['grade']} ({entry['average']:.1f})")
    print(f"\nClass Mean: {stats['class_mean']:.1f}")
    print(f"Passing: {stats['passing']}/{len(report)}")

main()
```

> "Step 0 before debugging anything: can you reproduce the failure consistently? Run it three times. Does it crash every run? If it does, you have a reliable reproduction — you can confirm your fix works. If it only crashes sometimes, note what varies between runs. Always confirm reproducibility before hunting."

**Run it. It crashes.**

```
Traceback (most recent call last):
  File "buggy_grades.py", line 80, in <module>
    main()
  File "buggy_grades.py", line 71, in main
    report = build_grade_report(curved)
  File "buggy_grades.py", line 49, in build_grade_report
    avg = student.average()
  File "buggy_grades.py", line 10, in average
    return sum(self.scores) / len(self.scores)
ZeroDivisionError: division by zero
```

**Now ask the class:**

> "You have 60 seconds. Find the bug. Go."

Watch what happens. Most will stare at the code. Some will scroll up and down. A few will start adding `print()` calls. Almost nobody will **read the error message first**.

After 60 seconds, ask:

> "Who read the error message before touching the code? Raise your hand."

Usually one or two hands. That's the problem.

---

## 1.2 The Print Statement Trap

**Show what a typical beginner does:**

```python
# Attempt 1: Scatter print statements everywhere
def main():
    print("DEBUG 1: starting main")                              # Added
    raw_data = load_raw_data()
    students = parse_students(raw_data)
    print(f"DEBUG 2: created {len(students)} students")          # Added

    curved = apply_curve(students, 5)
    print(f"DEBUG 3: applied curve")                             # Added

    report = build_grade_report(curved)                          # Still crashes
    # ... never reaches here
```

Output:
```
DEBUG 1: starting main
DEBUG 2: created 5 students
DEBUG 3: applied curve
Traceback (most recent call last):
  ...
ZeroDivisionError: division by zero
```

"OK, it crashes inside `build_grade_report`. Let me add prints there..."

```python
# Attempt 2: More prints, deeper
def build_grade_report(students: list[Student]) -> list[dict]:
    report = []
    for student in students:
        print(f"DEBUG 4: processing {student.name}")             # Added
        avg = student.average()                                   # Crashes here
        ...
```

Output:
```
DEBUG 4: processing Alice
DEBUG 4: processing Bob
DEBUG 4: processing Charlie
DEBUG 4: processing Diana
Traceback (most recent call last):
  ...
ZeroDivisionError: division by zero
```

"It crashes on Diana. Let me add prints inside `average()`..."

```python
# Attempt 3: Even deeper
def average(self) -> float:
    print(f"DEBUG 5: scores = {self.scores}")                    # Added
    print(f"DEBUG 5: len = {len(self.scores)}")                  # Added
    return sum(self.scores) / len(self.scores)
```

Output:
```
DEBUG 5: scores = [90, 97, 83, 95]
DEBUG 5: scores = [77, 73, 80, 85]
DEBUG 5: scores = [100, 93, 96, 102]
DEBUG 5: scores = []
DEBUG 5: len = 0
Traceback (most recent call last):
  ...
ZeroDivisionError: division by zero
```

"Found it! Diana has empty scores!"

**Now count the cost:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE PRINT DEBUGGING COST                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it took:                                                  │
│  ├─ 5 print statements added                                    │
│  ├─ 3 re-runs of the program                                    │
│  ├─ ~3 minutes of editing and re-running                        │
│  ├─ You now have to DELETE all 5 debug prints                   │
│  └─ You WILL forget one. It WILL show up in production.         │
│                                                                 │
│  What the traceback already told you:                           │
│  ├─ File "buggy_grades.py", line 10, in average                │
│  ├─ return sum(self.scores) / len(self.scores)                  │
│  ├─ ZeroDivisionError: division by zero                         │
│  └─ len(self.scores) must be 0. Done. 5 seconds.               │
│                                                                 │
│  Print debugging: 3 minutes + cleanup risk                      │
│  Reading the traceback: 5 seconds                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Make this clear:**

> "I'm not saying print debugging is always wrong. Sometimes a quick `print(x)` is the fastest path. But it should be your scalpel, not your hammer. If you reach for print() as your FIRST instinct, you're working blind when the error message already has a flashlight pointed at the problem."

---

## 1.3 The Detective Analogy

**This analogy carries us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE DETECTIVE ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BAD DETECTIVE (Print Debugging)                                │
│  ───────────────────────────────                                │
│                                                                 │
│  1. Crime reported: "Someone broke something"                   │
│  2. Walk into EVERY room shouting "WHO DID IT?"                 │
│  3. Check every drawer, every closet, every corner              │
│  4. Eventually find the broken window                           │
│  5. Forget to put everything back when you leave                │
│                                                                 │
│                                                                 │
│  GOOD DETECTIVE (Systematic Debugging)                          │
│  ─────────────────────────────────────                          │
│                                                                 │
│  1. Crime reported: "Someone broke something"                   │
│  2. READ THE REPORT first (stack trace)                         │
│  3. Go directly to the crime scene (the failing line)           │
│  4. Examine the evidence (variable values)                      │
│  5. Form a hypothesis ("scores list was empty")                 │
│  6. Test the hypothesis (check where scores comes from)         │
│  7. Case closed                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The evidence-reading half of the analogy — which applies right now:**

```
Detective Work           │  Python Debugging
─────────────────────────│────────────────────────────────────
Crime report             │  Error message / wrong output
Witness testimony        │  Stack trace (who called whom)
Crime scene              │  The line where it crashed
Evidence at the scene    │  Variable values at crash point
```

> "The full analogy — including the tools — will complete itself as we introduce each debugging instrument in Part 3. For now, what we have is enough to read every piece of evidence the traceback gives us."

**The central insight:**

> "A bug is not a mystery. It's a gap between what you BELIEVE your code does and what it ACTUALLY does. The traceback, the debugger, the method — these are tools for seeing reality. The moment you see what your code actually does, the bug is obvious. It always is."

---

# PART 2: READING THE EVIDENCE

## The Execution Stack — Why Tracebacks Look the Way They Do

**Before reading a traceback, you need to understand what produces it.**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE EXECUTION STACK — A MENTAL MODEL                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every time Python calls a function, it pushes a new           │
│  EXECUTION FRAME onto a stack. Each frame holds:               │
│  - which function is running                                    │
│  - which line it's currently on                                 │
│  - the local variables in scope                                 │
│                                                                 │
│  When a function returns, its frame is POPPED off the stack.   │
│  This is LIFO — Last In, First Out.                             │
│                                                                 │
│                                                                 │
│  BUGGY_GRADES.PY — stack at the moment of the crash:           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  average()          ← TOP (most recently called)         │   │
│  │  "I am dividing by len(self.scores)"                     │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  build_grade_report()                                    │   │
│  │  "I called student.average() on line 49"                 │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  main()                                                  │   │
│  │  "I called build_grade_report(curved) on line 71"        │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  <module>           ← BOTTOM (oldest, first to run)      │   │
│  │  "I called main() on line 80"                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│                                                                 │
│  When the crash happens, Python cannot continue.                │
│  It prints out the entire stack — but NEWEST FRAME LAST.        │
│  Why? Because "most recent call last" literally means           │
│  "the last thing called is at the bottom of the printout."     │
│                                                                 │
│  The traceback is this stack, printed oldest-to-newest.         │
│  average() is at the BOTTOM of the printout because it was     │
│  the LAST function called — the top of the actual stack.        │
│                                                                 │
│  This is not a formatting choice. It is physics.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this model matters:**

When you encounter a traceback with 30 frames — from SQLAlchemy internals, from asyncio, from library code you didn't write — this model explains all of it. Every frame is a function call that was pushed onto the stack. The crash happened at the top of the stack (bottom of the printout). Everything between your entry point and the crash is the call chain.

> "Every weird traceback variant you encounter for the rest of this course — recursive calls showing the same frame many times, library code buried between your frames, async tasks creating separate frame chains — they are all instances of this one model. A stack. Pushed on call. Popped on return. Printed newest-last."

---

## 2.1 Anatomy of a Traceback

**The traceback is your crime scene report. Learn to read every line.**

Take our crash from Part 1:

```
Traceback (most recent call last):
  File "buggy_grades.py", line 80, in <module>
    main()
  File "buggy_grades.py", line 71, in main
    report = build_grade_report(curved)
  File "buggy_grades.py", line 49, in build_grade_report
    avg = student.average()
  File "buggy_grades.py", line 10, in average
    return sum(self.scores) / len(self.scores)
ZeroDivisionError: division by zero
```

**Every traceback has the same structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ANATOMY OF A TRACEBACK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Traceback (most recent call last):    ← HEADER                │
│  ─────────────────────────────────       Always the same.       │
│                                          "most recent call      │
│                                          last" = read bottom    │
│                                          to top for causality   │
│                                                                 │
│    File "grades.py", line 80, in <module>  ← FRAME 1 (oldest)  │
│      main()                                                     │
│    ─────────────────────────────────────                        │
│    │         │           │          │                           │
│    │         │           │          └─ The scope                │
│    │         │           │             <module> = top level     │
│    │         │           └─ Which line number                   │
│    │         └─ Which file                                      │
│    └─ The code on that line                                     │
│                                                                 │
│    File "grades.py", line 71, in main     ← FRAME 2            │
│      report = build_grade_report(curved)                        │
│                                                                 │
│    File "grades.py", line 49, in build_grade_report ← FRAME 3  │
│      avg = student.average()                                    │
│                                                                 │
│    File "grades.py", line 10, in average  ← FRAME 4 (newest)   │
│      return sum(self.scores) / len(self.scores)                 │
│    ───────────────────────────────────────                      │
│    THIS is where the crash happened.                            │
│    THIS is your crime scene.                                    │
│                                                                 │
│  ZeroDivisionError: division by zero      ← THE ERROR          │
│  ────────────────────────────────────       Two parts:          │
│  │                    │                     1. Error TYPE       │
│  │                    └─ Human-readable     2. Error MESSAGE    │
│  └─ The exception class    description                          │
│                                                                 │
│                                                                 │
│  ★ START READING HERE ★                                         │
│    The error + the bottom frame = 80% of your answer.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What each frame tells you:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE CALL CHAIN (Read bottom to top)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Frame 4 (bottom): WHERE it crashed                             │
│    average() tried to divide by len(self.scores)                │
│    len(self.scores) was 0 → ZeroDivisionError                   │
│                                                                 │
│  Frame 3: WHO called the crasher                                │
│    build_grade_report() called student.average()                │
│    This was inside a loop over students                         │
│                                                                 │
│  Frame 2: WHO called that                                       │
│    main() called build_grade_report(curved)                     │
│    So the problem student came from curved                      │
│                                                                 │
│  Frame 1: WHERE it all started                                  │
│    main() was called at the top level (<module>)                │
│                                                                 │
│                                                                 │
│  The chain:                                                     │
│  <module> → main() → build_grade_report() → average()           │
│                                               └─ 💥 CRASH       │
│                                                                 │
│  Think of it as a trail of footprints leading to the crime.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Recursive Tracebacks

**What do you do when a traceback is 500 lines long and every line looks the same?**

Not every crash produces a clean four-frame traceback. When a recursive function has no valid exit condition, Python prints every call frame — up to its recursion limit of 1,000. The result is a traceback that fills the entire terminal. Most students scroll to the top, see a wall of identical lines, and freeze.

Do not scroll. The reading technique is identical. The traceback is just noisier.

```python
# demo.py — recursive function with no base case
def factorial(n: int) -> int:
    return n * factorial(n - 1)   # Missing: if n == 0: return 1

factorial(5)
```

Running this produces:

```
Traceback (most recent call last):
  File "demo.py", line 4, in <module>
    factorial(5)
  File "demo.py", line 2, in factorial
    return n * factorial(n - 1)
  File "demo.py", line 2, in factorial
    return n * factorial(n - 1)
  File "demo.py", line 2, in factorial
    return n * factorial(n - 1)
  [Previous line repeated 996 more times]
RecursionError: maximum recursion depth exceeded
```

Python truncates the display — it shows the first few frames, writes `[Previous line repeated N more times]`, then shows the last line. You will never see all 1,000 frames. You do not need to.

```
┌─────────────────────────────────────────────────────────────────┐
│               READING RECURSIVE TRACEBACKS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Read the LAST LINE first                               │
│  ────────────────────────────────                               │
│  RecursionError: maximum recursion depth exceeded               │
│                                                                 │
│  RecursionError has exactly three causes:                       │
│  ├─ No base case — the function never has a stopping condition  │
│  ├─ Base case unreachable — it exists but the input bypasses it │
│  └─ Mutual recursion — A() calls B() calls A() indefinitely    │
│                                                                 │
│                                                                 │
│  STEP 2: Identify the REPEATING FRAME                           │
│  ──────────────────────────────────                             │
│  File "demo.py", line 2, in factorial                           │
│    return n * factorial(n - 1)   ← same line, hundreds of times │
│                                                                 │
│  This is the recursive call site. That function body is         │
│  missing its exit condition. This single pair of lines          │
│  contains the entire diagnosis.                                 │
│                                                                 │
│                                                                 │
│  STEP 3: SKIP the middle entirely                               │
│  ─────────────────────────────────                              │
│  The 996 repeated frames are noise. They say nothing the        │
│  first frame did not. Do not read them. The information         │
│  you need is: which function is repeating, and what is          │
│  the recursive call on that line?                               │
│                                                                 │
│                                                                 │
│  STEP 4: Ask "What would STOP this call?"                       │
│  ─────────────────────────────────────────                      │
│  Go to the function definition. Does it check for n == 0?       │
│  For n < 0? For any termination condition at all?               │
│  If no: that is the bug. Add the base case.                     │
│                                                                 │
│  Fix:                                                           │
│  def factorial(n: int) -> int:                                  │
│      if n == 0:          ← base case                            │
│          return 1                                               │
│      return n * factorial(n - 1)                                │
│                                                                 │
│  RULE: A long traceback with repeating frames = recursion.      │
│  Read the last line and the repeating pair. That is all.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Mutual recursion — when two functions call each other:**

```python
def is_even(n: int) -> bool:
    if n == 0:
        return True
    return is_odd(n - 1)

def is_odd(n: int) -> bool:
    if n == 0:
        return False
    return is_even(n - 1)   # Bug: negative input decrements forever

is_even(-3)   # -3 → is_odd(-4) → is_even(-5) → is_odd(-6) → ...
```

```
Traceback (most recent call last):
  File "demo.py", line 10, in <module>
    is_even(-3)
  File "demo.py", line 4, in is_even
    return is_odd(n - 1)
  File "demo.py", line 8, in is_odd
    return is_even(n - 1)
  File "demo.py", line 4, in is_even
    return is_odd(n - 1)
  [Previous 2 lines repeated 498 more times]
RecursionError: maximum recursion depth exceeded
```

The repeating pattern is now a **pair of frames** alternating. The technique is unchanged: find the repeating unit, ask what stops it, check whether input `-3` ever reaches `n == 0` by subtracting 1 each call. It cannot — it goes more negative forever.

Fix: add `if n < 0: return False` (or raise `ValueError`) as a guard before the recursive call.

> "The length of a recursive traceback gives you one piece of information: the function did not terminate before hitting Python's safety limit. Beyond that, you need exactly two things — the error type and the repeating frame. Everything else is repetition."

**One practical note — `sys.getrecursionlimit()`:**

```python
import sys
print(sys.getrecursionlimit())   # 1000 by default
```

If you hit this limit with correct logic on legitimately deep data (unusual in backend work), you can increase it with `sys.setrecursionlimit(N)`. If you're hitting it because of a bug: fix the bug. Never raise the limit to silence a RecursionError.

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 6 — Read a Recursive Traceback Cold
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  You receive this traceback from a tree-traversal function.
  You have NOT read the source code yet.

  Traceback (most recent call last):
    File "tree.py", line 14, in <module>
      print(find_depth(root))
    File "tree.py", line 7, in find_depth
      return 1 + max(find_depth(c) for c in node.children)
    File "tree.py", line 7, in find_depth
      return 1 + max(find_depth(c) for c in node.children)
    [Previous line repeated 994 more times]
  RecursionError: maximum recursion depth exceeded in comparison

  Answer these questions before looking at the source code:

  a) What does the error type tell you immediately?
  b) What is the repeating frame, and what does it imply about
     the data structure being traversed?
  c) When you open find_depth(), what two specific things do
     you check first?

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) RecursionError → find_depth() recurses without terminating.
     Either there is no base case, or the input tree contains
     a cycle (a child node that points back to an ancestor),
     making the traversal infinite.

  b) find_depth() calling itself on node.children at line 7.
     Every node visited has children — the traversal never
     reaches a leaf (a node with no children). In a valid tree,
     leaves exist and the recursion terminates there. The most
     likely cause: the node data contains a cycle, where a child
     references back to an ancestor, making traversal loop forever.

  c) Check these two things in order:
     1. Is there a base case for an empty children list?
        If node.children is empty, max() of an empty generator
        raises ValueError before RecursionError anyway — so
        the missing guard is: if not node.children: return 0
     2. Does the tree data contain cycles?
        A base case alone does not fix a cyclic graph.
        You would also need a `visited` set to detect already-seen
        nodes and stop re-entering them.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 2.2 Reading Bottom-Up (The Technique)

**Train yourself to read tracebacks in this exact order:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE BOTTOM-UP TECHNIQUE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Read the LAST LINE (the error)                         │
│  ─────────────────────────────────────                          │
│  ZeroDivisionError: division by zero                            │
│                                                                 │
│  Ask: "What TYPE of error?" → ZeroDivisionError                 │
│  Ask: "What does the message say?" → division by zero           │
│  Ask: "What causes this?" → dividing by 0 somewhere             │
│                                                                 │
│                                                                 │
│  STEP 2: Read the BOTTOM FRAME (the crash site)                 │
│  ──────────────────────────────────────                         │
│  File "grades.py", line 10, in average                          │
│    return sum(self.scores) / len(self.scores)                   │
│                                                                 │
│  Ask: "Where is the division?" → / len(self.scores)             │
│  Ask: "What could be 0?" → len(self.scores)                     │
│  Ask: "When is len() zero?" → when self.scores is empty         │
│                                                                 │
│  ✅ YOU ALREADY KNOW THE BUG.                                   │
│  A student has an empty scores list.                            │
│  The fix: guard against empty list in average().                │
│                                                                 │
│                                                                 │
│  STEP 3: Read UPWARD only if Step 2 isn't enough                │
│  ────────────────────────────────────────                       │
│  If you need to know HOW the code reached the crash,            │
│  read each frame upward to trace the call chain.                │
│                                                                 │
│  "Who called average()?" → build_grade_report()                 │
│  "Who called that?" → main()                                    │
│  "Where did students come from?" → apply_curve() → Stage 3      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Say this to the class:**

> "From now on, when your program crashes, your eyes go to the **last line** of the traceback. Not the first. Not the middle. The LAST LINE. That's the error. Then the frame right above it. That's where it happened. Two lines. That's 80% of debugging a crash."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 1 — Read a New Traceback Cold
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  You receive this traceback from a teammate's code:

  Traceback (most recent call last):
    File "report.py", line 12, in <module>
      generate_summary(data)
    File "report.py", line 8, in generate_summary
      total = sum(entry["score"] for entry in data)
    File "report.py", line 8, in <genexpr>
      total = sum(entry["score"] for entry in data)
  KeyError: 'score'

  Answer three questions WITHOUT looking at the rest of the code:

  a) What is the error type and what does it mean?
  b) What is the first line IN YOUR CODE to look at?
  c) What specific question do you ask about the data on that line?

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) KeyError — the dictionary 'entry' does not have a key named
     'score'. The key was not added, is misspelled, or was deleted.

  b) Line 8 in report.py — the bottom frame in YOUR code.
     (The <genexpr> frame is just Python's internal representation
     of a generator expression — it IS line 8, just annotated.)

  c) "What keys do the dicts in 'data' actually have?"
     If the dicts use 'scores' (plural) instead of 'score',
     that single character difference is the entire bug.
     Inspect one entry with p data[0] in the debugger.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 2.3 Common Error Types and What They Tell You

**Each error type is a different kind of clue:**

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMON ERROR TYPES — YOUR CHEAT SHEET              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TypeError                                                      │
│  ─────────                                                      │
│  "You used the wrong TYPE of thing"                             │
│  Cause: Passed str where int expected, wrong number of args,    │
│         called something that isn't callable                    │
│                                                                 │
│  NameError                                                      │
│  ─────────                                                      │
│  "That name doesn't EXIST here"                                 │
│  Cause: Misspelled variable name, used before defining,         │
│         forgot an import                                        │
│                                                                 │
│  AttributeError                                                 │
│  ──────────────                                                 │
│  "That object doesn't HAVE that attribute"                      │
│  Cause: Misspelled attribute/method, wrong object type,         │
│         object is None when you expected something else         │
│                                                                 │
│  KeyError                                                       │
│  ────────                                                       │
│  "That KEY doesn't exist in this dict"                          │
│  Cause: Wrong key name, key was never added, key was deleted    │
│                                                                 │
│  IndexError                                                     │
│  ──────────                                                     │
│  "That INDEX is out of range"                                   │
│  Cause: List is shorter than expected, off-by-one error,        │
│         empty list accessed with [0]                            │
│                                                                 │
│  ValueError                                                     │
│  ──────────                                                     │
│  "Right type, WRONG value"                                      │
│  Cause: int("hello"), list.remove(item_not_in_list),            │
│         invalid argument to a function                          │
│                                                                 │
│  ZeroDivisionError                                              │
│  ─────────────────                                              │
│  "You divided by zero"                                          │
│  Cause: Denominator is 0, empty sequence in average calc        │
│                                                                 │
│  ImportError / ModuleNotFoundError                              │
│  ────────────────────────────────                               │
│  "Can't find that module"                                       │
│  Cause: Not installed, misspelled, wrong virtual environment    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked examples for every type — learn the pattern, not just the name:**

**TypeError — wrong argument type:**

```python
def add_scores(scores: list[int], bonus: int) -> list[int]:
    return [s + bonus for s in scores]

result = add_scores([80, 90, 85], "5")  # Passed str, expected int
```

```
Traceback (most recent call last):
  File "demo.py", line 4, in <module>
    result = add_scores([80, 90, 85], "5")
  File "demo.py", line 2, in add_scores
    return [s + bonus for s in scores]
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

**Reading it:** Bottom frame tells you `int + str` failed. The message says exactly what types collided. Look at who called it — `"5"` is a string, should be `5`.

> "Notice something: if you had type hints AND ran mypy (Lecture 1), this bug would have been caught BEFORE you even ran the program. Type checking is debugging that happens before the crime."

---

**NameError — misspelled variable:**

```python
def calculate_bonus(score: float, factor: float) -> float:
    threshold = 90.0
    if scroe >= threshold:     # Misspelled: should be 'score'
        return score * factor
    return 0.0

calculate_bonus(95.0, 1.5)
```

```
Traceback (most recent call last):
  File "demo.py", line 6, in <module>
    calculate_bonus(95.0, 1.5)
  File "demo.py", line 3, in calculate_bonus
    if scroe >= threshold:
NameError: name 'scroe' is not defined. Did you mean: 'score'?
```

**Reading it:** `NameError: name 'scroe' is not defined` — Python 3.10+ even suggests the correction. The name in the error IS the clue. Look at the crash line, find that name, search for where it should have been defined.

> "NameError also fires when you forget an import: `import httpx` missing will produce `NameError: name 'httpx' is not defined`. If the name looks like a library, check your imports first."

---

**AttributeError — misspelled attribute:**

```python
@dataclass
class Config:
    host: str
    port: int
    debug: bool = False

def connect(config: Config) -> str:
    return f"Connecting to {config.hostname}:{config.port}"
    #                             ^^^^^^^^ should be .host
```

```
Traceback (most recent call last):
  File "demo.py", line 10, in <module>
    connect(Config(host="localhost", port=8080))
  File "demo.py", line 8, in connect
    return f"Connecting to {config.hostname}:{config.port}"
AttributeError: 'Config' object has no attribute 'hostname'
```

**Reading it:** The message tells you EXACTLY what's wrong — `Config` has no `hostname`. You wrote `hostname`, but the field is `host`. Also: `AttributeError: 'NoneType' object has no attribute 'X'` means you called `.X` on `None`. Look for a function that was supposed to return something but returned `None` instead.

> "Again — mypy catches this. But when it doesn't, the traceback tells you the object type (`Config`) and the attribute you tried to access (`hostname`). Compare those two facts against your class definition."

---

**KeyError — wrong dict key:**

```python
def process_student(raw: dict) -> str:
    return f"{raw['name']}: {raw['score']}"
    #                             ^^^^^ should be 'scores'

data = {"name": "Alice", "scores": [85, 92]}
process_student(data)
```

```
Traceback (most recent call last):
  File "demo.py", line 6, in <module>
    process_student(data)
  File "demo.py", line 2, in process_student
    return f"{raw['name']}: {raw['score']}"
KeyError: 'score'
```

**Reading it:** `KeyError: 'score'` — the key `'score'` doesn't exist. You meant `'scores'` (plural).

---

**IndexError — out of range:**

```python
def get_highest_score(scores: list[int]) -> int:
    return scores[len(scores)]   # Off-by-one: max valid index is len-1

get_highest_score([85, 92, 78])
```

```
Traceback (most recent call last):
  File "demo.py", line 4, in <module>
    get_highest_score([85, 92, 78])
  File "demo.py", line 2, in get_highest_score
    return scores[len(scores)]
IndexError: list index out of range
```

**Reading it:** `len(scores)` is 3, but valid indices are 0, 1, 2. `scores[3]` doesn't exist. Fix: `scores[-1]` for the last element, or `scores[len(scores) - 1]`. Classic off-by-one error.

---

**ValueError — right type, wrong value:**

```python
def parse_score(raw: str) -> int:
    return int(raw)

parse_score("not-a-number")
```

```
Traceback (most recent call last):
  File "demo.py", line 4, in <module>
    parse_score("not-a-number")
  File "demo.py", line 2, in parse_score
    return int(raw)
ValueError: invalid literal for int() with base 10: 'not-a-number'
```

**Reading it:** The right type was passed (a string), but the value can't be converted. The message tells you exactly what the bad value was. Add input validation or a try/except before the conversion.

---

**ZeroDivisionError — standalone example:**

```python
def class_average(total: float, count: int) -> float:
    return total / count

class_average(350.0, 0)   # count is 0
```

```
Traceback (most recent call last):
  File "demo.py", line 4, in <module>
    class_average(350.0, 0)
  File "demo.py", line 2, in class_average
    return total / count
ZeroDivisionError: float division by zero
```

**Reading it:** Look at the division on the crash line. Which value could be 0? Here: `count`. Guard with `if count == 0: return 0.0` or raise a descriptive error. You already saw this exact error type in Part 1 — now you have both the generic form and the specific instance in your memory.

---

**ImportError / ModuleNotFoundError — module not found:**

```python
import httpx   # httpx not installed in this environment

response = httpx.get("https://api.example.com/data")
```

```
Traceback (most recent call last):
  File "demo.py", line 1, in <module>
    import httpx
ModuleNotFoundError: No module named 'httpx'
```

**Reading it:** Python can't find this module. Three possible causes — diagnose in this order:

```
┌─────────────────────────────────────────────────────────────────┐
│            DIAGNOSING ModuleNotFoundError                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Not installed                                               │
│     → Run: uv add httpx                                         │
│     → Then re-run your program.                                 │
│                                                                 │
│  2. Wrong virtual environment                                   │
│     → Check: uv run python -c "import httpx; print('ok')"      │
│     → If this fails too, your uv environment doesn't have it.  │
│     → Ensure you're running with uv run, not bare python.       │
│                                                                 │
│  3. Module name is misspelled                                   │
│     → import HTTPx vs import httpx (case-sensitive)             │
│     → Check PyPI for the exact package name.                    │
│                                                                 │
│  This is the most common first-day blocker for new projects.    │
│  Every developer hits it. The diagnosis order above is fast.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 2 — Identify the Error Type
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  For each error message, state: (1) the error type, (2) the most
  likely specific cause, (3) the first thing you'd check.

  A) AttributeError: 'NoneType' object has no attribute 'name'

  B) ModuleNotFoundError: No module named 'pandas'

  C) IndexError: list index out of range
     (crash line: return scores[len(scores)])

  D) ValueError: invalid literal for int() with base 10: 'abc'

───────────────────────────────────────────────────────────────────
▼ SOLUTION
───────────────────────────────────────────────────────────────────

  A) AttributeError — a variable that should hold an object is
     None instead. Check the function that was supposed to return
     an object: does it have a missing return statement? A
     conditional path that falls through without returning?

  B) ModuleNotFoundError — pandas isn't installed in your current
     environment. Run: uv add pandas. If it still fails, check
     you're using uv run, not bare python.

  C) IndexError — off-by-one. len(scores) is 3 for a 3-element
     list, but valid indices are 0, 1, 2. scores[3] doesn't exist.
     Fix: scores[-1] or scores[len(scores) - 1].

  D) ValueError — int() received a string it cannot convert.
     'abc' is not a number. Check where the string comes from
     and add validation (try/except ValueError or a .isdigit()
     check) before attempting the conversion.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 2.4 Chained Exceptions (Connection to Lecture 2)

**Remember custom exception hierarchies from Lecture 2? They appear in tracebacks too.**

In Lecture 2, you learned to re-raise exceptions with `raise` or `raise e` inside an `except` block. Python also supports **explicit exception chaining**:

```python
raise NewException("message") from original_exception
```

The `from` clause tells Python that this new exception was directly *caused by* the original one. It is what produces the two-section traceback you are about to see. Without `from e`, the original cause is discarded and you lose a critical piece of evidence.

When you use `raise ... from ...`, Python shows BOTH exceptions:

```python
class DatabaseError(Exception):
    pass

class UserNotFoundError(DatabaseError):
    pass

def get_user_from_db(user_id: int) -> dict:
    users = {1: {"name": "Alice"}, 2: {"name": "Bob"}}
    try:
        return users[user_id]
    except KeyError as e:
        raise UserNotFoundError(f"User {user_id} not found") from e

get_user_from_db(99)
```

```
Traceback (most recent call last):
  File "demo.py", line 10, in get_user_from_db
    return users[user_id]
KeyError: 99

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "demo.py", line 14, in <module>
    get_user_from_db(99)
  File "demo.py", line 12, in get_user_from_db
    raise UserNotFoundError(f"User {user_id} not found") from e
UserNotFoundError: User 99 not found
```

**How to read a chained traceback:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CHAINED TRACEBACKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TWO SECTIONS:                                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │  FIRST TRACEBACK (the original error)        │               │
│  │  KeyError: 99                                │               │
│  │  └─ This is what ORIGINALLY went wrong       │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  "The above exception was the direct cause of..."               │
│  └─ This sentence links them. Read BOTH.                        │
│                                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │  SECOND TRACEBACK (the re-raised error)      │               │
│  │  UserNotFoundError: User 99 not found        │               │
│  │  └─ This is what YOUR code raised instead    │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  Reading order:                                                 │
│  1. Read the BOTTOM error first (UserNotFoundError)             │
│  2. If that's not enough, read the ORIGINAL above it            │
│  3. The original (KeyError: 99) tells you the root cause        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is why good exception hierarchies matter (Lecture 2). When you `raise ... from e`, you preserve the detective's trail. Without `from e`, the original clue is lost. Always chain your exceptions."

---

## 2.5 Async Tracebacks (Connection to Lecture 3)

**Async tracebacks look different. Don't panic.**

```python
import asyncio

async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(0.1)  # Simulate network
    raise ConnectionError(f"Failed to reach user service")

async def get_user_profile(user_id: int) -> dict:
    user = await fetch_user(user_id)
    return {"profile": user}

async def main():
    profile = await get_user_profile(42)
    print(profile)

asyncio.run(main())
```

```
Traceback (most recent call last):
  File "demo.py", line 14, in <module>
    asyncio.run(main())
  File "/usr/lib/python3.12/asyncio/runners.py", line 194, in run
    return runner.run(main)
  File "/usr/lib/python3.12/asyncio/runners.py", line 118, in run
    return self._loop.run_until_complete(task)
  File "/usr/lib/python3.12/asyncio/base_events.py", line 687, in run_until_complete
    return future.result()
  File "demo.py", line 12, in main
    profile = await get_user_profile(42)
  File "demo.py", line 8, in get_user_profile
    user = await fetch_user(user_id)
  File "demo.py", line 5, in fetch_user
    raise ConnectionError(f"Failed to reach user service")
ConnectionError: Failed to reach user service
```

**The difference: extra frames from asyncio internals.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASYNC TRACEBACKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  File "demo.py", line 14, in <module>                           │
│    asyncio.run(main())                          ← Your code     │
│                                                                 │
│  File ".../asyncio/runners.py", line 194 ─┐                     │
│  File ".../asyncio/runners.py", line 118  │─ asyncio internals  │
│  File ".../asyncio/base_events.py", ...  ─┘   IGNORE THESE     │
│                                                                 │
│  File "demo.py", line 12, in main                               │
│    profile = await get_user_profile(42)         ← Your code     │
│  File "demo.py", line 8, in get_user_profile                    │
│    user = await fetch_user(user_id)             ← Your code     │
│  File "demo.py", line 5, in fetch_user                          │
│    raise ConnectionError(...)                   ← The crash     │
│                                                                 │
│  RULE: Skip any frame that points to a file you didn't write.   │
│  Focus on YOUR files. The asyncio frames are just the event     │
│  loop plumbing — they're the "restaurant manager" (Lecture 3)   │
│  doing its job. The bug is in YOUR code.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Async tracebacks are noisier because the event loop inserts itself into the call chain. Your reading technique is the same: error at the bottom, then scan upward for lines in YOUR files. Skip everything from `asyncio/`, `concurrent/`, or any library internals."

**This rule generalizes — it applies to ALL library-heavy tracebacks, not just asyncio.**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHEN THE BOTTOM FRAME IS LIBRARY CODE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Week 6, SQLAlchemy errors arrive with 20+ internal frames   │
│  before reaching your code.                                     │
│                                                                 │
│  In Week 8, httpx errors may bury your call 15 frames deep.    │
│                                                                 │
│  In Week 11, Celery tracebacks may show 30+ framework frames   │
│  before the line you wrote.                                     │
│                                                                 │
│  THE RULE NEVER CHANGES:                                        │
│  ────────────────────────                                       │
│  1. Read the error type and message (last line).                │
│  2. Scan upward through the frames.                             │
│  3. Skip every frame whose file path is inside a library        │
│     you imported — anything in site-packages, anything with     │
│     a path like .../sqlalchemy/..., .../httpx/..., etc.         │
│  4. The FIRST frame in a file YOU wrote = your crime scene.     │
│                                                                 │
│  The library frames are footprints. Your frame is the body.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## When the Bug IS in the Library

The rule from Section 2.5 is: *skip library frames, find the first frame in your own files.* But sometimes every frame points to library internals and there is no frame in your code to find.

> "When there is no user frame visible, the call was made directly from your entry point — which means the library received your input and failed on it. That is API misuse 95% of the time. The framework below gets you to the answer in under 10 minutes."

```
┌─────────────────────────────────────────────────────────────────┐
│             WHEN EVERY FRAME IS LIBRARY CODE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Assume API misuse first                                │
│  ─────────────────────────────                                  │
│  Library bugs are rare. Misuse bugs are common.                 │
│  Read the official documentation for the exact function         │
│  shown in the bottom frame. Ask:                                │
│  "Am I passing the right type? The right argument order?        │
│   Am I using the async version when I need the sync one?"       │
│                                                                 │
│  STEP 2: Check the library version                              │
│  ─────────────────────────────────                              │
│  Run: uv pip list | grep <library-name>                         │
│  Ask: Does the version match the tutorial or Stack Overflow      │
│  answer you were following? A post from 2021 may describe        │
│  an API that changed in version 2.x. You may be on 3.x.        │
│                                                                 │
│  STEP 3: Build a minimal reproduction                           │
│  ─────────────────────────────────────                          │
│  Strip your code to the fewest lines that still crash.          │
│  If the crash disappears: your surrounding context is           │
│  passing bad state into the library.                            │
│  If it still crashes with 5 isolated lines: a library bug       │
│  is plausible. Proceed to Step 4.                               │
│                                                                 │
│  STEP 4: Search the library's issue tracker                     │
│  ─────────────────────────────────────────                      │
│  GitHub → Issues tab → paste the error type + message.          │
│  Look for:                                                      │
│  ├─ "Known in X.Y, fixed in X.Y+1" → uv add "lib>=X.Y+1"       │
│  ├─ "This is by design, use method Z instead" → update code    │
│  └─ "Workaround: wrap with..." → apply and leave a comment      │
│                                                                 │
│  STEP 5: Upgrade or work around                                 │
│  ─────────────────────────────────                              │
│  If it is a confirmed library bug:                              │
│  ├─ Upgrade if a fix exists.                                    │
│  ├─ Add a workaround with a code comment explaining WHY.        │
│  └─ Report the issue with your minimal reproduction.            │
│                                                                 │
│  ⚠  Complete Steps 1–3 before filing a bug report.             │
│     Most "library bugs" are API misuse found in Step 3.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked example — an SSL traceback with no user frames:**

```python
import httpx

# Crashes. The entire traceback is httpx and ssl internals.
response = httpx.get("https://internal.company.dev/api/data")
```

```
Traceback (most recent call last):
  File "client.py", line 4, in <module>
    response = httpx.get("https://internal.company.dev/api/data")
  File ".../httpx/_client.py", line 901, in get
    return self.request("GET", url, ...)
  File ".../httpcore/_sync/connection.py", line 97, in connect_tcp
    ...
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED]
  certificate verify failed: unable to get local issuer certificate
```

**Applying the framework:**

- **Step 1:** Using httpx correctly? `httpx.get(url)` is the documented API. Yes.
- **Step 2:** Version current. Not a version regression.
- **Step 3:** Three isolated lines still crash. The environment may be the issue.
- **Step 4:** Search httpx issues for `SSLCertVerificationError`. Result: internal or self-signed server certificates require `verify="path/to/cert.pem"` or `verify=False` (development only).
- **Conclusion:** Not a library bug. The development server has a self-signed certificate that the system does not trust. `httpx.get(url, verify=False)` for local dev only; never in production.

> "The entire traceback pointed to httpx and ssl. There was no user frame. The framework above revealed the true cause — an environment issue — without panic and without blaming the library."

---

# PART 3: THE TOOLS

## 3.1 The IDE Debugger (Your Forensic Lab)

**Your forensic lab has arrived.**

The IDE debugger is your forensic lab — equipped with surveillance cameras (breakpoints), instant replay (stepping through code), and a full evidence locker (variable inspection). We'll complete each piece of the detective analogy as we introduce each tool in the sections that follow.

**The IDE debugger is the single most powerful tool you'll learn today.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PRINT DEBUGGING           │  IDE DEBUGGER                      │
│  ─────────────────         │  ────────────                      │
│                            │                                    │
│  See one variable at a     │  See ALL variables at once         │
│  time, where you put the   │                                    │
│  print() statement         │  Pause anywhere, anytime           │
│                            │                                    │
│  Must re-run after each    │  Navigate forward through          │
│  change                    │  execution in real time            │
│                            │                                    │
│  Must manually remove      │  No code changes needed            │
│  debug prints after        │  Nothing to clean up               │
│                            │                                    │
│  Can't inspect complex     │  Expand nested objects, dicts,     │
│  objects easily             │  lists — click to explore          │
│                            │                                    │
│  Flashlight in a dark room │  Full lights, security cameras,    │
│                            │  and instant replay                │
│                            │                                    │
└─────────────────────────────────────────────────────────────────┘
```

> "If you leave this lecture using only one thing, let it be the IDE debugger. It will save you more hours than any other tool in your career."

**This works in both VS Code and PyCharm. The concepts are identical, the buttons look slightly different.**

---

## 3.2 Breakpoints (Pausing Execution)

**A breakpoint says: "Stop here. Let me look around." — Your first surveillance camera.**

To set a breakpoint: **click in the gutter** (left margin next to line numbers). A red dot appears.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SETTING A BREAKPOINT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EDITOR                                                         │
│  ──────────────────────────────────────────────────             │
│     46│  def build_grade_report(students: list[Student]):       │
│     47│      report = []                                        │
│     48│      for student in students:                           │
│  🔴 49│          avg = student.average()                        │
│     50│          grade = calculate_letter_grade(avg)            │
│     51│          report.append({...})                           │
│     52│      return report                                      │
│      ▲                                                          │
│      │                                                          │
│      └─ Breakpoint on line 49 — the line that CALLS average()  │
│         One line before the crash, inside the loop.            │
│         This lets us inspect the student BEFORE entering        │
│         average(), so we can see which student triggers it.     │
│                                                                 │
│  When you RUN with the debugger (F5, not regular run):          │
│  The program starts normally, then PAUSES at line 49.           │
│  You can now inspect everything.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Where to set your FIRST breakpoint:**

> "The traceback told us the crash is in `average()` at line 10. But we don't just want to see the crash — we want to see the moment BEFORE the crash. Set the breakpoint at the line that CALLS `average()`, which is line 49 in `build_grade_report`. That way we can inspect WHICH student triggers the problem before entering the function."

---

## 3.3 Stepping Through Code

**Once paused at a breakpoint, you have three movement controls. Crucially, you do not pick one at random — you pick based on your active hypothesis.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    STEPPING CONTROLS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  STEP OVER (F10)                                                │
│  ───────────────                                                │
│  Execute this line completely. Move to the NEXT line.           │
│  If this line calls a function, run the ENTIRE function         │
│  and come back — don't go inside it.                            │
│                                                                 │
│  Like watching a suspect enter and exit a building              │
│  from across the street. You see the result, not the details.   │
│                                                                 │
│      def main():                                                │
│       →  report = build_grade_report(curved)   [STEP OVER]     │
│          stats = get_class_statistics(report)  ← land here     │
│                                                                 │
│          build_grade_report runs entirely, you get the result,  │
│          but you never went inside it.                          │
│                                                                 │
│                                                                 │
│  STEP INTO (F11)                                                │
│  ────────────────                                               │
│  If this line calls a function, GO INSIDE that function.        │
│  Now you're on the first line of that function.                 │
│                                                                 │
│  Like following the suspect through the door.                   │
│                                                                 │
│      def main():                                                │
│       →  report = build_grade_report(curved)   [STEP INTO]     │
│                                                                 │
│      def build_grade_report(students):                          │
│       →  report = []                           ← land here     │
│                                                                 │
│                                                                 │
│  STEP OUT (Shift+F11)                                           │
│  ────────────────────                                           │
│  Run the REST of the current function. Return to the caller.    │
│                                                                 │
│  Like leaving the building and going back to the street.        │
│                                                                 │
│      def build_grade_report(students):                          │
│          report = []                                            │
│       →  for student in students:              [STEP OUT]       │
│              ...                                                │
│                                                                 │
│      def main():                                                │
│          report = build_grade_report(curved)                    │
│       →  stats = get_class_statistics(report)  ← land here     │
│                                                                 │
│                                                                 │
│  CONTINUE (F5)                                                  │
│  ─────────────                                                  │
│  Resume normal execution until the next breakpoint              │
│  (or the program ends / crashes).                               │
│                                                                 │
│  Like saying "I've seen enough here, take me to the             │
│  next surveillance point."                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The decision framework — how to choose between them:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHICH COMMAND DO I PRESS?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You are paused at a line that calls a function.                │
│  Ask: "Where does my hypothesis say the bug is?"                │
│                                                                 │
│                                                                 │
│  "The bug is INSIDE that function"                              │
│  → STEP INTO (F11)                                              │
│  → Follow the suspect through the door.                         │
│                                                                 │
│                                                                 │
│  "That function is probably fine, I need its result"            │
│  → STEP OVER (F10)                                              │
│  → Watch the suspect enter and exit. You only need the result.  │
│                                                                 │
│                                                                 │
│  "I've confirmed this function is NOT the problem"              │
│  → STEP OUT (Shift+F11)                                         │
│  → Leave the building. Return to the caller.                    │
│                                                                 │
│                                                                 │
│  ⚠  NEVER: Step Into blindly on every line.                    │
│  That descends into Python stdlib internals.                    │
│  You will get lost inside list comprehensions and               │
│  built-in functions and never find your bug.                    │
│  Your hypothesis decides your direction.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Walk through debugging buggy_grades.py step by step — with decision reasoning at each step:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEBUGGING buggy_grades.py — WALKTHROUGH            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Set breakpoint on line 49: avg = student.average()          │
│  2. Press F5 (Start Debugging)                                  │
│  3. Program runs, PAUSES at line 49                             │
│                                                                 │
│  ITERATION 1 (Alice):                                           │
│  ─────────────────────                                          │
│    Paused at line 49. Variables pane:                           │
│    student = Student(name='Alice', scores=[90, 97, 83, 95])    │
│    Hypothesis: scores could be empty. Check: 4 items. Safe.    │
│    Decision: STEP OVER (average() is fine for Alice)            │
│    Press F5 → goes to next iteration                            │
│                                                                 │
│  ITERATION 2 (Bob):                                             │
│  ──────────────────                                             │
│    student = Student(name='Bob', scores=[77, 73, 80, 85])      │
│    4 items. Safe. Decision: STEP OVER. Press F5.               │
│                                                                 │
│  ITERATION 3 (Charlie):                                         │
│  ───────────────────────                                        │
│    student = Student(name='Charlie', scores=[100, 93, 96, 102])│
│    4 items. Safe. Decision: STEP OVER. Press F5.               │
│                                                                 │
│  ITERATION 4 (Diana):                                           │
│  ─────────────────────                                          │
│    student = Student(name='Diana', scores=[])                   │
│    ❌ scores is EMPTY!                                          │
│    Hypothesis confirmed: empty scores is the cause.             │
│    Decision: STEP INTO (F11) → verify inside average()          │
│    You see: len(self.scores) = 0. Division by 0 imminent.      │
│    ─────────────────────────────────────────────────           │
│    BUG FOUND. Diana has empty scores.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice we found this WITHOUT adding a single print statement. We didn't change the code at all. The debugger showed us the exact state of every variable at the exact moment that mattered."

**Now write the fix and verify it — finding a bug is only half the job:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FIX AND VERIFICATION LOOP                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Write the fix                                          │
│  ──────────────────────                                         │
│  def average(self) -> float:                                    │
│      if not self.scores:                                        │
│          return 0.0   # Guard: empty list returns 0.0           │
│      return sum(self.scores) / len(self.scores)                 │
│                                                                 │
│  Note: returning 0.0 is one choice. Raising ValueError          │
│  ("Cannot average empty scores") is another — it depends on     │
│  whether 0.0 is meaningful in your domain. For now, 0.0.        │
│                                                                 │
│  STEP 2: Run with the debugger again (F5)                       │
│  ─────────────────────────────────────────                      │
│  The breakpoint on line 49 is still there.                      │
│  Step through Alice, Bob, Charlie — still get correct averages  │
│  Step through Diana — scores is [], returns 0.0. No crash.      │
│  Step through Eve — scores has 4 items. Fine.                   │
│                                                                 │
│  STEP 3: Confirm the full output is reasonable                  │
│  ─────────────────────────────────────────────                  │
│  Press F5 to run to completion:                                 │
│                                                                 │
│  === GRADE REPORT ===                                           │
│    Alice:   A (91.2)                                            │
│    Bob:     C (78.8)                                            │
│    Charlie: A (97.8)                                            │
│    Diana:   F (0.0)    ← no crash; grade is F (0.0 < 60)        │
│    Eve:     B (90.0)   ← wait. 90.0 is the minimum for A...     │
│                                                                 │
│  STEP 4: Note what is still wrong                               │
│  ────────────────────────────────                               │
│  Diana's crash is fixed. But Eve's grade looks wrong.           │
│  A score of 90.0 should earn an A, not a B.                     │
│  This is a separate bug. We investigate it in Part 4.           │
│                                                                 │
│  RULE: Finding a fix is only half the job.                      │
│  Confirming the fix works, confirming it doesn't break          │
│  anything else, and checking the full output for new            │
│  anomalies — that is the other half.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 3 — Choose Your Stepping Command
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  You are paused at this line in the debugger:

      grade = calculate_letter_grade(avg)

  avg = 90.0. Eve is the current student.
  Your hypothesis: "The letter grade function must be handling
  the boundary condition at 90 incorrectly — using strict > 
  instead of >= ."

  Which command do you press, and why?

  A) Step Over (F10)
  B) Step Into (F11)
  C) Step Out (Shift+F11)
  D) Continue (F5)

───────────────────────────────────────────────────────────────────
▼ SOLUTION
───────────────────────────────────────────────────────────────────

  B) Step Into (F11).

  Your hypothesis says the bug is INSIDE calculate_letter_grade().
  To test it, you must follow the execution into that function and
  watch which branch the if/elif chain takes when avg = 90.0.

  If you Step Over (F10), the function runs as a black box —
  you get the result (B) but you miss the moment the wrong
  comparison is made. You need to SEE which branch is taken.

  Decision rule: Step Into when your hypothesis says the bug
  lives inside the function you are about to call.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 3.4 Variable Inspection and Watch Expressions

**The Variables pane shows you everything in the current scope:**

```
┌─────────────────────────────────────────────────────────────────┐
│  VS CODE DEBUGGER — PAUSED AT LINE 49 (Diana's iteration)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌── VARIABLES ──────────┐  ┌── EDITOR ─────────────────────┐  │
│  │                        │  │                               │  │
│  │  Locals:               │  │  48│  for student in stude..: │  │
│  │  ▼ student: Student    │  │→ 49│      avg = student.av..  │  │
│  │      name: "Diana"     │  │  50│      grade = calcula..   │  │
│  │    ▼ scores: list[0]   │  │  51│      report.append({..   │  │
│  │        (empty)         │  │                               │  │
│  │  ▼ report: list[3]     │  │                               │  │
│  │    ▼ 0: dict           │  │                               │  │
│  │        name: "Alice"   │  │                               │  │
│  │        average: 91.2   │  │                               │  │
│  │        grade: "A"      │  │                               │  │
│  │    ▼ 1: dict           │  │                               │  │
│  │        name: "Bob"     │  │                               │  │
│  │        ...             │  │                               │  │
│  │                        │  │                               │  │
│  │                        │  │                               │  │
│  ├── WATCH ──────────────┤  │                               │  │
│  │                        │  │                               │  │
│  │  len(student.scores)   │  │                               │  │
│  │    = 0                 │  │                               │  │
│  │  student.name          │  │                               │  │
│  │    = "Diana"           │  │                               │  │
│  │                        │  │                               │  │
│  ├── CALL STACK ─────────┤  │                               │  │
│  │                        │  │                               │  │
│  │  build_grade_report    │  │                               │  │
│  │  main                  │  │                               │  │
│  │  <module>              │  │                               │  │
│  │                        │  │                               │  │
│  └────────────────────────┘  └───────────────────────────────┘  │
│                                                                 │
│  DEBUG TOOLBAR:  ▶Continue  ⤵Over  ⤓Into  ⤒Out  ↻Restart  ■Stop│
│                    F5        F10    F11   ⇧F11                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The three panes you care about:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE THREE PANES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VARIABLES PANE                                                 │
│  ──────────────                                                 │
│  Shows ALL variables in the current scope automatically.        │
│  Click the ▼ arrow to expand objects, lists, dicts.             │
│  Updates live as you step through code.                         │
│                                                                 │
│  WATCH PANE                                                     │
│  ──────────                                                     │
│  Add ANY expression. It evaluates live as you step.             │
│  Examples:                                                      │
│    len(student.scores)                                          │
│    student.scores == []                                         │
│    report[-1]["grade"]                                          │
│    type(student).__name__                                       │
│                                                                 │
│  This is like adding print() statements that update             │
│  automatically and never pollute your code.                     │
│                                                                 │
│  CALL STACK PANE                                                │
│  ───────────────                                                │
│  Shows the same chain as the traceback — but LIVE.              │
│  Click any frame to jump to that line in the editor AND see     │
│  that frame's local variables in the Variables pane.            │
│  Want to see what main() passed as an argument?                 │
│  Click the "main" frame. Its locals appear instantly.           │
│  Use Ctrl+Click (Windows/Linux) or Cmd+Click (macOS) on any    │
│  function name in the editor to jump to its definition.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key point — dataclass `__repr__` helps you here:**

> "Remember from Lecture 2 that dataclasses auto-generate `__repr__`? This is where it pays off. In the Variables pane, the debugger shows `Student(name='Diana', scores=[])` instead of the useless `<Student object at 0x7f...>` you'd get from a regular class. If you define your own classes for this course, use `@dataclass` — your future debugging self will thank you."

---

## 3.5 Conditional Breakpoints

**Regular breakpoints pause EVERY time. Conditional breakpoints pause only when a condition is true. — Your targeted surveillance camera.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CONDITIONAL BREAKPOINTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem: Our loop runs 5 times (Alice, Bob, Charlie, Diana,    │
│  Eve). The bug is only in Diana. Do we really want to press F5  │
│  four times to skip the other students?                         │
│                                                                 │
│  Solution: RIGHT-CLICK the breakpoint → "Edit Breakpoint"       │
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  Breakpoint condition:                                │      │
│  │  ┌─────────────────────────────────────────────────┐  │      │
│  │  │ student.name == "Diana"                         │  │      │
│  │  └─────────────────────────────────────────────────┘  │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  Now the debugger ONLY pauses when student.name is "Diana".     │
│  Skip straight to the problem. No clicking through good data.   │
│                                                                 │
│                                                                 │
│  USEFUL CONDITIONS:                                             │
│  ──────────────────                                             │
│  student.name == "Diana"          ← Specific record             │
│  len(student.scores) == 0         ← Empty data condition        │
│  i > 100                          ← Deep in a long loop         │
│  avg < 0                          ← Invalid calculated value    │
│  isinstance(result, type(None))   ← Caught an unexpected None   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Conditional breakpoints are like telling a surveillance camera: 'Only record if the suspect is wearing a red hat.' You skip the noise and go straight to the evidence. Conditional breakpoints cover 90% of real debugging scenarios."

VS Code also supports two variants you will encounter as specific needs arise: **hit count breakpoints** (pause only on the Nth execution — useful when a bug only appears after hundreds of loop iterations) and **logpoints** (print a message to the Debug Console without pausing the program — print debugging without modifying your code). Both are in the supplementary reference card. Learn them when you need them; conditional breakpoints are sufficient for now.

---

## 3.6 Debug Configurations

**To debug, you need to tell the IDE WHAT to run and HOW. Here is why that is necessary.**

When you run `uv run buggy_grades.py` in the terminal, uv knows which Python interpreter to use, which virtual environment holds your dependencies, and which file is the entry point. When you press F5, VS Code knows none of this — it needs explicit instructions. The `launch.json` provides those instructions: which debug adapter to use, which file is the entry point, and where to show output. Without it, VS Code cannot start your program. With it, pressing F5 is equivalent to `uv run buggy_grades.py` — but with the debugger attached.

**VS Code: `launch.json`**

In VS Code, press `Ctrl+Shift+D` (Debug pane), then "create a launch.json file":

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Current File",
            "type": "debugpy",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal"
        }
    ]
}
```

**What each field means:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    launch.json EXPLAINED                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "name":    What you see in the debug dropdown.                 │
│             Name it something meaningful.                       │
│                                                                 │
│  "type":    The debugger engine. "debugpy" for Python.          │
│             debugpy is VS Code's Python debug adapter —         │
│             the component that bridges VS Code's debugger       │
│             interface to your running Python process. It was    │
│             installed automatically as part of the Python       │
│             extension during course onboarding.                 │
│                                                                 │
│  "request": "launch" = start a new process                      │
│             "attach" = connect to running process (advanced)     │
│                                                                 │
│  "program": What file to run.                                   │
│             ${file} = whatever file is currently open.          │
│             ⚠ Caution: ${file} runs whatever is open.           │
│             If you're editing a helper module at the moment      │
│             you press F5, VS Code runs THAT file as a script.   │
│             For projects with a clear entry point, use:         │
│             "program": "main.py"   ← always runs the right file │
│                                                                 │
│  "console": Where to show output.                               │
│             "integratedTerminal" = VS Code's terminal           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PyCharm: Built-in. Right-click any file → "Debug 'filename'".**

PyCharm auto-generates run configurations. The same concepts apply — breakpoints, stepping, variable inspection — the buttons are just in different places.

**Preview — configurations you'll use later this course:**

```json
// You'll add these as we progress through the course:

// For pytest (this week — Lecture 2):
{
    "name": "Debug Tests",
    "type": "debugpy",
    "request": "launch",
    "module": "pytest",
    "args": ["-xvs"],
    "console": "integratedTerminal"
}

// For FastAPI (Week 3 — when we get there):
{
    "name": "Debug FastAPI",
    "type": "debugpy",
    "request": "launch",
    "module": "uvicorn",
    "args": ["app.main:app", "--reload"],
    "console": "integratedTerminal"
}
```

> "Don't memorize these now. The important thing is knowing they exist. When we start testing next lecture and building APIs in Week 3, you'll configure the debugger for those specific tools. The debugging SKILLS are the same — breakpoints, stepping, inspecting. Only the launch configuration changes."

---

## 3.7 pdb and breakpoint() (The Universal Fallback)

**The IDE debugger is your primary tool. But sometimes you can't use it.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHEN YOU NEED pdb                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ IDE debugger works when:                                    │
│  ├─ You're on your own machine                                  │
│  ├─ You have VS Code / PyCharm open                             │
│  └─ You're running code locally                                 │
│                                                                 │
│  ❌ IDE debugger NOT available when:                            │
│  ├─ SSH into a remote server (no GUI)                           │
│  ├─ Inside a Docker container (Week 15)                         │
│  ├─ In a CI/CD pipeline (Week 15)                               │
│  ├─ On a colleague's machine via screen share                   │
│  └─ Quick check where launching the IDE is overkill             │
│                                                                 │
│  pdb works EVERYWHERE. It runs in the terminal.                 │
│  It's Python's built-in debugger. No install needed.            │
│  It's your flashlight when the full forensic lab isn't around.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The modern way to invoke pdb: `breakpoint()`**

```python
def build_grade_report(students: list[Student]) -> list[dict]:
    report = []
    for student in students:
        breakpoint()  # ← Execution PAUSES here. Terminal takes over.
        avg = student.average()
        grade = calculate_letter_grade(avg)
        report.append({"name": student.name, "average": avg, "grade": grade})
    return report
```

**When you run this, the terminal transforms:**

```
$ python buggy_grades.py
> /home/user/buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) 
```

**You're now in pdb. The 6 commands you need to know today:**

```
┌─────────────────────────────────────────────────────────────────┐
│                pdb — 6 ESSENTIAL COMMANDS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  p <expr>    Print the value of an expression                   │
│              p student.name  →  'Diana'                         │
│              p student.scores  →  []                            │
│              p len(student.scores)  →  0                        │
│              IDE equivalent: Variables / Watch pane             │
│                                                                 │
│  n (next)    Step OVER — execute line, move to next             │
│              IDE equivalent: F10                                │
│                                                                 │
│  s (step)    Step INTO — enter the function on this line        │
│              IDE equivalent: F11                                │
│                                                                 │
│  c (continue) Resume execution until next breakpoint            │
│              IDE equivalent: F5                                 │
│                                                                 │
│  l (list)    Show source code around the current line           │
│              Useful when you've lost your place                 │
│                                                                 │
│  q (quit)    Exit the debugger and terminate the program        │
│                                                                 │
│                                                                 │
│  Additional commands (pp, whatis, locals, r, w, u, d) are      │
│  in the supplementary pdb Reference Card. Learn them when a     │
│  specific need arises. These 6 handle 95% of situations.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Live pdb session — debugging our buggy program:**

```
$ python buggy_grades.py
> /home/user/buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) p student
Student(name='Alice', scores=[90, 97, 83, 95])
(Pdb) c
> /home/user/buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) p student.name
'Bob'
(Pdb) c
> /home/user/buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) p student.name
'Charlie'
(Pdb) c
> /home/user/buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) p student
Student(name='Diana', scores=[])
(Pdb) p len(student.scores)
0
(Pdb) p student.scores
[]
```

**Found it. Same result as the IDE debugger, just in text.**

> "Notice how the dataclass `__repr__` immediately shows us `scores=[]`. If `Student` were a plain class without `__repr__`, pdb would show `<Student object at 0x7f...>` — useless. This is why we learn dataclasses before debugging. The tools reinforce each other."

**CRITICAL: Remove `breakpoint()` before committing.**

```python
# Don't be this person in a code review:
def build_grade_report(students: list[Student]) -> list[dict]:
    report = []
    for student in students:
        breakpoint()  # ← OOPS. Shipped to production. 🔥
        avg = student.average()
```

> "Unlike print statements, `breakpoint()` will FREEZE your program in production — every request to a live API hits your breakpoint and hangs. In Week 15, we'll configure ruff to detect leftover `breakpoint()` calls. This requires explicitly enabling the debugger-detection rules in your `pyproject.toml`, which we'll do as part of that lecture's linting setup. For now, search your files manually for the string `breakpoint` before every git commit."

---

## Post-Mortem Debugging (pdb.pm())

**`breakpoint()` pauses execution before a crash. `pdb.pm()` enters the debugger after one has already occurred.**

The interactive debugger and `breakpoint()` both require you to anticipate where a bug is before running the program. But sometimes a program crashes unexpectedly and you need to inspect the exact state at the moment of failure — without re-running and guessing where to set a breakpoint.

`pdb.pm()` does exactly this. `pm` stands for **post-mortem**. It re-enters the interpreter at the frame where the most recent unhandled exception occurred, with all local variables intact.

```
┌─────────────────────────────────────────────────────────────────┐
│                   THREE FORMS OF POST-MORTEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FORM 1: Run the whole script under pdb (most general)          │
│  ──────────────────────────────────────────────────             │
│  $ python -m pdb buggy_grades.py                                │
│  (Pdb) c           ← continue; run the program normally         │
│  ... program crashes ...                                        │
│  > buggy_grades.py(10)average()                                 │
│  -> return sum(self.scores) / len(self.scores)                  │
│  (Pdb) _           ← you are now at the crash frame             │
│                                                                 │
│  -m pdb instruments the session automatically. When the         │
│  program crashes, pdb drops into post-mortem mode at the        │
│  crash frame. No code changes needed.                           │
│                                                                 │
│  FORM 2: Wrap the call in try/except (surgical)                 │
│  ─────────────────────────────────────────────                  │
│  import pdb                                                     │
│  try:                                                           │
│      main()                                                     │
│  except Exception:                                              │
│      pdb.post_mortem()   ← enter at crash frame                 │
│                                                                 │
│  Use when you know which function is likely to crash            │
│  and want to intercept it specifically.                         │
│                                                                 │
│  FORM 3: After a crash in an interactive REPL session           │
│  ─────────────────────────────────────────────────              │
│  >>> main()                                                     │
│  Traceback: ... ZeroDivisionError                               │
│  >>> import pdb; pdb.pm()    ← re-enters the crash frame        │
│                                                                 │
│  Python sets sys.last_traceback after every unhandled           │
│  exception. pdb.pm() reads it. Works only if the exception      │
│  just occurred in the same REPL session.                        │
│                                                                 │
│                                                                 │
│  WHEN TO USE POST-MORTEM:                                       │
│  ├─ A crash is unexpected and you have no hypothesis yet        │
│  ├─ You can reproduce the crash but don't know where            │
│     to put a breakpoint                                         │
│  └─ You want to explore the crash environment interactively     │
│     before writing any fix                                      │
│                                                                 │
│  All six core pdb commands from Section 3.7 work here:          │
│  p, n, s, c, l, q — plus u / d to navigate up and down frames. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Live post-mortem session on buggy_grades.py:**

```
$ python -m pdb buggy_grades.py
> buggy_grades.py(1)<module>()
-> from dataclasses import dataclass
(Pdb) c
Traceback (most recent call last):
  ...
ZeroDivisionError: division by zero
Uncaught exception. Entering post mortem debugging
Running 'cont' or 'step' will restart the program
> buggy_grades.py(10)average()
-> return sum(self.scores) / len(self.scores)
(Pdb) p self
Student(name='Diana', scores=[])
(Pdb) p self.scores
[]
(Pdb) l
  7         def average(self) -> float:
  8  ->         return sum(self.scores) / len(self.scores)
(Pdb) u
> buggy_grades.py(49)build_grade_report()
-> avg = student.average()
(Pdb) p student.name
'Diana'
(Pdb) q
```

> "Notice: we changed zero lines of code. We ran the program, let it crash, and then inspected the exact state at the crash point. The `u` command moved us up one frame — from `average()` to `build_grade_report()` — so we could confirm which student triggered it. This is post-mortem in miniature."

---

## Attaching to a Running Process

**Sometimes you cannot restart the program. The debugger must join a process that is already running.**

Both the IDE debugger and `breakpoint()` require launching the process from scratch under debug control. If your program is already running — a server, a long-running worker, a process launched by a framework — you need **attach mode**.

`debugpy` supports this. The process must opt in by calling `debugpy.listen()` before you connect:

```python
# Add at the very top of your entry point, before any other code:
import debugpy

debugpy.listen(5678)          # Open a debug port
debugpy.wait_for_client()     # Pause here until VS Code connects
# ↑ Remove both lines when done. Like breakpoint(), these must
#   never ship to production. An open debug port is a remote
#   code execution vulnerability.

main()   # ... rest of your program
```

Add this configuration to `.vscode/launch.json`:

```json
{
    "name": "Attach to Running Process",
    "type": "debugpy",
    "request": "attach",
    "connect": {
        "host": "localhost",
        "port": 5678
    }
}
```

Start your program. It pauses at `wait_for_client()`. Press F5 in VS Code with "Attach to Running Process" selected. The full debugger attaches — breakpoints, stepping, variable inspection — on a process launched by something else entirely.

```
┌─────────────────────────────────────────────────────────────────┐
│                  LAUNCH vs ATTACH — THE CHOICE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "launch" (Section 3.6)                                         │
│  VS Code starts the program itself.                             │
│  Use for: normal local development.                             │
│                                                                 │
│  "attach"                                                       │
│  VS Code joins a process that something else started.           │
│  Use when:                                                      │
│  ├─ Debugging inside a Docker container (Week 15)               │
│  ├─ Debugging a process managed by a daemon or supervisor       │
│  ├─ Debugging a long-running worker that took minutes           │
│     to reach the state you need to inspect                      │
│  └─ Any scenario where re-launching would lose critical state   │
│                                                                 │
│  ⚠  debugpy.listen() in production = remote code execution.    │
│     Use only on localhost during development, or inside a       │
│     private Docker network. Week 15 covers the safe pattern     │
│     for attaching inside containers.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Attach mode is preview-level for this course — you won't need it for the mini-project. The mental model is: `launch` means VS Code is the launcher; `attach` means something else launched it and VS Code joins. When Docker arrives in Week 15, this distinction will already be familiar."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 7 — Post-Mortem Debugging
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  A colleague hands you a script that crashes. You do not know
  where the crash occurs, so you cannot place a breakpoint.

  a) Write the exact terminal command that lets you run the
     script AND drop into pdb at the crash frame automatically,
     without modifying a single line of source code.

  b) Once you are at the pdb prompt inside the crash frame,
     which two commands do you run first, and why?

  c) At the pdb prompt, you realise the crash happened inside
     a helper function and you want to see the state of the
     caller. Which command do you use, and what does it do?

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) python -m pdb script.py
     Running with -m pdb instruments the entire session.
     When the program crashes with an unhandled exception,
     pdb enters post-mortem mode at the crash frame automatically.
     No try/except wrapping or source changes required.

  b) Two commands, in this order:
     l  (list) — shows the source code around the current line.
        You just landed at an unknown crash frame; you need
        to orient yourself before inspecting values.
     p <variable> — inspect the value that caused the crash.
        The traceback named the line; the variables pane
        tells you what was in scope when it failed.

  c) u (up)
     Moves the active frame one level up in the call stack —
     from the crashed function to the function that called it.
     Local variables in the Variables pane update to reflect
     the caller's scope. Use d (down) to return.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 3.8 Logging — Debugging Without Stopping the Program

**The IDE debugger and pdb are powerful. But they share one critical limitation: they require interactive access to the running program. Logging is what you use when you have neither.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREE TOOLS — THREE ROLES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRINT STATEMENT                                                │
│  ───────────────                                                │
│  Your scratch pad. Fast, disposable, local.                     │
│  Best for: 30-second throwaway inspection during development.   │
│  Problem: disappears into the void in server processes;         │
│           must be deleted; can reach production accidentally.   │
│                                                                 │
│  IDE DEBUGGER / pdb                                             │
│  ────────────────────                                           │
│  Your magnifying glass. Interactive, real-time inspection.      │
│  Best for: local development, when you need to stop and look.   │
│  Problem: cannot attach to a remote server under load;          │
│           freezes the program at every breakpoint.              │
│                                                                 │
│  LOGGING                                                        │
│  ────────                                                       │
│  Your permanent record. Level-filtered, structured, persistent. │
│  Best for: servers, CI pipelines, Docker containers,            │
│            intermittent bugs that only appear under load.       │
│  Survives past your terminal session.                           │
│  Never freezes your program.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The basics:**

```python
import logging

# Configure once, at the top of your program or entry point
logging.basicConfig(level=logging.DEBUG)

# Now use it anywhere:
logger = logging.getLogger(__name__)

def build_grade_report(students: list[Student]) -> list[dict]:
    report = []
    for student in students:
        logger.debug("Processing student: %s, scores: %s",
                     student.name, student.scores)
        avg = student.average()
        grade = calculate_letter_grade(avg)
        logger.debug("Computed average=%.2f, grade=%s", avg, grade)
        report.append({"name": student.name, "average": avg, "grade": grade})
    return report
```

**The five severity levels — each has a specific meaning:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE FIVE LOG LEVELS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEBUG    (level 10)  — Detailed diagnostic information         │
│  ─────────────────────  "I am processing Diana, scores=[]"      │
│                         Use for: tracing execution flow,        │
│                         variable values during development.     │
│                         Suppress in production.                 │
│                                                                 │
│  INFO     (level 20)  — Normal, expected events                 │
│  ────────────────────   "Grade report generated for 5 students" │
│                         Use for: significant milestones,        │
│                         startup messages, request completions.  │
│                                                                 │
│  WARNING  (level 30)  — Unexpected but recoverable              │
│  ───────────────────    "Diana has empty scores; returning 0.0" │
│                         Use for: conditions that are wrong       │
│                         but the program can continue.           │
│                                                                 │
│  ERROR    (level 40)  — A failure occurred                      │
│  ──────────────────     "Failed to compute average for Diana"   │
│                         Use for: exceptions caught at the top   │
│                         level, operations that failed but did   │
│                         not crash the whole program.            │
│                                                                 │
│  CRITICAL (level 50)  — Fatal; program cannot continue          │
│  ─────────────────────  "Database connection lost; shutting down"│
│                         Use for: unrecoverable failures.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why logging is not the same as print — level filtering:**

```python
# Development: show everything
logging.basicConfig(level=logging.DEBUG)
# Output: ALL messages including DEBUG

# Production: suppress debug noise
logging.basicConfig(level=logging.WARNING)
# Output: only WARNING, ERROR, CRITICAL

# The key: you change ONE line to control ALL output.
# With print(), you'd have to delete or comment out every line.
```

**Replace the 5 print statements from Section 1.2 with logging:**

```python
# BEFORE (print debugging — 5 statements, all must be deleted)
print("DEBUG 1: starting main")
print(f"DEBUG 2: created {len(students)} students")
print(f"DEBUG 3: applied curve")
print(f"DEBUG 4: processing {student.name}")
print(f"DEBUG 5: scores = {self.scores}")

# AFTER (logging — stays in the code, filtered by level)
logger.debug("starting main")
logger.debug("created %d students", len(students))
logger.debug("applied curve of %d", curve)
logger.debug("processing %s", student.name)
logger.debug("scores = %s", self.scores)

# In development: logging.basicConfig(level=logging.DEBUG)
#   → All 5 messages appear

# In production: logging.basicConfig(level=logging.WARNING)
#   → Zero messages appear. Code unchanged. No cleanup needed.
```

> "From Week 3 onward, you will see error information appearing in server logs. When a FastAPI request fails, you won't get a crash in your terminal — you'll get a traceback in the log output. When a background job fails in Week 11, the failure lives in the worker logs. Every backend system communicates through logs. Learning to read them now, and to produce meaningful ones yourself, is not optional. This is where structlog comes in during Week 15 — it extends this same model with structured JSON output. The model you just learned is the foundation."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 4 — Choose the Right Debugging Tool
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  For each scenario, which tool would you reach for FIRST?
  (IDE debugger / pdb / logging / print statement)

  A) A FastAPI endpoint running on a remote production server
     returns HTTP 500 errors — but only under heavy load.
     You cannot reproduce it on your laptop.

  B) A function you just wrote locally returns -1 instead of the
     expected index. You want to watch which branch of an if/elif
     chain is being taken, live.

  C) You're SSH'd into a staging Linux server with no GUI.
     A script crashes with an AttributeError and you need to
     inspect the object at the crash point.

  D) A teammate asks: "Does your function crash on an empty list?"
     You want to check quickly before writing a proper test.

───────────────────────────────────────────────────────────────────
▼ SOLUTION
───────────────────────────────────────────────────────────────────

  A) LOGGING — it is the only tool that survives a remote,
     load-triggered scenario. Add DEBUG-level log statements at
     key points, deploy, reproduce under load, read the log file.
     Neither the IDE debugger nor pdb can attach to a remote
     process that only fails under concurrent request pressure.

  B) IDE DEBUGGER — you have local access and a GUI. Set a
     breakpoint on the if statement, step through to watch exactly
     which branch executes with the actual input. Interactive
     inspection is ideal here.

  C) pdb / breakpoint() — no GUI means no IDE debugger. Insert
     breakpoint() at the crash site, run the script, interact via
     the terminal. pdb is precisely designed for this scenario.

  D) PRINT STATEMENT — a quick print(result) for a 30-second
     local check is entirely appropriate. The print statement is
     the scalpel: fast, disposable, local. You'll delete it and
     write a proper test immediately after confirming.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# PART 4: THE METHOD

## 4.1 Hypothesis-Driven Debugging

**The debugger shows you WHAT is happening. But you need a METHOD for deciding WHERE to look.**

After fixing Diana's crash in Part 3, the program runs. But the output showed something wrong:

```
  Eve:     B (90.0)   ← Expected A. 90.0 is the minimum for an A.
```

No crash. No traceback. The program "works" — it just gives the wrong answer.

> "This is the hardest kind of bug. No error message. No red text. Just wrong output. This is where wandering with a debugger fails and hypothesis-driven debugging succeeds."

**Hypothesis-driven debugging works like a doctor diagnosing a patient:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HYPOTHESIS-DRIVEN DEBUGGING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. OBSERVE the symptom                                         │
│     "What went wrong?"                                          │
│     → Eve gets B with average 90.0. She should get A.           │
│                                                                 │
│  2. HYPOTHESIZE a cause                                         │
│     "What could produce this?"                                  │
│     → "The letter grade function must be using strict >         │
│        instead of >= for the A boundary at 90."                 │
│                                                                 │
│  3. PREDICT what you'd see if the hypothesis is correct         │
│     "If > is used instead of >=, then calculate_letter_grade    │
│      returns B for an input of exactly 90.0."                   │
│                                                                 │
│  4. TEST — check the prediction                                 │
│     Set breakpoint on grade = calculate_letter_grade(avg)       │
│     when student is Eve (conditional: student.name == "Eve")    │
│     Step INTO calculate_letter_grade.                           │
│     Watch the if/elif chain execute with avg = 90.0.            │
│     → if 90.0 > 90: → False. Falls through.                    │
│     → elif 90.0 >= 80: → True. Returns "B".                    │
│     → Hypothesis confirmed.                                     │
│                                                                 │
│  5. FIX and verify                                              │
│     Change: if avg > 90:                                        │
│     To:     if avg >= 90:                                       │
│     Re-run. Eve now gets A (90.0). Alice and Charlie still      │
│     get A (91.2, 97.8 — both > 90 and ≥ 90, both unaffected).  │
│     Case closed.                                                │
│                                                                 │
│                                                                 │
│  THE KEY: Don't wander. Have a THEORY before you look.          │
│  Every debugging action should TEST a specific hypothesis.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The wrong approach (no hypothesis):**

```
"It's broken. Let me set a breakpoint at the start and
step through every single line... line 1... line 2...
line 3... what does this variable do... line 4..."

This takes 10x longer because you're watching EVERYTHING
instead of testing a SPECIFIC theory.
```

> "Wandering through code with a debugger is only slightly better than wandering with print statements. The debugger is a microscope — but a microscope is useless if you don't know which slide to put under it. The hypothesis tells you where to look."

---

## 4.2 Binary Search Debugging

**When the bug could be anywhere in a long pipeline, don't check every step. Cut the search space in half.**

We know buggy_grades.py processes data through a 4-stage pipeline. The output is wrong — Eve gets B instead of A. Suppose we didn't immediately form the hypothesis about `>` vs `>=`. Binary search tells us which stage to investigate.

```
PIPELINE:
Stage 1 → Stage 2 → Stage 3 → Stage 4 → WRONG OUTPUT
load()     parse()   curve()   report()   Eve: B (90.0)

Bug is in one of the 4 stages. Which one?
```

```
┌─────────────────────────────────────────────────────────────────┐
│                BINARY SEARCH DEBUGGING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  4 stages. Bug is in ONE of them.                               │
│                                                                 │
│  Stage 1 ─ Stage 2 ─ Stage 3 ─ Stage 4                          │
│                                                                 │
│                                                                 │
│  CHECK 1: Inspect data AFTER Stage 2 (halfway)                  │
│  ─────────────────────────────────────────────                  │
│  Breakpoint after parse_students(raw_data)                      │
│  p students →                                                   │
│    [Student(name='Alice', scores=[85, 92, 78, 90]),             │
│     Student(name='Bob', ...), Student(name='Charlie', ...),     │
│     Student(name='Diana', scores=[]),                           │
│     Student(name='Eve', scores=[75, 85, 85, 95])]               │
│                                                                 │
│  Eve's raw scores parsed correctly ✅                           │
│  Bug is NOT in Stages 1–2. It's in Stages 3 or 4.              │
│                                                                 │
│  Stage 1 ─ Stage 2 ─ Stage 3 ─ Stage 4                          │
│  ────── eliminated ──────   ─── bug is here ───                │
│                                                                 │
│                                                                 │
│  CHECK 2: Inspect data AFTER Stage 3                            │
│  ──────────────────────────────────                             │
│  Breakpoint after apply_curve(students, 5)                      │
│  p [s for s in curved if s.name == "Eve"] →                    │
│    [Student(name='Eve', scores=[80, 90, 90, 100])]              │
│                                                                 │
│  Eve's curved scores look correct ✅                            │
│  Expected post-curve average: (80+90+90+100)/4 = 90.0 ✅       │
│  Stage 3 is correct. Bug must be in Stage 4.                    │
│                                                                 │
│  Stage 1 ─ Stage 2 ─ Stage 3 ─ Stage 4                          │
│  ────── eliminated ──────   ── ok ──   ─BUG─                   │
│                                                                 │
│                                                                 │
│  FOUND IT: 2 checks instead of 4.                               │
│                                                                 │
│  Look at Stage 4 — build_grade_report.                          │
│  Step into calculate_letter_grade(90.0):                        │
│  if avg > 90: → False   (90.0 is NOT strictly greater than 90) │
│  elif avg >= 80: → True → returns "B"                           │
│                                                                 │
│  Fix: if avg >= 90:  → 90.0 >= 90 is True → returns "A"        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The principle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  LINEAR SEARCH: Check stage 1, then 2, then 3... worst case N  │
│  BINARY SEARCH: Check the middle, eliminate half, repeat        │
│                 worst case log₂(N)                              │
│                                                                 │
│  4 stages:                                                      │
│    Linear: up to 4 checks                                       │
│    Binary: up to 2 checks (log₂4 = 2)                          │
│                                                                 │
│  20 stages (large codebase):                                    │
│    Linear: up to 20 checks                                      │
│    Binary: up to 5 checks (log₂20 ≈ 4.3)                       │
│                                                                 │
│  This is the SAME principle behind binary search in algorithms. │
│  And it works for debugging because data either looks correct   │
│  at a checkpoint or it doesn't — that tells you which half      │
│  contains the bug.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "There is a Git command called `git bisect` that applies binary search to your *commit history* — it finds which commit introduced a bug. This is not something we've covered yet, but the mental model is already built. When you encounter it, the concept will be immediately familiar."

---

## Mutation Tracking — Finding Exactly Where Data Changes

Binary search locates **which stage** contains a bug. Mutation tracking finds **the exact line** within that stage where a value goes wrong.

**The scenario:** binary search told you Eve's wrong grade originates in Stage 4 (`build_grade_report`). Inside that function there are three operations: compute the average, compute the letter grade, and assemble the dict. Which one produces the wrong value?

**Tool 1: Watch expressions in the IDE debugger**

Set a conditional breakpoint on the loop entry: `student.name == "Eve"`. Add these expressions to the Watch pane:

```
WATCH:
  avg                        → 90.0   (updates each step)
  calculate_letter_grade(90.0)  → shows return value live
  student.scores             → [80, 90, 90, 100]
```

Step through the loop body. The moment a watch expression changes to an unexpected value, the bug is on the line you just stepped over. You do not need to inspect every variable — only the one your hypothesis says is wrong.

**Tool 2: Logging at mutation boundaries**

When interactive stepping is impractical (large data, intermittent crash), add `logger.debug()` immediately before and after every operation that transforms the data:

```python
def build_grade_report(students: list[Student]) -> list[dict]:
    report = []
    for student in students:
        logger.debug("Entered loop — student=%s scores=%s",
                     student.name, student.scores)
        avg = student.average()
        logger.debug("After average — avg=%.2f", avg)
        grade = calculate_letter_grade(avg)
        logger.debug("After grade — grade=%s", grade)
        report.append({"name": student.name, "average": avg, "grade": grade})
    return report
```

With `logging.basicConfig(level=logging.DEBUG)`:

```
DEBUG Entered loop — student=Eve scores=[80, 90, 90, 100]
DEBUG After average — avg=90.00
DEBUG After grade — grade=B              ← wrong here
```

The log pinpoints `calculate_letter_grade` as the step where the value goes wrong. Before you touch anything else, you know exactly where to look.

**Tool 3: Snapshot copies to confirm a function does not mutate its input**

When you suspect a function is modifying data it should not touch, take a temporary snapshot before the call and assert afterward:

```python
import copy

# Temporary debugging aid — remove after fix
before = copy.deepcopy(curved)
report = build_grade_report(curved)
assert curved == before, "build_grade_report mutated its input!"
```

If `curved` was silently modified inside `build_grade_report`, the assertion fires immediately with a clear message. This is especially useful in pipelines where stage outputs feed the next stage — an accidental mutation in Stage 3 would corrupt Stage 4's input invisibly without this check.

> "Binary search tells you the ZIP code. Mutation tracking tells you the address. Watch expressions work best when you can reproduce the bug interactively. Logging works best when you cannot. The snapshot copy works best when you suspect aliasing — when two variables point to the same underlying list and a function that should read-only is quietly writing."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 5 — Apply Binary Search Debugging
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  You have a 6-stage pipeline. The final output is wrong but
  there is no crash:

  Stage 1: load_config()
  Stage 2: validate_input()
  Stage 3: transform_data()
  Stage 4: apply_filters()
  Stage 5: aggregate_results()
  Stage 6: format_output()

  Using binary search:

  a) Which stage do you inspect FIRST, and why?
  b) The output after that stage looks correct. Now where do
     you look?
  c) That stage also looks correct. Where is the bug?
  d) What is the maximum number of checks you would ever need,
     no matter where the bug is in a 6-stage pipeline?

───────────────────────────────────────────────────────────────────
▼ SOLUTION
───────────────────────────────────────────────────────────────────

  a) After Stage 3 — the midpoint. Set a breakpoint after
     transform_data() and inspect its output. "Is the data
     correct here?" If yes, the bug is in Stages 4–6.
     If no, the bug is in Stages 1–3.

  b) Stage 3 is correct, so the bug is in Stages 4–6.
     Check the midpoint of the remaining range: after Stage 5
     (aggregate_results). "Is the aggregated data correct?"
     If yes, the bug is in Stage 6 only.
     If no, the bug is in Stages 4 or 5.

  c) Stages 3 and 5 are both correct. The bug must be in Stage 6
     (format_output) — the only remaining uninspected stage.
     Look at format_output directly.

  d) Maximum 3 checks: ⌈log₂(6)⌉ = 3.
     Compare to linear search: up to 6 checks.
     As the pipeline grows to 20 stages, linear requires 20 checks;
     binary requires at most 5.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4.3 Rubber Duck Debugging

**Sometimes you don't need a tool. You need a listener.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  RUBBER DUCK DEBUGGING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The method:                                                    │
│  1. Get a rubber duck (or any object, or a wall).               │
│  2. Explain your code to the duck, LINE BY LINE.                │
│  3. "On this line, I take the list of students and..."          │
│  4. "Then I calculate the average by dividing the sum by..."    │
│  5. "Wait. What if the list is empty?"                          │
│                                                                 │
│  🦆 The duck didn't say anything. YOU figured it out.           │
│                                                                 │
│                                                                 │
│  WHY IT WORKS:                                                  │
│                                                                 │
│  When you read code silently, your brain SKIMS.                 │
│  It fills in what it EXPECTS to see instead of what's THERE.    │
│                                                                 │
│  When you explain code out loud, you're forced to               │
│  articulate EVERY assumption. The bug hides in the              │
│  assumption you never questioned.                               │
│                                                                 │
│  "I assume scores always has at least one element."             │
│  ← THAT is the bug. You just didn't know it was an             │
│     assumption until you said it out loud.                      │
│                                                                 │
│                                                                 │
│  WHEN TO USE IT:                                                │
│  ├─ You've been staring at the same code for 15+ minutes        │
│  ├─ The debugger shows "correct" values but it still fails      │
│  ├─ You're confused about the LOGIC, not the mechanics          │
│  └─ Before asking someone else (often, you solve it yourself)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked demonstration — rubber-ducking Eve's wrong grade:**

> "Let me narrate this out loud. I'm looking at `calculate_letter_grade`. On this line, I check `if avg > 90`. I pass in Eve's average. Eve's pre-curve scores are `[75, 85, 85, 95]`. Sum is 340, divided by 4, that's 85.0. With the curve of 5 applied to each score, her new scores are `[80, 90, 90, 100]`. Sum is 360, divided by 4, equals 90.0. I'm passing 90.0 to the function. Now the check: is 90.0 greater than 90? No. 90.0 is equal to 90. So the condition is False. It falls through to the next check: is 90.0 greater than or equal to 80? Yes. Returns B."

> "Wait. I assumed that 'checking for an A' meant 'is the average at least 90.' But the code checks 'is the average strictly above 90.' Those are different things. I wrote `>` when I meant `>=`. The bug is hidden in the assumption that my comparison operator was correct. I never questioned it until I had to say it out loud."

This is rubber duck debugging. The duck said nothing. The narration forced the assumption into words.

> "This sounds silly. It works. Professional developers with 20 years of experience do this. The act of translating code from 'something you read' into 'something you explain' engages a different part of your brain. Try it before you dismiss it."

---

## 4.4 When to Ask for Help (And How)

**There IS a time to stop debugging alone. Here's when, and here's how to do it well.**

**When to ask:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHEN TO ASK FOR HELP                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ ASK when:                                                   │
│  ├─ You've spent 30+ minutes on the same bug                    │
│  ├─ You've formed and tested at least 2-3 hypotheses            │
│  ├─ You've read the traceback and searched for the error        │
│  ├─ You've tried the debugger and can describe what you saw     │
│  └─ You can clearly state WHAT you expected vs WHAT happened    │
│                                                                 │
│  ❌ DON'T ask when:                                             │
│  ├─ You haven't read the error message yet                      │
│  ├─ You can't describe what you've already tried                │
│  ├─ Your question is "it doesn't work" (that tells no one any.) │
│  └─ You haven't searched for the error message online           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**THE template for asking for help — filled out with the Diana bug as a live example:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE HELP REQUEST TEMPLATE (with example)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. WHAT I'M TRYING TO DO                                       │
│     "I'm calculating letter grades for a list of Student        │
│      objects, each with a name and a list of scores."           │
│                                                                 │
│  2. WHAT I EXPECTED TO HAPPEN                                   │
│     "All five students receive a grade and average.             │
│      Diana has no scores, so I expected either a graceful       │
│      default or an informative error."                          │
│                                                                 │
│  3. WHAT ACTUALLY HAPPENED                                      │
│     "ZeroDivisionError on line 10. Full traceback:              │
│      File 'buggy_grades.py', line 10, in average                │
│        return sum(self.scores) / len(self.scores)               │
│      ZeroDivisionError: division by zero"                       │
│                                                                 │
│  4. WHAT I'VE ALREADY TRIED                                     │
│     "I set a conditional breakpoint in build_grade_report()     │
│      for student.name == 'Diana' and confirmed she has an       │
│      empty scores list. I'm not sure whether to return 0.0,     │
│      skip her in the report, or raise a ValueError."            │
│                                                                 │
│  5. MINIMAL CODE TO REPRODUCE                                   │
│     "@dataclass                                                  │
│      class Student:                                             │
│          scores: list[int]                                      │
│          def average(self) -> float:                            │
│              return sum(self.scores) / len(self.scores)         │
│      Student(scores=[]).average()   # reproduces the crash"     │
│                                                                 │
│  6. ENVIRONMENT                                                  │
│     "Python 3.12, macOS, running with uv run."                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice item 5: building the minimal code to reproduce the problem. This is itself a debugging technique. The process of isolating a bug into the smallest possible reproduction case forces you to eliminate variables until only the essential ones remain. Frequently, that isolation process reveals the bug before you ever share it. Build the minimal example. Share it if you haven't solved it. Either way, you win."

**One common mistake when asking for help:** asking about your attempted solution rather than your actual goal. If you're trying to get a file's extension and you ask "how do I get the last 3 characters of a string," you'll get an answer to the wrong question — and miss that `Path.suffix` solves the real problem cleanly. Always state what you're trying to DO, not just what you tried. This is known as the XY Problem: you want X, you think Y is the solution, Y doesn't work, and you ask about Y when you should have asked about X.

---

## 4.5 Assertions as Early Bug Detection

**The best time to catch a bug is at the moment the assumption behind it is first violated — not five functions later when the corrupted data finally causes a crash.**

The central thesis of this lecture is: a bug is a gap between what you *think* your code does and what it *actually* does. Assertions are automated gap-detectors. An `assert` statement checks an assumption about program state every time it executes. When the assumption is wrong, execution halts immediately — at the exact point of violation, with an error message you control.

Without assertions, Diana's empty-scores bug produced `ZeroDivisionError: division by zero` inside `average()` — with no indication of which student caused it. With an assertion at the right location, the failure says exactly what went wrong, where, and for whom.

```python
# WITHOUT assertions — opaque crash deep in the call chain
def average(self) -> float:
    return sum(self.scores) / len(self.scores)
# → ZeroDivisionError: division by zero  (which student? why?)

# WITH an assertion — immediate, informative failure
def average(self) -> float:
    assert len(self.scores) > 0, \
        f"Cannot compute average: '{self.name}' has no scores"
    return sum(self.scores) / len(self.scores)
# → AssertionError: Cannot compute average: 'Diana' has no scores
```

```
┌─────────────────────────────────────────────────────────────────┐
│               THREE PLACES TO PUT ASSERTIONS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNTAX:                                                        │
│  assert <condition>, "<message if condition is False>"          │
│                                                                 │
│                                                                 │
│  PRECONDITIONS — validate inputs before using them              │
│  ────────────────────────────────────────────────               │
│  Assert what you assume must be true BEFORE the function runs.  │
│                                                                 │
│  def build_grade_report(students: list[Student]) -> list[dict]: │
│      assert students, "student list must not be empty"          │
│      assert all(len(s.scores) > 0 for s in students), \        │
│          "every student must have at least one score"           │
│      ...                                                        │
│                                                                 │
│  The crash now happens at the entry point, not inside           │
│  average() three calls later.                                   │
│                                                                 │
│                                                                 │
│  POSTCONDITIONS — validate outputs before returning them        │
│  ───────────────────────────────────────────────────────        │
│  Assert that what you are about to return is valid.             │
│                                                                 │
│  def apply_curve(students, curve: int) -> list[Student]:        │
│      curved = [Student(s.name, [x+curve for x in s.scores])    │
│                for s in students]                               │
│      assert len(curved) == len(students), \                     │
│          "apply_curve dropped students"                         │
│      assert all(s.scores for s in curved), \                    │
│          "apply_curve produced empty score lists"               │
│      return curved                                              │
│                                                                 │
│                                                                 │
│  INVARIANTS — validate mid-execution assumptions                │
│  ─────────────────────────────────────────────                  │
│  Assert properties that must hold at a specific point in        │
│  a loop or pipeline — not just at function boundaries.          │
│                                                                 │
│  for student in students:                                       │
│      avg = student.average()                                    │
│      assert 0.0 <= avg <= 100.0, \                              │
│          f"Average out of valid range for {student.name}: {avg}"│
│      grade = calculate_letter_grade(avg)                        │
│                                                                 │
│                                                                 │
│  WHAT ASSERTIONS ARE NOT:                                       │
│  ─────────────────────────                                      │
│  ├─ Not for validating user or external input (use Pydantic     │
│     or raise ValueError — external data is expected to be bad) │
│  ├─ Not for error handling in production paths (use raise)       │
│  └─ Not guaranteed to run — python -O strips assert statements  │
│     entirely. Never assert something safety-critical that        │
│     must always be checked.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A complete set of assertions for buggy_grades.py:**

```python
def load_raw_data() -> list[dict]:
    data = [
        {"name": "Alice",   "scores": [85, 92, 78, 90]},
        {"name": "Bob",     "scores": [72, 68, 75, 80]},
        {"name": "Charlie", "scores": [95, 88, 91, 97]},
        {"name": "Diana",   "scores": []},
        {"name": "Eve",     "scores": [75, 85, 85, 95]},
    ]
    for entry in data:
        assert "name" in entry and "scores" in entry, \
            f"Malformed record: {entry}"
        assert isinstance(entry["scores"], list), \
            f"'scores' must be a list for {entry.get('name', '?')}"
    return data   # Diana's empty list passes here — and is caught below

def average(self) -> float:
    assert len(self.scores) > 0, \
        f"Cannot average empty scores: student '{self.name}'"
    return sum(self.scores) / len(self.scores)   # ZeroDivisionError is now impossible

def calculate_letter_grade(avg: float) -> str:
    assert 0.0 <= avg <= 100.0, \
        f"Average {avg:.1f} is outside the valid range [0.0, 100.0]"
    if avg >= 90:
        return "A"
    ...
```

Running with these assertions, Diana's bug produces:
```
AssertionError: Cannot average empty scores: student 'Diana'
```
before the program reaches Stage 4. The message names the student. The stack trace points to `average()`. No hunting required.

> "Every `assert` is a comment that bites. `# I believe self.scores will always have elements` is invisible to the runtime. `assert len(self.scores) > 0` is verified every time. Write the comment version first — then ask whether it should be an assertion. If your code would produce silently wrong results when the comment is violated, it should be an assertion."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 8 — Write Defensive Assertions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Given this function:

  def build_grade_report(students: list[Student]) -> list[dict]:
      report = []
      for student in students:
          avg = student.average()
          grade = calculate_letter_grade(avg)
          report.append({
              "name": student.name,
              "average": avg,
              "grade": grade,
          })
      return report

  a) Write ONE precondition assertion that catches Diana's
     empty-scores bug before any computation begins.
     Your message must name what invariant was violated.

  b) Write ONE postcondition assertion on the returned report
     that would catch a bug where a student was silently skipped.

  c) Should the empty-scores condition in average() be an
     assert or a raise ValueError? Explain the difference
     and when to use each.

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) Precondition — validate all students before the loop starts:
     assert all(len(s.scores) > 0 for s in students), \
         "Every student must have at least one score before " \
         "building the grade report"
     This fires immediately when Diana (empty scores) is in
     the list, with a message that describes the requirement.

  b) Postcondition — verify the report has the right length:
     assert len(report) == len(students), \
         f"Report has {len(report)} entries but {len(students)} " \
         "students were provided — a student was silently dropped"

  c) assert: "I, the programmer, guarantee this will never happen
              given correct internal usage of my own module."
              Appropriate when empty scores is a programming error
              that should never reach average() — it should be
              caught upstream.

     raise ValueError: "This is a real, foreseeable condition from
                        external data. The caller must handle it."
                        Appropriate when your function is a public
                        API that may receive empty scores from
                        user-supplied data.

     For a production grading system: raise, not assert. A student
     with no scores is a real business scenario — it should produce
     a clear error or a defined default, not an AssertionError
     that crashes the process and leaks internal assumptions.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4.6 Performance Debugging

**Programs have three types of bugs: crashes, wrong output, and slow execution. You now have tools for the first two. This section covers the third.**

A program can run correctly — no crashes, correct results — and still be a bug report in waiting if it takes 30 seconds to do what should take 30 milliseconds. Performance problems are bugs. Finding them requires a specific discipline: **measure first, change second, confirm afterward**.

> "The cardinal sin of optimization is changing code without measuring first. Developers consistently guess wrong about where the bottleneck is. They optimize the loop they just wrote while a database query 3 layers away is consuming 95% of the runtime. Profile before touching anything. No exceptions."

**Tool 1: `cProfile` — which functions consume time**

`cProfile` is Python's built-in profiler. It records every function call, how many times it ran, and how long it took.

```python
import cProfile
import pstats

# Profile main() and print a report
with cProfile.Profile() as pr:
    main()

stats = pstats.Stats(pr)
stats.sort_stats(pstats.SortKey.CUMULATIVE)
stats.print_stats(15)    # Top 15 functions by cumulative time
```

**What the output looks like:**

```
         2031 function calls in 3.124 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.001    0.001    3.124    3.124 buggy_grades.py:64(main)
        1    0.000    0.000    3.122    3.122 buggy_grades.py:46(build_grade_report)
        5    3.119    0.624    3.119    0.624 buggy_grades.py:12(calculate_letter_grade)
        5    0.002    0.000    0.002    0.000 buggy_grades.py:8(average)
       ...
```

**Reading the cProfile output:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 READING cProfile OUTPUT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COLUMNS:                                                       │
│  ncalls   — how many times this function was called             │
│  tottime  — time spent INSIDE this function only                │
│             (excludes time in functions it called)              │
│  percall  — tottime ÷ ncalls                                    │
│  cumtime  — total time including all called sub-functions       │
│  percall  — cumtime ÷ ncalls                                    │
│                                                                 │
│                                                                 │
│  FINDING THE BOTTLENECK:                                        │
│                                                                 │
│  Sort by cumtime (CUMULATIVE sort — the default here).          │
│                                                                 │
│  The top entry (main) is NOT the bottleneck — it wraps         │
│  everything. Look for the function with:                        │
│                                                                 │
│  ├─ HIGH tottime relative to its cumtime                        │
│  │   → It IS the slow work, not just a wrapper around it.      │
│  │                                                              │
│  ├─ HIGH percall with many ncalls                               │
│  │   → Each call is slow AND it is called many times.          │
│  │                                                              │
│  └─ Surprising tottime for something you expected to be fast    │
│      → An assumption about complexity is wrong.                 │
│                                                                 │
│  From the example above:                                        │
│  calculate_letter_grade — 5 calls, 3.119s TOTAL                 │
│  That is 0.624 seconds per call for a simple if/elif.           │
│  Something is drastically wrong inside that function.           │
│  average() — 5 calls, 0.002s total. NOT the bottleneck.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tool 2: `timeit` — precise micro-benchmarks**

`cProfile` shows the whole program. `timeit` measures one operation in isolation, running it thousands of times to eliminate noise.

```python
import timeit

# Compare two implementations of the same operation
list_lookup = timeit.timeit(
    stmt="85 in scores_list",
    setup="scores_list = list(range(1000))",
    number=100_000
)
set_lookup = timeit.timeit(
    stmt="85 in scores_set",
    setup="scores_set = set(range(1000))",
    number=100_000
)

print(f"list: {list_lookup:.4f}s for 100k lookups")
print(f"set:  {set_lookup:.4f}s for 100k lookups")
```

```
list: 1.8431s for 100k lookups
set:  0.0042s for 100k lookups   ← 400x faster
```

Use `timeit` when: cProfile has identified a function as the bottleneck, and you want to compare two alternative implementations before committing to one.

**Tool 3: `time.perf_counter()` — quick wall-clock measurement**

```python
import time

start = time.perf_counter()
result = expensive_function()
elapsed = time.perf_counter() - start
print(f"expensive_function: {elapsed * 1000:.1f} ms")
```

Use `perf_counter` for a quick "is this even slow?" check before pulling out `cProfile`. It gives you total elapsed time for a block, not a function-level breakdown.

```
┌─────────────────────────────────────────────────────────────────┐
│             THREE TOOLS — THREE QUESTIONS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  time.perf_counter()                                            │
│  ────────────────────                                           │
│  "Is this block slow?"                                          │
│  Total elapsed wall-clock time for any section of code.         │
│  Fastest to add. Least informative. The first tool to reach for.│
│                                                                 │
│  cProfile                                                       │
│  ────────                                                       │
│  "Which function is slow?"                                      │
│  Full call-level breakdown of an entire program run.            │
│  Reach for this when you know something is slow but not where.  │
│                                                                 │
│  timeit                                                         │
│  ───────                                                        │
│  "Which of these two implementations is faster?"                │
│  Precise micro-benchmark of one expression, run many times.     │
│  Reach for this after cProfile has found the guilty function    │
│  and you are comparing candidate fixes.                         │
│                                                                 │
│  THE WORKFLOW:                                                   │
│  1. Observe slowness — perf_counter confirms: "yes, it's slow"  │
│  2. Profile — cProfile finds: "calculate_letter_grade, 3.1s"   │
│  3. Read the function — find the hidden complexity              │
│  4. Benchmark alternatives — timeit: "implementation B is 50x"  │
│  5. Confirm the fix — perf_counter shows overall improvement    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked example — a hidden O(n²) function:**

```python
# Correct output. Works fine with 5 students. Ships to production.
def find_duplicate_names(students: list[Student]) -> list[str]:
    duplicates = []
    for i, s in enumerate(students):
        for j, other in enumerate(students):
            if i != j and s.name == other.name:
                if s.name not in duplicates:
                    duplicates.append(s.name)
    return duplicates
```

With 5 students: instant. With 10,000 students: 100 million comparisons, several seconds. `cProfile` on a 10,000-student dataset would show this function consuming 99% of total runtime.

Looking at the loops: an outer loop over every student, an inner loop over every student. That is O(n²). The fix is a single-pass set lookup:

```python
def find_duplicate_names(students: list[Student]) -> list[str]:
    seen: set[str] = set()
    duplicates: set[str] = set()
    for s in students:
        if s.name in seen:
            duplicates.add(s.name)
        seen.add(s.name)
    return list(duplicates)   # O(n) — one pass, O(1) set lookup each iteration
```

`timeit` comparison on 10,000 names: the set version runs 2,000x faster. This is not a micro-optimization — it is the difference between a feature that works and one that makes your server time out.

> "Performance bugs are the slowest-burning bugs in software. They ship in code that passes all tests, runs correctly in development with 10 records, and fails catastrophically in production with 100,000. cProfile is your first response. Measure, then move."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 9 — Read a cProfile Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  You profile a report-generation endpoint and receive this:

   ncalls  tottime  percall  cumtime  percall  function
        1    0.000    0.000    6.104    6.104  main
        1    0.001    0.001    6.103    6.103  generate_report
      500    0.003    0.000    6.099    0.012  process_row
      500    6.095    0.012    6.095    0.012  fetch_user_from_db
        1    0.001    0.001    0.001    0.001  format_output

  a) Which function is the bottleneck, and how do you know?

  b) process_row has cumtime 6.099s. fetch_user_from_db has
     tottime 6.095s. What does this tell you about the
     relationship between the two functions?

  c) fetch_user_from_db is called once per row, for 500 rows.
     Name TWO fixes you would investigate before rewriting
     any algorithm, and explain why each would help.

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) fetch_user_from_db — tottime = 6.095s across 500 calls.
     tottime measures time spent INSIDE the function itself,
     not in functions it calls. 6.095s out of 6.104s total
     runtime — over 99% of the program is spent here.
     format_output at 0.001s is invisible by comparison.

  b) process_row's cumtime (6.099s) ≈ fetch_user_from_db's
     tottime (6.095s). Almost all of process_row's execution
     time is inside fetch_user_from_db. process_row is simply
     a wrapper that calls fetch_user_from_db for every row.
     This is the N+1 pattern: 500 individual database round-trips
     instead of one query that fetches all 500 rows at once.

  c) Two fixes to investigate before rewriting:
     1. Batch the query: fetch all 500 users in a single
        SELECT WHERE id IN (...) before the loop. Do
        id-based dict lookups inside the loop. This reduces
        500 database round-trips to 1.
     2. Cache: if the same users appear repeatedly across
        different report runs, an lru_cache or Redis cache
        would serve repeated lookups without hitting the DB.
        Check whether the data is stable enough to cache.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4.7 Memory Debugging

**A program can be fast, crash-free, and produce correct output — and still slowly consume your production server's memory until the process is killed.**

Memory bugs are dangerous precisely because they are invisible. A server that uses 200 MB at startup and grows to 4 GB over 72 hours will crash with no warning. The tools to find this are in Python's standard library.

**The most common causes in Python:**

```
┌─────────────────────────────────────────────────────────────────┐
│                COMMON MEMORY BUG SOURCES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNBOUNDED ACCUMULATION                                         │
│  ──────────────────────                                         │
│  A list, dict, or set that items are appended to but never      │
│  removed from. Classic form: caching without eviction.          │
│                                                                 │
│  _cache = {}                                                    │
│  def get_report(key):                                           │
│      if key not in _cache:                                      │
│          _cache[key] = expensive_compute(key)  # grows forever  │
│      return _cache[key]                                         │
│                                                                 │
│                                                                 │
│  REFERENCE CYCLES                                               │
│  ────────────────                                               │
│  Object A holds a reference to B; B holds one back to A.        │
│  Python's garbage collector handles simple cycles, but cycles   │
│  involving __del__ methods can prevent collection entirely.     │
│                                                                 │
│                                                                 │
│  UNCLOSED RESOURCES                                             │
│  ───────────────────                                            │
│  File handles, database connections, or network sockets opened  │
│  but never closed. Context managers (Lecture 2) prevent this   │
│  — `with open(...)` ensures closure even on exception.          │
│                                                                 │
│                                                                 │
│  LARGE OBJECTS RETAINED IN SCOPE                                │
│  ────────────────────────────────                               │
│  Loading a large dataset into a module-level variable for       │
│  processing, then never deleting it — the entire dataset        │
│  stays in memory for the lifetime of the process.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tool 1: `tracemalloc` — where is memory being allocated**

`tracemalloc` is Python's built-in memory allocation tracker. It records the source location of every allocation.

```python
import tracemalloc

tracemalloc.start()   # Begin tracking all allocations

# Run the code you want to profile
for i in range(100):
    handle_request(i)

# Take a snapshot and show the top consumers by source line
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("Top 5 memory consumers by source line:")
for stat in top_stats[:5]:
    print(stat)
```

**Sample output:**

```
Top 5 memory consumers by source line:
api_handler.py:88: size=9.4 MiB, count=80000, average=123 B
api_handler.py:41: size=0.3 MiB, count=300, average=1.0 KiB
```

Reading it:
- `size` — total memory currently allocated from this line
- `count` — number of live allocation objects from this line
- `average` — average size per object

Line 88 has 80,000 live objects. If that line is inside a function that runs once per request, and your server has served 80,000 requests, nothing is being cleaned up.

**Tool 2: Comparing snapshots to isolate a leak**

```python
import tracemalloc

tracemalloc.start()

handle_request(0)                              # Warm-up
snapshot1 = tracemalloc.take_snapshot()        # Baseline

for i in range(1, 100):
    handle_request(i)

snapshot2 = tracemalloc.take_snapshot()        # After 99 more requests

# Show what GREW between the two snapshots
top_stats = snapshot2.compare_to(snapshot1, 'lineno')
print("Memory growth over 99 requests:")
for stat in top_stats[:5]:
    print(stat)
```

```
Memory growth over 99 requests:
api_handler.py:88: +8.1 MiB (+79000 objects)
api_handler.py:41: +0.0 MiB (+0 objects)
```

Line 88 grew by 8.1 MB and 79,000 objects across 99 requests. Line 41 is stable. The leak is at line 88.

**Tool 3: `gc` for reference cycle detection**

```python
import gc

gc.set_debug(gc.DEBUG_LEAK)   # Report objects that cannot be freed
gc.collect()                  # Force a full garbage collection pass
# Python prints any objects involved in uncollectable cycles
```

```
┌─────────────────────────────────────────────────────────────────┐
│             MEMORY DEBUGGING WORKFLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Confirm growth is real                                 │
│  ─────────────────────────────                                  │
│  import os, psutil                                              │
│  proc = psutil.Process(os.getpid())                             │
│  mb = proc.memory_info().rss / 1024 / 1024                     │
│  print(f"Memory: {mb:.1f} MB")                                 │
│  Run before and after N requests. If memory grows and does      │
│  not shrink, you have a leak worth investigating.               │
│                                                                 │
│  STEP 2: tracemalloc snapshot comparison                        │
│  ──────────────────────────────────────                         │
│  Take snapshot at baseline, run one full cycle, take another.  │
│  compare_to() shows what grew. That is your suspect.            │
│                                                                 │
│  STEP 3: Inspect the growing line                               │
│  ─────────────────────────────────                              │
│  Is it a collection that items are added to but never removed   │
│  from? Is it a resource that is opened but not closed?          │
│  Is it a callback registered but never de-registered?           │
│                                                                 │
│  STEP 4: If Step 3 finds no obvious cause, check for cycles    │
│  ──────────────────────────────────────────────────────────     │
│  gc.set_debug(gc.DEBUG_LEAK) + gc.collect()                     │
│  Python will print objects in reference cycles that prevent     │
│  automatic collection.                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked example — the unbounded cache:**

```python
# This looks like a performance improvement. It is also a slow leak.
_report_cache: dict[int, dict] = {}   # Module-level, never evicted

def get_student_report(student_id: int) -> dict:
    if student_id not in _report_cache:
        _report_cache[student_id] = compute_report(student_id)
    return _report_cache[student_id]
```

After serving 50,000 unique student IDs, `_report_cache` holds 50,000 entries. `tracemalloc` would show this dict dominating the allocation statistics after many requests.

**Fix: bounded cache with `functools.lru_cache`:**

```python
from functools import lru_cache

@lru_cache(maxsize=512)   # Keeps the 512 most recently used entries
def get_student_report(student_id: int) -> dict:
    return compute_report(student_id)
```

`lru_cache` evicts the least-recently-used entry when `maxsize` is reached. Memory use plateaus at 512 entries rather than growing indefinitely. No external library required.

> "Memory bugs are the slowest-burning production failures in existence. The server is healthy at 9 AM and dead at 9 PM. `tracemalloc.compare_to()` is your most important tool: take a snapshot, run one request cycle, take another. The diff shows you exactly what grew. The fix is almost always 'this collection needs a size limit.'"

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 10 — Diagnose a Memory Leak
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  A tracemalloc compare_to() report shows this after 50 requests:

  +11.2 MiB  server.py:103  count=+72000
  +0.1 MiB   server.py:61   count=+50

  You look at line 103 of server.py:

  103:  active_sessions[session_id] = Session(user, data)

  active_sessions is a module-level dict.

  a) What type of memory bug is this? What single question
     do you ask about active_sessions?

  b) The server has had 72,000 sessions since restart but
     only 50 concurrent users right now. What does this tell
     you about the lifecycle of entries in active_sessions?

  c) Write the one line of code you would add to the session
     cleanup handler to stop this leak.

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) Unbounded accumulation. The dict grows without bound.
     Question: "Is there code that REMOVES entries from
     active_sessions when a session expires or a user logs out?"
     If no such code exists, every session ever created since
     server startup is still in the dict.

  b) Entries are added when sessions are created but never
     removed when sessions end. 72,000 sessions have been
     created; all 72,000 are still in the dict despite only
     50 users being active now. The dict is a graveyard of
     dead sessions consuming memory indefinitely.

  c) In the session cleanup / logout / expiry handler:
     active_sessions.pop(session_id, None)
     pop() removes the entry if it exists, does nothing if
     it has already been removed. Memory usage will now
     plateau near (peak concurrent sessions × session size)
     rather than growing with every new session ever created.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4.8 Concurrency Bug Patterns

**Section 2.5 covered async tracebacks. Tracebacks tell you where a function crashed. They say nothing about race conditions — bugs where two concurrent operations interact in an order you did not expect, producing wrong results without any crash or error message.**

This section gives you the vocabulary to recognize these bugs when they appear. Full concurrent debugging is advanced and will deepen when threading and Celery workers are introduced later in the course.

**The core problem: shared mutable state read and written by more than one concurrent actor**

```python
# Run this. Then run it again. Count how often you get 500,000.
import threading

counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1   # NOT atomic — three operations disguised as one

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Expected: 500,000")
print(f"Actual:   {counter}")
```

You will see a different number on every run. Sometimes 347,821. Sometimes 423,554. Sometimes 289,103. Never 500,000.

**Why:** `counter += 1` is not one atomic operation. It is three:

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY counter += 1 IS NOT THREAD-SAFE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: READ  the current value of counter   (e.g., 42)       │
│  Step 2: ADD   1 to the read value             (43)             │
│  Step 3: WRITE the result back to counter      (counter = 43)   │
│                                                                 │
│                                                                 │
│  THREAD A                    THREAD B                           │
│  ────────────────────────    ────────────────────────           │
│  READ counter → 42                                              │
│                              READ counter → 42  ← same value   │
│  ADD  → 43                                                      │
│  WRITE counter = 43                                             │
│                              ADD  → 43                          │
│                              WRITE counter = 43  ← overwrites! │
│                                                                 │
│  Both threads read 42. Both write 43.                           │
│  Two increments occurred. The counter advanced by 1, not 2.    │
│  One increment was silently lost.                               │
│                                                                 │
│  This is a RACE CONDITION: the output depends on which          │
│  thread executes which step first — and that ordering is        │
│  determined by the OS scheduler at runtime.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Fix: mutual exclusion with `threading.Lock`:**

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:        # Only one thread executes this block at a time
            counter += 1  # Read → Add → Write is now atomic

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # Always 500,000
```

`Lock` ensures the three-step operation is never interrupted. Only one thread holds the lock at a time. The others wait.

**Race conditions in async code: they look different but work the same way**

Python's `asyncio` is single-threaded, which eliminates threading race conditions. But async race conditions exist at every `await` point — anywhere the event loop can switch to a different coroutine:

```python
next_order_id = 1000  # Shared module-level state

async def create_order(item: str) -> dict:
    global next_order_id
    order_id = next_order_id          # Read
    await save_to_database(order_id)  # ← Yield: another task may run here
    next_order_id += 1                # Write — but another task already read
    return {"order_id": order_id, "item": item}
```

Two concurrent `create_order()` calls can both read `next_order_id = 1000` before either increments it — producing two orders with the same ID, with no error raised.

```
┌─────────────────────────────────────────────────────────────────┐
│              RECOGNIZING A RACE CONDITION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The output is NONDETERMINISTIC                                 │
│  ─────────────────────────────                                  │
│  Same input, different results on each run.                     │
│  Intermittent. Difficult or impossible to reproduce reliably.   │
│                                                                 │
│  It involves CONCURRENCY                                        │
│  ────────────────────────                                       │
│  Threads, async tasks, multiple processes, or worker queues     │
│  that share any mutable state.                                  │
│                                                                 │
│  It involves SHARED MUTABLE STATE                               │
│  ──────────────────────────────                                 │
│  A global variable, a module-level dict or list, a database     │
│  row, or a file — anything more than one actor can modify.      │
│                                                                 │
│  It DISAPPEARS when you slow down                               │
│  ──────────────────────────────────                             │
│  Adding time.sleep() or a print() statement makes it vanish —  │
│  because slowing one actor changes the timing and moves the     │
│  race outside the danger window.                                │
│  This is the heisenbug symptom (Section 4.9).                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Three concurrency bug types — vocabulary:**

```
┌─────────────────────────────────────────────────────────────────┐
│             CONCURRENCY BUG VOCABULARY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RACE CONDITION                                                 │
│  ──────────────                                                 │
│  Two actors read and write the same state in an unexpected      │
│  order. Result is nondeterministic. No crash, no error — just   │
│  wrong and inconsistent output.                                 │
│  Fix: Lock (threads), atomic DB operations (async), or          │
│       eliminate shared mutable state entirely.                  │
│                                                                 │
│  DEADLOCK                                                       │
│  ────────                                                       │
│  Task A holds Lock 1 and waits for Lock 2.                      │
│  Task B holds Lock 2 and waits for Lock 1.                      │
│  Both wait forever. Program freezes completely.                 │
│  Symptom: program hangs, zero output, zero CPU usage.           │
│  Fix: always acquire multiple locks in the same order.          │
│                                                                 │
│  STARVATION                                                     │
│  ──────────                                                     │
│  One task perpetually wins the scheduler while others wait.     │
│  Symptom: one worker processes all items; others sit idle.      │
│  Fix: fair task scheduling; bounded work per task per turn.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The debugging tool for concurrency bugs is logging (Section 3.8), not the interactive debugger. Attaching a breakpoint to concurrent code changes the timing — which changes whether the race occurs. The bug vanishes under observation. Add timestamped debug logs, run under load, and read the log after the fact. That is the only reliable way to capture a race condition in the act."

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRACTICE CHECKPOINT 11 — Identify a Concurrency Bug
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  A FastAPI endpoint occasionally assigns duplicate invoice IDs
  in production but always works correctly in tests:

  next_invoice_id = 5000   # module-level

  async def create_invoice(amount: float) -> dict:
      global next_invoice_id
      invoice_id = next_invoice_id          # Read
      await save_invoice(invoice_id, amount)  # Yield point
      next_invoice_id += 1                  # Write
      return {"invoice_id": invoice_id}

  a) Is this a race condition, deadlock, or starvation?

  b) At which exact line does the dangerous window open,
     and what can happen inside that window?

  c) Why does this bug appear in production but not in tests?
     What condition is necessary to trigger it?

───────────────────────────────────────────────────────────────────
▼ SOLUTION — attempt it yourself first
───────────────────────────────────────────────────────────────────

  a) Race condition. Two async tasks can both read next_invoice_id
     before either increments it, producing two invoices with
     the same ID and no error raised anywhere.

  b) await save_invoice(...) — this yields control back to the
     event loop. While Task A is suspended waiting for the DB
     write to complete, the event loop can start Task B.
     Task B reads next_invoice_id (still 5000 — A hasn't
     incremented yet). Both tasks use invoice_id = 5000.
     The dangerous window is any code between the Read and the
     Write that contains an await.

  c) Tests send requests one at a time. Task A completes fully
     before Task B starts. There is no overlap, so the race
     window is never open simultaneously for two tasks.
     Production receives concurrent requests. Multiple tasks
     genuinely interleave at the await point. Two or more tasks
     can occupy the Read-to-Write window simultaneously.
     The fix: use a database sequence (auto-increment primary key)
     to generate IDs atomically at the DB level, not in
     application code shared across concurrent tasks.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4.9 Intermittent Bugs and Heisenbugs

**The hardest bugs to debug are the ones that are not consistently there.**

A **heisenbug** (named after Heisenberg's uncertainty principle) is a bug that changes behavior when you try to observe it. You add a `print()` and the bug disappears. You attach the debugger and the program runs correctly. You remove your tools and the bug returns. Almost every heisenbug has timing or concurrency at its root: your observation tool changes the timing, which changes whether the race condition or timing window is triggered.

```
┌─────────────────────────────────────────────────────────────────┐
│               STRATEGIES FOR INTERMITTENT BUGS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. USE LOGGING, NOT THE INTERACTIVE DEBUGGER                   │
│  ─────────────────────────────────────────────                  │
│  Breakpoints pause execution and change timing. The bug         │
│  that depends on timing will disappear.                         │
│  Logging adds minimal overhead and does not pause the program.  │
│  Add DEBUG-level logs with timestamps at every suspected        │
│  boundary and let the program run at full speed.                │
│                                                                 │
│  logger.debug("[%f] task=%s reading state=%s",                  │
│               time.perf_counter(), task_id, state)              │
│                                                                 │
│  If two tasks log the same state value with overlapping         │
│  timestamps, you have found the race.                           │
│                                                                 │
│                                                                 │
│  2. REPRODUCE IT AT SCALE                                       │
│  ────────────────────────                                       │
│  If the bug appears once per 1,000 requests, run 10,000.        │
│  Higher volume = more concurrent overlaps = more reliable       │
│  reproduction. locust (Week 12) is the right tool for this.     │
│  A simple loop suffices for local testing:                      │
│  for _ in range(10_000): await create_invoice(99.99)            │
│                                                                 │
│                                                                 │
│  3. BINARY SEARCH THE VARIABLES                                 │
│  ─────────────────────────────                                  │
│  Is the bug environment-specific? (production but not staging?) │
│  Load-specific? (only under 50+ concurrent requests?)           │
│  Data-specific? (only with a specific user ID range?)           │
│  Eliminate one variable at a time until the bug becomes         │
│  consistently reproducible in a controlled environment.         │
│                                                                 │
│                                                                 │
│  4. ACCEPT EXTENDED OBSERVATION AS A VALID STRATEGY             │
│  ──────────────────────────────────────────────────             │
│  If you cannot reproduce the bug reliably, instrument the       │
│  code with comprehensive DEBUG logging, deploy the change,      │
│  and wait for the bug to occur in production. Read the logs     │
│  after it happens. This is uncomfortable. It is sometimes       │
│  the only option for bugs that require specific production       │
│  state to trigger.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The key insight: if adding observability makes the bug disappear, the bug is timing-dependent and your observation tool is changing the timing. Stop trying to observe it interactively. Log everything non-interactively. The bug will eventually tell you what happened — but only if you were recording when it did."

---

## 4.10 Environment Debugging

**"It works on my machine" is not a dead end. It is a diagnostic clue: something in the environment differs between the two machines.**

When code that works locally fails elsewhere — a colleague's laptop, a CI server, a staging environment — the failure is almost always one of five things. Check them in this order:

```
┌─────────────────────────────────────────────────────────────────┐
│              ENVIRONMENT DEBUGGING CHECKLIST                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CHECK 1: Python version                                        │
│  ────────────────────────                                       │
│  Local:   python --version  →  Python 3.12.3                    │
│  Remote:  python --version  →  Python 3.10.1  ← mismatch!      │
│                                                                 │
│  Or in code: import sys; print(sys.version_info)                │
│                                                                 │
│  Watch for features added after the remote version:             │
│  match/case (3.10+), tomllib (3.11+), ExceptionGroup (3.11+),   │
│  improved f-strings (3.12+). These produce SyntaxError or       │
│  ImportError on older interpreters.                             │
│                                                                 │
│  Fix: pin Python in pyproject.toml:                             │
│  requires-python = ">=3.12"                                     │
│  And: uv python pin 3.12  (sets .python-version)               │
│                                                                 │
│                                                                 │
│  CHECK 2: Installed package versions                            │
│  ────────────────────────────────────                           │
│  Local:   uv pip list | grep httpx   →  httpx 0.27.0            │
│  Remote:  uv pip list | grep httpx   →  httpx 0.24.1  ← old!   │
│                                                                 │
│  uv.lock prevents this: both environments running uv sync       │
│  from the same committed lockfile will have identical packages.  │
│  If the remote environment used pip install without a lockfile,  │
│  versions will drift.                                           │
│                                                                 │
│  Fix: ensure CI and the remote environment run: uv sync         │
│                                                                 │
│                                                                 │
│  CHECK 3: Environment variables                                 │
│  ──────────────────────────────                                 │
│  import os; print(os.environ.get("DATABASE_URL", "NOT SET"))    │
│                                                                 │
│  Missing: DATABASE_URL → connection error or KeyError           │
│  Wrong:   DATABASE_URL points to localhost instead of prod DB   │
│                                                                 │
│  Common missing variables: DATABASE_URL, SECRET_KEY, API_KEY,   │
│  any variable your app reads with os.environ[].                 │
│  pydantic-settings (Week 15) validates all required variables   │
│  are present at startup — before the first request arrives.     │
│                                                                 │
│                                                                 │
│  CHECK 4: File paths and working directory                      │
│  ──────────────────────────────────────────                     │
│  import os; print(os.getcwd())                                  │
│                                                                 │
│  Hardcoded paths break across machines and operating systems:   │
│  "/Users/alice/project/data"  →  works on Alice's Mac only      │
│  "data/grades.csv"            →  works only if CWD is correct   │
│                                                                 │
│  Fix: use pathlib.Path(__file__).parent / "data" / "grades.csv" │
│  This resolves relative to the script's location, not the CWD. │
│                                                                 │
│                                                                 │
│  CHECK 5: Active virtual environment                            │
│  ────────────────────────────────────                           │
│  uv run python -c "import sys; print(sys.executable)"           │
│  Should resolve to .venv/bin/python inside your project.        │
│  If it shows /usr/bin/python3 — you are using the system        │
│  Python, which does not have your project's packages.           │
│  All imports will fail with ModuleNotFoundError.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fastest protocol for "works locally, fails in CI":**

Add this temporarily to the top of your entry point. Run locally. Run in CI. Compare the four values. The difference is your bug.

```python
import sys, os

print(f"Python:    {sys.version}")
print(f"Exe:       {sys.executable}")
print(f"DB URL:    {os.environ.get('DATABASE_URL', 'NOT SET')}")
print(f"CWD:       {os.getcwd()}")
```

> "Committing `uv.lock` and running `uv sync` in CI is the single most powerful environment debugging tool available to you. A project that does both has near-identical environments everywhere. 'It works on my machine but not in CI' becomes: either a missing environment variable, or a Python version mismatch, or someone ran pip instead of uv. This checklist covers all three."

---

# PART 5: READING OTHERS' CODE

## 5.1 You Already Did It

The 5-layer reading strategy from the very start of this lecture was not new content — it was the framework you applied when you first opened `buggy_grades.py`. You identified its structure (one file), its data model (`Student` dataclass with name and scores), its entry point (`main()` calling four pipeline stages), its function signatures (each stage's input and output type), and traced one execution path from raw dict to grade report entry. That was layer-reading. You have been doing it all lecture. Now you have a name for it — use it every time you open unfamiliar code.

> "Tests are also one of the best entry points into an unfamiliar codebase. If there's a `tests/` folder, read the tests before the source. Tests show you how the code is *meant* to be used — what inputs it expects, what outputs it produces. Tests are living documentation. You'll write your own in the next lecture."

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                  DEBUGGING QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHEN YOUR CODE CRASHES:                                        │
│  1. Read the LAST LINE of the traceback (the error type)        │
│  2. Read the BOTTOM FRAME in YOUR code (the crash site)         │
│  3. Skip library frames — find the first frame in your files    │
│  4. Read upward to trace the call chain if needed               │
│                                                                 │
│  IDE DEBUGGER (VS Code / PyCharm):                              │
│  Breakpoint:      Click the gutter (left margin)                │
│  Start debugging: F5                                            │
│  Step Over:       F10 (execute line, don't enter function)      │
│  Step Into:       F11 (enter the function — bug hypothesis here) │
│  Step Out:        Shift+F11 (finish function, return to caller) │
│  Continue:        F5 (run to next breakpoint)                   │
│  Conditional:     Right-click breakpoint → Edit → add condition │
│  Call Stack:      Click any frame → jump there + see its vars   │
│                                                                 │
│  STEP DECISION RULE:                                            │
│  "Bug is inside this function" → Step Into (F11)               │
│  "This function is probably fine" → Step Over (F10)            │
│  "This function is confirmed fine" → Step Out (Shift+F11)      │
│                                                                 │
│  pdb (TERMINAL — 6 CORE COMMANDS):                              │
│  Start:           Add breakpoint() in your code                 │
│  p <expr>         Print expression value     (= Watch pane)     │
│  n                Step over (next line)      (= F10)            │
│  s                Step into (enter function) (= F11)            │
│  c                Continue (to next breakpoint) (= F5)          │
│  l                List source around current line               │
│  q                Quit debugger                                  │
│  Full reference:  See pdb Reference Card handout                │
│                                                                 │
│  LOGGING:                                                       │
│  import logging; logging.basicConfig(level=logging.DEBUG)       │
│  logger = logging.getLogger(__name__)                           │
│  logger.debug / .info / .warning / .error / .critical           │
│  Production: set level=logging.WARNING to silence debug noise   │
│                                                                 │
│  THREE-TOOL MODEL:                                              │
│  Print     = scratch pad (fast, local, disposable)              │
│  Debugger  = magnifying glass (interactive, real-time, local)   │
│  Logging   = permanent record (level-filtered, remote-safe)     │
│                                                                 │
│  SYSTEMATIC METHODS:                                            │
│  Hypothesis-driven: Observe → Hypothesize → Predict → Test      │
│  Binary search:     Check the middle, eliminate half, repeat    │
│  Rubber duck:       Narrate every line out loud — assumptions   │
│                     become visible when you have to say them    │
│                                                                 │
│  POST-MORTEM DEBUGGING:                                         │
│  Run under pdb:    python -m pdb script.py → c → crashes → pdb │
│  Wrap a call:      try: f() except: pdb.post_mortem()           │
│  After REPL crash: import pdb; pdb.pm()                         │
│  Navigate frames:  u (up to caller)    d (down to callee)       │
│                                                                 │
│  ATTACH MODE (debugpy):                                         │
│  In code:  debugpy.listen(5678); debugpy.wait_for_client()      │
│  launch.json: "request": "attach", "connect": {port: 5678}      │
│  Remove both debugpy lines before committing.                   │
│                                                                 │
│  ASSERTIONS:                                                    │
│  assert <condition>, "message if False"                         │
│  Precondition: at function entry — validate inputs              │
│  Postcondition: before return — validate outputs                │
│  Invariant: mid-loop or mid-pipeline — validate assumptions     │
│  Not for: external input, production error handling             │
│                                                                 │
│  PERFORMANCE DEBUGGING:                                         │
│  Quick:    time.perf_counter()  → wall-clock elapsed time       │
│  Profiler: cProfile.Profile() as pr → pstats.Stats(pr)          │
│            Sort by CUMULATIVE. High tottime = the bottleneck.   │
│  Benchmark: timeit.timeit(stmt, setup, number)                  │
│  Workflow: observe → profile → read → benchmark → confirm       │
│                                                                 │
│  MEMORY DEBUGGING:                                              │
│  tracemalloc.start()                                            │
│  s1 = tracemalloc.take_snapshot()    (baseline)                 │
│  ... run code ...                                               │
│  s2 = tracemalloc.take_snapshot()    (after)                    │
│  s2.compare_to(s1, 'lineno')  → shows what grew                 │
│  gc.set_debug(gc.DEBUG_LEAK) + gc.collect() → find cycles       │
│  lru_cache(maxsize=N) → bounded cache with automatic eviction   │
│                                                                 │
│  CONCURRENCY BUGS:                                              │
│  Race condition: nondeterministic output + shared mutable state  │
│  Deadlock: program hangs, zero output, zero CPU                 │
│  Debug with: logging + timestamps (not interactive debugger)    │
│  Threading fix: threading.Lock() + with lock: block             │
│  Async fix: atomic DB operations; never share state across await│
│                                                                 │
│  ENVIRONMENT CHECKLIST (works here, fails there):               │
│  1. python --version           (version match?)                 │
│  2. uv pip list | grep <name>  (package versions match?)        │
│  3. os.environ.get("KEY")      (all env vars present?)          │
│  4. os.getcwd()                (working directory correct?)     │
│  5. sys.executable             (right virtual environment?)     │
│  Prevention: commit uv.lock; run uv sync in CI                  │
│                                                                 │
│  RECURSIVE TRACEBACKS:                                          │
│  RecursionError → missing or unreachable base case              │
│  Find the repeating frame. Skip the 994 identical copies.       │
│  Ask: "What condition stops this call for this input?"          │
│  Never raise sys.setrecursionlimit() to silence the error.      │
│                                                                 │
│  ASKING FOR HELP:                                               │
│  State: Goal → Expected → Actual → Tried → Minimal code → Env  │
│  Build the minimal reproducible example first — it often        │
│  reveals the bug before you need to share it.                   │
│  State your real goal, not just the approach that failed.       │
│                                                                 │
│  BEFORE ALL ELSE:                                               │
│  Read. The. Error. Message.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  DEBUGGING = CLOSING THE GAP                                    │
│                                                                 │
│  What you THINK            What ACTUALLY                        │
│  your code does            happens                              │
│  ──────────────            ────────────                         │
│       │                         │                               │
│       │     ┌───────────┐       │                               │
│       └────▶│    GAP    │◀──────┘                               │
│             │  (= bug)  │                                       │
│             └───────────┘                                       │
│                   │                                             │
│           Tools to close it:                                    │
│           ├─ Stack traces (read the evidence)                   │
│           ├─ IDE debugger (see reality in real time)            │
│           ├─ pdb (see reality, anywhere, no GUI needed)         │
│           ├─ Logging (permanent record, remote-safe)            │
│           ├─ Hypothesis testing (look with purpose)             │
│           ├─ Binary search (find it fast in long pipelines)     │
│           └─ Rubber duck (surface hidden assumptions)           │
│                                                                 │
│                                                                 │
│  THE DETECTIVE ANALOGY (completed):                             │
│  ├─ Traceback = Crime scene report (read it first!)             │
│  ├─ Error type = Witness description of the crime               │
│  ├─ Call stack = Trail of footprints leading to the body        │
│  ├─ Breakpoint = Surveillance camera (pause and observe)        │
│  ├─ Step Into/Over/Out = Following / watching / leaving         │
│  ├─ Conditional breakpoint = "Alert me only if..."              │
│  ├─ Hypothesis = "I think the operator is wrong. Let me test."  │
│  ├─ Binary search = Eliminating suspects by halves              │
│  ├─ Rubber duck = Explaining your theory out loud               │
│  └─ Logging = The security system that records everything       │
│                                                                 │
│  THE ONE RULE:                                                  │
│  Read the error message before you do anything else.            │
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
│  THIS WEEK (Lecture 2 — Testing):                               │
│  └─ Tests CATCH bugs before they reach users.                   │
│     Debugging finds bugs after they appear.                     │
│     Testing prevents them from appearing.                       │
│     You'll also debug FAILING TESTS — same skills, new context. │
│     You'll set up the "Debug Tests" launch config from 3.6.     │
│                                                                 │
│     PREVIEW — debugging a failing test:                         │
│     When a test fails, the traceback ends with an AssertionError│
│     on the assertion line inside the test file. Apply the       │
│     bottom-up technique: read the AssertionError message first, │
│     then find the test frame.                                   │
│     To inspect interactively: set a breakpoint at the first     │
│     line of the test function and step through. The Variables   │
│     pane shows your fixtures, the object under test, and every  │
│     local at the moment the assertion fails.                    │
│     To enter post-mortem after any test failure: run            │
│     pytest --pdb — it drops into pdb.pm() automatically at      │
│     the crash frame of every failing test.                      │
│                                                                 │
│  THIS WEEK (Mini-Project):                                      │
│  └─ Your async CLI tool WILL have bugs. Guaranteed.             │
│     Use the debugger. Form hypotheses. Read tracebacks.         │
│     Add logging.debug() calls at key async boundaries.          │
│     This is your first real practice ground.                    │
│                                                                 │
│  WEEK 3 (FastAPI):                                              │
│  └─ API bugs produce HTTP 500 errors with tracebacks in the     │
│     server log. Same skill — read bottom-up, find the frame     │
│     in YOUR code, ignore framework internals (same rule as 2.5).│
│     You'll set up the "Debug FastAPI" launch config from 3.6.   │
│                                                                 │
│  WEEK 6 (SQLAlchemy):                                           │
│  └─ ORM errors produce LONG tracebacks full of SQLAlchemy       │
│     internals. Same technique: skip library frames,             │
│     find YOUR code, read the error. This is 2.5's rule          │
│     applied to a new library.                                   │
│                                                                 │
│  WEEK 15 (Docker & CI/CD):                                      │
│  └─ No IDE debugger inside a container. pdb becomes essential.  │
│     breakpoint() is your best friend on remote servers.         │
│     structlog extends the logging model from 3.8 with           │
│     structured JSON output and correlation IDs.                 │
│     ruff breakpoint-detection rules get explicitly enabled      │
│     in pyproject.toml — that's when the linting safety net      │
│     for leftover breakpoint() calls becomes active.             │
│                                                                 │
│  EVERY WEEK, EVERY PROJECT:                                     │
│  └─ Debugging is not a topic. It's a daily practice.            │
│     These skills compound. By Week 8, reading a traceback       │
│     will be as natural as reading English.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
