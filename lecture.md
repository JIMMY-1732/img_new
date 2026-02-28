# GITHUB COPILOT CHAT CUSTOMIZATION & EXTENSIBILITY
## Mastering Custom Agents, Prompt Files, Skills, Instructions & Rules, Hooks, MCP Servers, Tool Sets, Diagnostics, and Chat Settings

---

## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MENU FIRST, BUTTONS LAST                                       │
│  ────────────────────────                                       │
│  Students must understand WHY these features exist before       │
│  learning HOW to configure them. We start with the pain of     │
│  repeating yourself to an AI, then build toward automation.     │
│                                                                 │
│  LAYERED LEARNING                                               │
│  ────────────────                                               │
│  Each feature builds on the previous one. Prompt Files are      │
│  passive instructions. Custom Agents combine prompt files       │
│  with tools. MCP Servers extend everything further.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  We use a "hiring an assistant" analogy throughout.             │
│  Every Copilot customization maps to onboarding a new hire.     │
│                                                                 │
│  HANDS-ON IMMEDIATELY                                           │
│  ────────────────────                                           │
│  Every concept comes with a concrete file or setting you can    │
│  create right now in VS Code. No theory without practice.       │
│                                                                 │
│  FROM CONFUSED TO CONFIDENT                                     │
│  ──────────────────────────                                     │
│  The lecture is designed as if we know NOTHING about these       │
│  menu items, discover what each does, and by the end can        │
│  build a fully customized Copilot workflow from scratch.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│          GITHUB COPILOT CHAT CUSTOMIZATION                      │
│          (Comprehensive Deep-Dive Lecture)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE BIG PICTURE (Overview & Motivation)                │
│  ├─ 1.1 The Problem: Why Default Copilot Isn't Enough           │
│  ├─ 1.2 The Settings Menu: What Are All These Items?            │
│  ├─ 1.3 The Hiring Analogy: Onboarding Your AI Assistant        │
│  └─ 1.4 How the Features Relate to Each Other                   │
│                                                                 │
│  PART 2: PROMPT FILES (The Instruction Manuals)                 │
│  ├─ 2.1 What Are Prompt Files?                                  │
│  ├─ 2.2 Creating Your First Prompt File                         │
│  ├─ 2.3 The .github/prompts Directory                           │
│  ├─ 2.4 Prompt File Syntax and Variables                        │
│  ├─ 2.5 Reusable Prompt Patterns                                │
│  └─ 2.6 Prompt Files vs Chat Messages                           │
│                                                                 │
│  PART 3: INSTRUCTIONS & RULES (The Company Policy)              │
│  ├─ 3.1 What Are Instructions & Rules?                          │
│  ├─ 3.2 .github/copilot-instructions.md                         │
│  ├─ 3.3 VS Code Settings-Based Instructions                     │
│  ├─ 3.4 Instruction Scoping: Global, Workspace, Folder          │
│  ├─ 3.5 Writing Effective Rules                                 │
│  └─ 3.6 Instructions vs Prompt Files: When to Use Which         │
│                                                                 │
│  PART 4: CUSTOM AGENTS (The Specialist Employees)               │
│  ├─ 4.1 What Are Custom Agents?                                 │
│  ├─ 4.2 The chat-participant API Concept                        │
│  ├─ 4.3 Built-In Agents: @workspace, @terminal, @vscode         │
│  ├─ 4.4 Creating a Custom Agent (chat participant)              │
│  ├─ 4.5 Agent Configuration with Prompt Files                   │
│  ├─ 4.6 Agent Tools and Capabilities                            │
│  └─ 4.7 Real-World Agent Examples                               │
│                                                                 │
│  PART 5: SKILLS (The Toolbox)                                   │
│  ├─ 5.1 What Are Skills?                                        │
│  ├─ 5.2 Built-In Skills Overview                                │
│  ├─ 5.3 Enabling and Disabling Skills                           │
│  ├─ 5.4 How Skills Interact with Agents                         │
│  └─ 5.5 Skill Selection: Automatic vs Manual                    │
│                                                                 │
│  PART 6: HOOKS (The Automated Triggers)                         │
│  ├─ 6.1 What Are Copilot Hooks?                                 │
│  ├─ 6.2 Hook Events: When Do They Fire?                         │
│  ├─ 6.3 Configuring Hooks in settings.json                      │
│  ├─ 6.4 Hook Commands and Scripts                               │
│  ├─ 6.5 Practical Hook Examples                                 │
│  └─ 6.6 Security Considerations                                 │
│                                                                 │
│  PART 7: MCP SERVERS (The External Specialists)                 │
│  ├─ 7.1 What Is MCP (Model Context Protocol)?                   │
│  ├─ 7.2 Why MCP Matters                                         │
│  ├─ 7.3 How MCP Servers Work                                    │
│  ├─ 7.4 Configuring MCP Servers in VS Code                      │
│  ├─ 7.5 Available MCP Servers                                   │
│  ├─ 7.6 Building Your Own MCP Server (Conceptual)               │
│  └─ 7.7 MCP vs Extensions vs Skills                             │
│                                                                 │
│  PART 8: TOOL SETS (The Equipment Assignments)                  │
│  ├─ 8.1 What Are Tool Sets?                                     │
│  ├─ 8.2 Defining Tool Sets                                      │
│  ├─ 8.3 Assigning Tools to Agents                               │
│  └─ 8.4 Tool Permissions and Safety                             │
│                                                                 │
│  PART 9: DIAGNOSTICS (The Health Check)                         │
│  ├─ 9.1 What Does Diagnostics Show?                             │
│  ├─ 9.2 Troubleshooting Common Issues                           │
│  ├─ 9.3 Checking Authentication & Entitlements                  │
│  └─ 9.4 Reading Diagnostic Logs                                 │
│                                                                 │
│  PART 10: CHAT SETTINGS (The Control Panel)                     │
│  ├─ 10.1 Overview of Chat Settings                              │
│  ├─ 10.2 Model Selection                                        │
│  ├─ 10.3 Context Window and Token Limits                        │
│  ├─ 10.4 Code Generation Preferences                            │
│  └─ 10.5 Privacy and Data Settings                              │
│                                                                 │
│  PART 11: PUTTING IT ALL TOGETHER                               │
│  ├─ 11.1 Architecture: How All Features Interact                │
│  ├─ 11.2 Building a Complete Custom Workflow                     │
│  ├─ 11.3 Team Sharing: Version-Controlled AI Configuration      │
│  └─ 11.4 Decision Framework: What to Use When                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE BIG PICTURE

## 1.1 The Problem: Why Default Copilot Isn't Enough

**Start with the frustration. Make it relatable.**

Imagine you open VS Code with GitHub Copilot Chat and ask it to generate a React component. It gives you a class component. But your team uses functional components with hooks. You correct it. Next day, you ask again — same mistake. You correct it again. And again. And again.

Or imagine you're working on a Python FastAPI project. Every time you ask Copilot to write a route handler, you have to say: "Use Pydantic V2 model syntax. Use async. Follow our naming conventions. Put the file in `src/api/routes/`." Every. Single. Time.

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE REPETITION PROBLEM                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Day 1:  "Write a component. Oh, use functional style please."  │
│  Day 2:  "Write a component. Use functional style. Also hooks." │
│  Day 3:  "Write a component. Functional. Hooks. TypeScript."    │
│  Day 4:  "Write a component. Functional. Hooks. TypeScript.     │
│           Also use our custom Button from @/components."        │
│  Day 5:  😤 "WHY DON'T YOU JUST KNOW THIS ALREADY?!"           │
│                                                                 │
│  You're re-onboarding your AI assistant EVERY conversation.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The core pain:**

> "GitHub Copilot is a brilliant intern who gets amnesia every time you start a new chat. The features we're about to learn are how you cure that amnesia — permanently."

Every one of the settings menu items we see in Copilot's configuration dropdown exists to solve a specific dimension of this problem:

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE SOLUTIONS MAP                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM                            SOLUTION                    │
│  ───────                            ────────                    │
│  "It forgets my coding style"    →  Instructions & Rules        │
│  "I type the same prompts daily" →  Prompt Files                │
│  "I need domain-specific help"   →  Custom Agents               │
│  "It doesn't know my tools"     →  Skills & Tool Sets           │
│  "It can't access my database"   →  MCP Servers                 │
│  "I want auto-linting after gen" →  Hooks                       │
│  "Something isn't working"       →  Diagnostics                 │
│  "I want to change the model"    →  Chat Settings               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 The Settings Menu: What Are All These Items?

**Let's look at what's in the dropdown and categorize them:**

When you click the settings gear icon (⚙️) in the GitHub Copilot Chat panel in VS Code, you see a menu with these items:

```
┌─────────────────────────────────────────────────────────────────┐
│              THE COPILOT SETTINGS MENU                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─ CONTENT CUSTOMIZATION ──────────────────────────────────┐   │
│  │                                                          │   │
│  │  Custom Agents    = Specialist personas with tools       │   │
│  │  Prompt Files     = Reusable prompt templates            │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─ BEHAVIOR CONFIGURATION ─────────────────────────────────┐   │
│  │                                                          │   │
│  │  Skills           = Capabilities Copilot can use         │   │
│  │  Instructions     = Persistent coding rules/guidelines   │   │
│  │    & Rules                                               │   │
│  │  Hooks            = Auto-run commands on events          │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─ EXTERNAL INTEGRATIONS ──────────────────────────────────┐   │
│  │                                                          │   │
│  │  MCP Servers      = External tool servers (databases,    │   │
│  │                     APIs, custom tools)                   │   │
│  │  Tool Sets        = Grouped tool configurations          │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─ MAINTENANCE & PREFERENCES ──────────────────────────────┐   │
│  │                                                          │   │
│  │  Diagnostics      = Health check & troubleshooting       │   │
│  │  Chat Settings    = Model selection, preferences         │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Hiring Analogy: Onboarding Your AI Assistant

**This analogy will carry us through the entire lecture.**

Think of GitHub Copilot as a new employee you just hired. They're incredibly smart and know a lot about programming in general — but they know nothing about YOUR company, YOUR codebase, YOUR conventions, or YOUR tools.

```
┌─────────────────────────────────────────────────────────────────┐
│               THE HIRING ANALOGY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ONBOARDING STEP               │  COPILOT EQUIVALENT            │
│  ──────────────                │  ──────────────────            │
│                                │                                │
│  Company handbook with rules   │  Instructions & Rules          │
│  "We use tabs, not spaces.     │  .github/copilot-instructions  │
│   Always write tests."         │                                │
│                                │                                │
│  Template documents            │  Prompt Files                  │
│  "Here's our standard format   │  .github/prompts/*.md          │
│   for proposals and reports."  │                                │
│                                │                                │
│  Specialist roles              │  Custom Agents                 │
│  "Sarah does frontend,         │  @frontend, @backend,          │
│   Bob does database work."     │  @database agents              │
│                                │                                │
│  Job skills & training         │  Skills                        │
│  "You can use our CI/CD        │  Code search, terminal,        │
│   pipeline and deploy tools."  │  web fetch capabilities        │
│                                │                                │
│  Workflow automations          │  Hooks                         │
│  "After every code review,     │  "After code generation,       │
│   run the linter automatically" │  run eslint --fix"            │
│                                │                                │
│  External consultant access    │  MCP Servers                   │
│  "Call this outside firm       │  Connect to database,          │
│   for legal/tax questions."    │  Jira, Figma, etc.             │
│                                │                                │
│  Equipment assignments         │  Tool Sets                     │
│  "Interns get basic tools,     │  Group and assign tools        │
│   seniors get full access."    │  to specific agents            │
│                                │                                │
│  IT health check               │  Diagnostics                   │
│  "Is your laptop working?      │  Auth, connection,             │
│   VPN connected?"              │  entitlement checks            │
│                                │                                │
│  Personal preferences          │  Chat Settings                 │
│  "I prefer dark mode and       │  Model choice, context         │
│   standing desk."              │  window preferences            │
│                                │                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 How the Features Relate to Each Other

**Understanding the hierarchy is critical before diving into details:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FEATURE DEPENDENCY MAP                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌──────────────┐                              │
│                    │ Chat Settings│ ← Foundation layer           │
│                    │ (model, etc) │   (what brain does it have?) │
│                    └──────┬───────┘                              │
│                           │                                     │
│              ┌────────────┼────────────┐                        │
│              │            │            │                        │
│              ▼            ▼            ▼                        │
│     ┌────────────┐ ┌───────────┐ ┌──────────┐                  │
│     │Instructions│ │  Skills   │ │   MCP    │                  │
│     │  & Rules   │ │(built-in  │ │ Servers  │                  │
│     │(how to     │ │ abilities)│ │(external │                  │
│     │ behave)    │ │           │ │ tools)   │                  │
│     └─────┬──────┘ └─────┬─────┘ └────┬─────┘                  │
│           │              │            │                         │
│           └──────────┬───┴────────────┘                         │
│                      │                                          │
│              ┌───────┴────────┐                                  │
│              │   Tool Sets    │ ← Groups tools together         │
│              └───────┬────────┘                                  │
│                      │                                          │
│           ┌──────────┼──────────┐                               │
│           │          │          │                               │
│           ▼          ▼          ▼                               │
│    ┌───────────┐ ┌────────┐ ┌──────┐                            │
│    │ Prompt    │ │ Custom │ │Hooks │                            │
│    │ Files     │ │ Agents │ │(auto │                            │
│    │(templates)│ │(roles) │ │ run) │                            │
│    └───────────┘ └────────┘ └──────┘                            │
│                                                                 │
│    ┌──────────┐                                                 │
│    │Diagnostics│ ← Observability for all of the above           │
│    └──────────┘                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "Each feature addresses a different layer of customization. Instructions tell Copilot HOW to behave. Skills tell it WHAT it can do. MCP Servers give it access to EXTERNAL systems. Custom Agents package all of these into reusable personas. And Hooks automate what happens AFTER Copilot acts."

---

# PART 2: PROMPT FILES (The Instruction Manuals)

## 2.1 What Are Prompt Files?

**Prompt Files are reusable, saved prompts stored as Markdown files.**

Instead of typing the same complex prompt every time you want Copilot to do something specific, you save it as a `.prompt.md` file. Then you can invoke it from chat with a single click or reference.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROMPT FILES                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Markdown files (*.prompt.md) that contain predefined    │
│         instructions for Copilot Chat.                          │
│                                                                 │
│  WHERE: .github/prompts/ directory in your workspace            │
│                                                                 │
│  WHY:   Stop re-typing the same complex prompts.                │
│         Share prompts with your team via version control.        │
│         Create consistent, repeatable AI interactions.          │
│                                                                 │
│  ANALOGY: Template documents at a company.                      │
│           "Use this template for all project proposals."        │
│           Staff don't reinvent the format each time.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Without prompt files (the pain):**

```
You (Monday):
  "Generate a React component using TypeScript, functional style,
   with CSS Modules, default export, JSDoc, Props interface..."

You (Tuesday):
  "Generate a React component using TypeScript... ugh, let me
   find what I typed yesterday and copy-paste it..."

You (Wednesday):
  "I forgot the part about CSS modules. The generated component
   doesn't match our conventions again."
```

**With prompt files (the solution):**

```
You (any day):
  /react-component

Copilot automatically reads .github/prompts/react-component.prompt.md
and follows ALL your instructions perfectly, every time.
```

---

## 2.2 Creating Your First Prompt File

**Step-by-step: Create a prompt file right now.**

**Step 1:** Enable the prompt files feature in VS Code settings:

```json
// In your VS Code settings.json
{
    "chat.promptFiles": true
}
```

> **Note:** As of VS Code 1.99+, prompt files are enabled by default. In earlier versions or Insiders, you may need to enable `chat.promptFiles` or `github.copilot.chat.promptFiles` explicitly.

**Step 2:** Create the directory structure:

```bash
mkdir -p .github/prompts
```

**Step 3:** Create your first prompt file:

```markdown
<!-- filepath: .github/prompts/react-component.prompt.md -->
# React Component Generator

Generate a React component with the following requirements:

## Technical Requirements
- Use **TypeScript** (.tsx files)
- Use **functional components** with hooks (no class components)
- Use **CSS Modules** for styling (ComponentName.module.css)
- Export the component as a **default export**

## Code Style
- Add **JSDoc comments** above the component
- Define **Props interface** above the component function
- Use **destructured props** in the function signature
- Keep components **under 100 lines** — extract sub-components if needed

## Imports
- Import shared UI components from `@/components/ui`
- Import hooks from `@/hooks`
- Import types from `@/types`
```

**Step 4:** Use it in Copilot Chat:

In the chat input, you can reference this prompt file or access it from the prompt files menu. Type `/` to see available prompt commands, or attach the prompt file as context.

---

## 2.3 The .github/prompts Directory

**Project layout and conventions:**

```
your-project/
├── .github/
│   ├── prompts/                      ← All prompt files live here
│   │   ├── react-component.prompt.md ← Component generation
│   │   ├── api-endpoint.prompt.md    ← API route generation
│   │   ├── unit-test.prompt.md       ← Test generation template
│   │   ├── code-review.prompt.md     ← Review checklist
│   │   ├── refactor.prompt.md        ← Refactoring guidance
│   │   └── database-migration.prompt.md
│   │
│   ├── copilot-instructions.md       ← Global instructions (Part 3)
│   └── workflows/                    ← GitHub Actions (unrelated)
│
├── src/
├── tests/
└── package.json
```

**Why `.github/prompts/`?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY THIS DIRECTORY?                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. VERSION CONTROLLED                                          │
│     └─ Prompt files are just Markdown in your repo              │
│        Team members pull them with `git pull`                   │
│        Changes are reviewed in PRs like any code                │
│                                                                 │
│  2. WORKSPACE-SCOPED                                            │
│     └─ Different projects can have different prompts            │
│        A React project has react-component.prompt.md            │
│        A Python project has fastapi-endpoint.prompt.md          │
│                                                                 │
│  3. DISCOVERABLE                                                │
│     └─ VS Code auto-discovers files matching *.prompt.md        │
│        in the .github/prompts/ directory                        │
│                                                                 │
│  4. TEAM-SHAREABLE                                              │
│     └─ Commit to repo → entire team gets same prompt files      │
│        Ensures consistent AI-assisted development               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Prompt File Syntax and Variables

**Prompt files support special syntax for dynamic content:**

```markdown
<!-- filepath: .github/prompts/api-endpoint.prompt.md -->
# API Endpoint Generator

Create a new API endpoint with the following context:

## Variables
- The current file is: #{file}
- The selected code is: #{selection}
- The workspace is: #{workspace}

## Requirements
- Use the project's existing patterns found in #{file:src/api/routes/users.ts}
- Follow the error handling pattern from #{file:src/utils/errors.ts}
- Add the endpoint to the router in #{file:src/api/index.ts}

## Standards
- All routes must be async
- Use Zod for request validation
- Return standardized JSON responses
- Include OpenAPI/Swagger documentation comments
```

**Variable reference table:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 PROMPT FILE VARIABLES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VARIABLE                │  RESOLVES TO                         │
│  ────────                │  ───────────                         │
│                          │                                      │
│  #{file}                 │  Content of the currently active     │
│                          │  editor file                         │
│                          │                                      │
│  #{file:path/to/file}    │  Content of a specific file in       │
│                          │  the workspace                       │
│                          │                                      │
│  #{selection}            │  Currently selected text in the      │
│                          │  active editor                       │
│                          │                                      │
│  #{workspace}            │  Information about the current       │
│                          │  workspace structure                 │
│                          │                                      │
│  #{<tool-name>}          │  Output from a specific tool         │
│                          │  (advanced usage)                    │
│                          │                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Why variables matter:**

> "Variables make prompt files DYNAMIC. Without them, a prompt file is just static text. With them, the prompt automatically pulls in relevant context — the file you're editing, code you've selected, or specific reference files from your project."

---

## 2.5 Reusable Prompt Patterns

**Here are production-tested prompt file patterns:**

**Pattern 1: Code Review Checklist**

```markdown
<!-- filepath: .github/prompts/code-review.prompt.md -->
# Code Review

Review the provided code for the following concerns:

## Security
- [ ] No hardcoded secrets or API keys
- [ ] Input validation on all user inputs
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (output encoding)

## Performance
- [ ] No N+1 query patterns
- [ ] Appropriate use of caching
- [ ] No unnecessary re-renders (React) or recomputations
- [ ] Database indexes exist for queried fields

## Code Quality
- [ ] Functions are under 30 lines
- [ ] No code duplication
- [ ] Descriptive variable and function names
- [ ] Error handling is comprehensive

## Testing
- [ ] New code has corresponding tests
- [ ] Edge cases are covered
- [ ] Test descriptions are clear

Provide specific line-by-line feedback. For each issue found, suggest a fix.
```

**Pattern 2: Database Migration Generator**

```markdown
<!-- filepath: .github/prompts/db-migration.prompt.md -->
# Database Migration Generator

Generate a database migration based on the described changes.

## Context
- ORM: Prisma / SQLAlchemy / TypeORM (adjust based on your project)
- Database: PostgreSQL
- Reference the current schema: #{file:prisma/schema.prisma}

## Requirements
- Migration must be reversible (include both up and down)
- Add appropriate indexes for foreign keys
- Include data migration if schema change affects existing rows
- Use transactions for safety
- Add comments explaining WHY the migration is needed, not just WHAT

## Safety Checklist
- Will this migration lock any tables? If so, note it.
- Is this backward-compatible with the current app version?
- Estimated data migration time for production dataset?
```

**Pattern 3: Bug Investigation**

```markdown
<!-- filepath: .github/prompts/debug.prompt.md -->
# Bug Investigation

Help me investigate and fix a bug.

## Process
1. **Reproduce**: Understand the steps to reproduce from my description
2. **Hypothesize**: List 3-5 most likely root causes
3. **Narrow Down**: Suggest specific files and lines to investigate
4. **Fix**: Propose a fix with explanation
5. **Prevent**: Suggest how to prevent this class of bug in the future

## Current Context
- Active file: #{file}
- Selected code (if any): #{selection}

## Guidelines
- Don't assume — ask clarifying questions if needed
- Consider race conditions and edge cases
- Check for recent changes that might have caused regression
- Suggest a test case that would catch this bug
```

---

## 2.6 Prompt Files vs Chat Messages

**Understanding when to use each:**

```
┌─────────────────────────────────────────────────────────────────┐
│           PROMPT FILES VS CHAT MESSAGES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROMPT FILES                        CHAT MESSAGES              │
│  ────────────                        ─────────────              │
│  Persistent, reusable               One-off, ephemeral         │
│  Version controlled                 Lost when chat clears      │
│  Team-shareable                     Personal only               │
│  Structured templates               Free-form questions        │
│  Complex multi-step tasks           Quick simple queries        │
│                                                                 │
│  USE FOR:                            USE FOR:                   │
│  • Repeated workflows               • Quick questions           │
│  • Team-standard processes           • One-time tasks            │
│  • Complex generation tasks          • Exploration              │
│  • Onboarding new developers         • "How does X work?"       │
│  • Enforcing consistency             • Ad-hoc debugging         │
│                                                                 │
│  EXAMPLE:                            EXAMPLE:                   │
│  "Generate a new API endpoint        "What does this regex      │
│   following our 15-step process       do?"                      │
│   with validation, tests, docs,                                 │
│   and router registration."                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: INSTRUCTIONS & RULES (The Company Policy)

## 3.1 What Are Instructions & Rules?

**Instructions are persistent, always-on rules that Copilot follows in EVERY interaction.**

Unlike prompt files (which you invoke for specific tasks), instructions are applied automatically to every chat message and every code completion. They're your "company policy" — the rules that are always in effect.

```
┌─────────────────────────────────────────────────────────────────┐
│                  INSTRUCTIONS & RULES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Persistent directives that shape how Copilot responds   │
│         to every single request — chat, completions, edits.     │
│                                                                 │
│  WHERE: Two places:                                             │
│         1. File: .github/copilot-instructions.md (per-repo)     │
│         2. VS Code settings.json (per-user or per-workspace)    │
│                                                                 │
│  WHY:   You don't want to repeat "use TypeScript" every chat.   │
│         You want Copilot to ALWAYS know your coding standards.  │
│                                                                 │
│  ANALOGY: The company employee handbook.                        │
│           "Every employee must follow these rules, always.      │
│            You don't re-read the handbook per task — you just   │
│            know it applies to everything you do."               │
│                                                                 │
│  KEY DIFFERENCE FROM PROMPT FILES:                              │
│  ├─ Prompt Files = invoked manually for specific tasks          │
│  └─ Instructions = applied automatically to ALL interactions    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 .github/copilot-instructions.md

**This is the primary file for repository-level instructions.**

When Copilot Chat starts, it automatically reads `.github/copilot-instructions.md` from your workspace root and includes its content as context in every conversation.

**Create it now:**

```bash
mkdir -p .github
touch .github/copilot-instructions.md
```

**Example: A well-crafted instructions file for a TypeScript project:**

```markdown
<!-- filepath: .github/copilot-instructions.md -->
# Copilot Instructions for MyProject

## Language & Framework
- This is a **TypeScript** project using **React 18** and **Next.js 14** (App Router).
- Backend API routes use **Next.js Route Handlers** (not Pages API routes).
- Database access uses **Prisma ORM** with **PostgreSQL**.

## Code Style
- Use **functional components** exclusively (no class components).
- Use **named exports** for components (not default exports).
- Use **arrow functions** for component definitions:
  `export const MyComponent = () => { ... }`
- Prefer **const** over let. Never use var.
- Use **template literals** instead of string concatenation.
- Maximum line length: **100 characters**.

## File Organization
- Components: `src/components/{feature}/{ComponentName}.tsx`
- Hooks: `src/hooks/use{HookName}.ts`
- Types: `src/types/{domain}.ts`
- API routes: `src/app/api/{resource}/route.ts`
- Utilities: `src/lib/{utilName}.ts`

## Testing
- Write tests using **Vitest** and **React Testing Library**.
- Test files: `__tests__/{ComponentName}.test.tsx`
- Every new component, hook, and utility MUST have tests.
- Prefer **integration tests** over unit tests for components.

## Error Handling
- Use custom error classes from `src/lib/errors.ts`.
- API routes must return proper HTTP status codes.
- Never swallow errors silently — always log or re-throw.

## Naming Conventions
- Components: PascalCase (`UserProfile`)
- Hooks: camelCase with `use` prefix (`useAuth`)
- Types/Interfaces: PascalCase with descriptive suffix (`UserProfileProps`)
- API routes: kebab-case directories (`/api/user-profiles/`)
- Database tables: snake_case (`user_profiles`)

## Comments
- Write JSDoc comments for all exported functions and components.
- Use `// TODO:` for planned improvements (never ship `// HACK:`).
- Comments explain WHY, not WHAT. The code explains what.

## Security
- Never store secrets in code. Use environment variables.
- Always validate and sanitize user input on the server side.
- Use parameterized queries (Prisma handles this).
```

**What happens when this file exists:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HOW INSTRUCTIONS ARE APPLIED                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT .github/copilot-instructions.md:                       │
│                                                                 │
│  You: "Create a user profile component"                         │
│  Copilot: (uses its default assumptions — might use class       │
│           components, default exports, wrong file path, etc.)   │
│                                                                 │
│                                                                 │
│  WITH .github/copilot-instructions.md:                          │
│                                                                 │
│  You: "Create a user profile component"                         │
│  Copilot: (reads instructions automatically)                    │
│    → Uses TypeScript                                            │
│    → Uses functional arrow component                            │
│    → Uses named export                                          │
│    → Places in src/components/users/UserProfile.tsx              │
│    → Creates UserProfileProps type                               │
│    → Adds JSDoc comments                                        │
│    → Creates test file in __tests__/                             │
│                                                                 │
│  Same prompt. Completely different (better) output.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 VS Code Settings-Based Instructions

**You can also define instructions directly in VS Code settings.**

This is useful for personal preferences that aren't project-specific, or for rules that shouldn't be committed to version control.

**In your settings.json:**

```json
{
    "github.copilot.chat.codeGeneration.instructions": [
        {
            "text": "Always add error handling with try-catch blocks."
        },
        {
            "text": "Prefer async/await over .then() chains."
        },
        {
            "text": "Add TypeScript strict mode compatible code."
        },
        {
            "file": ".github/copilot-instructions.md"
        }
    ]
}
```

**You can also point to external instruction files:**

```json
{
    "github.copilot.chat.codeGeneration.instructions": [
        { "file": "docs/coding-standards.md" },
        { "file": ".github/copilot-instructions.md" }
    ]
}
```

**Settings vs file-based instructions:**

```
┌─────────────────────────────────────────────────────────────────┐
│       SETTINGS vs FILE-BASED INSTRUCTIONS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VS CODE SETTINGS                    FILE-BASED                 │
│  ────────────────                    ──────────                 │
│  In settings.json                    In .github/copilot-        │
│                                        instructions.md          │
│                                                                 │
│  Can be user-level (global)          Always workspace-level     │
│  Can be workspace-level              Version controlled ✅      │
│  Not version controlled (if user)    Team-shareable ✅          │
│  Supports "file" references          Richer Markdown format     │
│  Good for personal preferences       Good for team standards    │
│                                                                 │
│  RECOMMENDATION:                                                │
│  • Project standards → .github/copilot-instructions.md          │
│  • Personal style → User-level settings.json                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Instruction Scoping: Global, Workspace, Folder

**Instructions can apply at different levels:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 INSTRUCTION SCOPING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCOPE           │ WHERE                │ APPLIES TO             │
│  ─────           │ ─────                │ ──────────             │
│                  │                      │                        │
│  GLOBAL (User)   │ User settings.json   │ Every workspace,       │
│                  │ (Ctrl+Shift+P →      │ every project,         │
│                  │  Preferences: Open   │ everywhere.            │
│                  │  User Settings)      │                        │
│                  │                      │                        │
│  WORKSPACE       │ .vscode/             │ Only this project      │
│                  │   settings.json      │                        │
│                  │ OR                   │                        │
│                  │ .github/copilot-     │                        │
│                  │   instructions.md    │                        │
│                  │                      │                        │
│  FOLDER          │ .vscode/ in a        │ Only files in that     │
│  (Multi-root)    │   specific folder    │ folder of a multi-     │
│                  │   of a multi-root    │ root workspace         │
│                  │   workspace          │                        │
│                  │                      │                        │
│                                                                 │
│  PRECEDENCE (most specific wins):                               │
│  Folder > Workspace > Global (User)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multi-root workspace example:**

```
my-workspace/
├── frontend/                    ← React project
│   ├── .vscode/settings.json    ← "Use React, TypeScript, CSS Modules"
│   └── .github/copilot-instructions.md
│
├── backend/                     ← Python FastAPI project
│   ├── .vscode/settings.json    ← "Use Python, FastAPI, SQLAlchemy"
│   └── .github/copilot-instructions.md
│
└── shared/                      ← Shared types/configs
    └── .vscode/settings.json
```

> "When you open a file in `frontend/`, Copilot reads the frontend instructions. Switch to a file in `backend/`, and Copilot automatically switches to Python rules. It's like having department-specific handbooks."

---

## 3.5 Writing Effective Rules

**The art of writing instructions Copilot actually follows:**

```
┌─────────────────────────────────────────────────────────────────┐
│             EFFECTIVE INSTRUCTIONS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ GOOD:  Specific, actionable, unambiguous                    │
│                                                                 │
│  "Use arrow functions for React components:                     │
│   export const MyComponent = () => { ... }"                     │
│                                                                 │
│  "Place test files in __tests__/ directory with .test.tsx       │
│   extension"                                                    │
│                                                                 │
│  "Import database utilities from @/lib/db — never use raw      │
│   SQL strings"                                                  │
│                                                                 │
│  ───────────────────────────────────────────────────────────    │
│                                                                 │
│  ❌ BAD:  Vague, contradictory, or too philosophical            │
│                                                                 │
│  "Write clean code"  ← What does "clean" mean?                  │
│                                                                 │
│  "Be consistent"     ← Consistent with what?                    │
│                                                                 │
│  "Follow best practices"  ← WHOSE best practices?              │
│                                                                 │
│  "Don't write bad code"   ← Not actionable                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Rules for writing rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│           META-RULES FOR INSTRUCTION WRITING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. BE SPECIFIC: Give exact patterns, file paths, and examples  │
│     ✅ "Use Zod for validation. Define schemas in              │
│         src/schemas/{resource}.schema.ts"                       │
│     ❌ "Use proper validation"                                  │
│                                                                 │
│  2. SHOW EXAMPLES: Code examples > prose descriptions           │
│     ✅ "Format error responses as:                              │
│         { error: { code: string, message: string } }"           │
│     ❌ "Return well-structured error responses"                 │
│                                                                 │
│  3. STATE BOTH DO AND DON'T:                                    │
│     ✅ "Use named exports. Do NOT use default exports."         │
│     ❌ "Use named exports." (doesn't prevent default exports)   │
│                                                                 │
│  4. KEEP IT CONCISE: ~50-100 lines maximum                      │
│     Long instructions dilute the important rules               │
│     Copilot has a limited context window                        │
│                                                                 │
│  5. ORGANIZE BY CATEGORY:                                       │
│     Group rules: Style, Architecture, Testing, Security         │
│     Makes it easier to maintain and update                      │
│                                                                 │
│  6. UPDATE REGULARLY:                                           │
│     Review instructions when you notice Copilot consistently    │
│     generating unwanted patterns                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Instructions vs Prompt Files: When to Use Which

```
┌─────────────────────────────────────────────────────────────────┐
│         INSTRUCTIONS VS PROMPT FILES DECISION MATRIX            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE INSTRUCTIONS WHEN:          USE PROMPT FILES WHEN:         │
│  ──────────────────────          ──────────────────────         │
│                                                                 │
│  Rule applies to ALL tasks       Task is specific & repeatable  │
│  "Always use TypeScript"         "Generate a React component"   │
│                                                                 │
│  It's about coding STYLE         It's about a specific WORKFLOW │
│  "Use arrow functions"           "Code review with 4 steps"     │
│                                                                 │
│  You never want exceptions       You choose when to invoke it   │
│  "Never use var"                 "Run migration generator"      │
│                                                                 │
│  It's SHORT (1-3 lines)          It's LONG (multi-section)      │
│  "Prefer async/await"            Full generation template with  │
│                                  examples and checklists        │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  THEY WORK TOGETHER:                                            │
│                                                                 │
│  Instructions: "Always use TypeScript"                          │
│  Prompt File:  "Generate React component using the project's    │
│                 TypeScript configuration..."                    │
│                                                                 │
│  The instruction ensures TypeScript everywhere.                 │
│  The prompt file adds React-specific generation details.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: CUSTOM AGENTS (The Specialist Employees)

## 4.1 What Are Custom Agents?

**Custom Agents are specialized AI personas you create for domain-specific tasks.**

Think of built-in Copilot as a generalist developer. Custom agents are like adding specialists to your team — a frontend expert, a database administrator, a security auditor — each with their own knowledge, instructions, and tools.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUSTOM AGENTS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Named AI personas with specific instructions, tools,    │
│         and knowledge scopes. Invoked with @agentname in chat.  │
│                                                                 │
│  WHERE: Defined as .prompt.md files in .github/prompts/ with    │
│         special YAML front matter, OR as VS Code extensions     │
│         using the Chat Participant API.                         │
│                                                                 │
│  WHY:   Different tasks need different expertise, tools, and    │
│         context. A database agent shouldn't have the same       │
│         capabilities as a frontend design agent.                │
│                                                                 │
│  ANALOGY: Specialist employees.                                 │
│           You don't ask the accountant to design the UI.        │
│           You don't ask the designer to optimize SQL queries.   │
│           Each specialist has their own tools and knowledge.    │
│                                                                 │
│  TWO TYPES:                                                     │
│  ├─ Built-in agents (come with Copilot)                         │
│  └─ Custom agents (you create them)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 The Chat Participant API Concept

**Understanding the architecture behind agents:**

Every agent in Copilot Chat is technically a "chat participant." When you type `@workspace` or `@terminal`, you're addressing a specific participant that knows how to handle certain types of queries.

```
┌─────────────────────────────────────────────────────────────────┐
│                CHAT PARTICIPANT ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOU TYPE IN CHAT:                                              │
│  ─────────────────                                              │
│  "@workspace how is authentication implemented?"                │
│        │                                                        │
│        ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Copilot Chat │ ── Sees @workspace prefix                     │
│  │   Router     │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼ Routes to workspace participant                       │
│  ┌──────────────────┐                                           │
│  │ @workspace Agent  │                                          │
│  │                   │                                          │
│  │ CAPABILITIES:     │                                          │
│  │ • Search all files│                                          │
│  │ • Read workspace  │                                          │
│  │   structure       │                                          │
│  │ • Understand code │                                          │
│  │   relationships   │                                          │
│  └──────┬────────────┘                                          │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────┐                                           │
│  │ Response with     │                                          │
│  │ workspace context │                                          │
│  └──────────────────┘                                           │
│                                                                 │
│  Without @workspace, Copilot only sees the active file.         │
│  With @workspace, it searches your entire project.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight about participants:**

> "Each participant is like a specialist with a specific set of tools. The @workspace agent can search files. The @terminal agent can run commands. The @vscode agent knows about editor features. Custom agents let you create your OWN specialists with your OWN tools and instructions."

---

## 4.3 Built-In Agents: @workspace, @terminal, @vscode

**Copilot comes with several built-in agents you should know:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUILT-IN AGENTS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @workspace                                                     │
│  ──────────                                                     │
│  WHAT: Searches and understands your entire codebase            │
│  USE:  "How is auth implemented?" "Find all API routes"         │
│  TOOLS: File search, symbol search, file reading                │
│  EXAMPLE:                                                       │
│    @workspace where is the database connection configured?      │
│                                                                 │
│  @terminal                                                      │
│  ─────────                                                      │
│  WHAT: Helps with terminal commands and shell operations        │
│  USE:  "How do I find large files?" "Fix this error"            │
│  TOOLS: Command suggestion, error explanation                   │
│  EXAMPLE:                                                       │
│    @terminal how do I find all .env files recursively?          │
│                                                                 │
│  @vscode                                                        │
│  ───────                                                        │
│  WHAT: Helps with VS Code features, settings, keybindings       │
│  USE:  "How to change theme?" "Set up debugging?"               │
│  TOOLS: Settings reference, keybinding lookup                   │
│  EXAMPLE:                                                       │
│    @vscode how do I enable word wrap for markdown files?        │
│                                                                 │
│  @github (if GitHub Copilot Enterprise/Business)                │
│  ───────                                                        │
│  WHAT: Searches across GitHub repositories, issues, PRs         │
│  USE:  "Find similar issues" "Search org repos"                 │
│  TOOLS: GitHub API search, issue/PR reading                     │
│  EXAMPLE:                                                       │
│    @github find issues related to memory leaks                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to use which built-in agent:**

```
┌─────────────────────────────────────────────────────────────────┐
│            BUILT-IN AGENT DECISION TREE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "I have a question about..."                                   │
│               │                                                 │
│               ▼                                                 │
│  ┌────────────────────────┐                                     │
│  │ My project's code?     │──YES──▶ @workspace                  │
│  └────────────┬───────────┘                                     │
│               │ NO                                              │
│               ▼                                                 │
│  ┌────────────────────────┐                                     │
│  │ Terminal / shell?      │──YES──▶ @terminal                   │
│  └────────────┬───────────┘                                     │
│               │ NO                                              │
│               ▼                                                 │
│  ┌────────────────────────┐                                     │
│  │ VS Code features?      │──YES──▶ @vscode                    │
│  └────────────┬───────────┘                                     │
│               │ NO                                              │
│               ▼                                                 │
│  ┌────────────────────────┐                                     │
│  │ GitHub repos/issues?   │──YES──▶ @github                    │
│  └────────────┬───────────┘                                     │
│               │ NO                                              │
│               ▼                                                 │
│       Just type normally                                        │
│       (general Copilot Chat)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Creating a Custom Agent (Chat Participant)

**There are two main approaches to creating custom agents:**

### Approach 1: Prompt-File-Based Agents (Simple — No Code)

As of VS Code 1.96+, you can create lightweight custom agents using `.prompt.md` files in your `.github/prompts/` directory with special YAML front matter that defines the agent's name, description, and tools.

```markdown
<!-- filepath: .github/prompts/frontend-expert.prompt.md -->
---
mode: agent
description: "Frontend specialist for React/TypeScript development"
tools: ["codebase", "githubRepo", "fetch"]
---

# Frontend Development Expert

You are a senior frontend developer specializing in React and TypeScript.

## Your expertise:
- React 18 patterns (hooks, suspense, server components)
- TypeScript strict mode with proper generic patterns
- CSS Modules and Tailwind CSS
- Accessibility (WCAG 2.1 AA compliance)
- Performance optimization (Core Web Vitals)

## When answering:
1. Always consider accessibility implications
2. Suggest performance-optimized patterns
3. Reference existing components in the codebase via @workspace
4. Follow the project's established patterns from #{file:.github/copilot-instructions.md}

## When generating components:
- Include ARIA attributes for interactive elements
- Add loading and error states
- Include unit test alongside the component
- Use the project's existing design tokens
```

### Approach 2: Extension-Based Agents (Advanced — Requires Code)

For full control, you can build a VS Code extension that registers a chat participant using the Chat Participant API:

```typescript
// This is extension code — you'd build this as a VS Code extension
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
    // Register a custom chat participant
    const agent = vscode.chat.createChatParticipant(
        'mycompany.database-expert',  // Unique ID
        async (request, context, response, token) => {
            // request.prompt contains the user's message
            
            // Add system instructions
            const systemPrompt = `You are a PostgreSQL and Prisma ORM expert.
            Always suggest migrations that are backward-compatible.
            Always include indexes for foreign keys.
            Reference the current schema when possible.`;
            
            // Use the language model API to generate a response
            const messages = [
                vscode.LanguageModelChatMessage.User(systemPrompt),
                vscode.LanguageModelChatMessage.User(request.prompt)
            ];
            
            const chatModel = await vscode.lm.selectChatModels({
                vendor: 'copilot',
                family: 'gpt-4o'
            });
            
            if (chatModel.length > 0) {
                const chatResponse = await chatModel[0].sendRequest(
                    messages, {}, token
                );
                for await (const chunk of chatResponse.text) {
                    response.markdown(chunk);
                }
            }
        }
    );
    
    // Set metadata
    agent.iconPath = new vscode.ThemeIcon('database');
    agent.description = 'Database & Prisma ORM expert';
    
    context.subscriptions.push(agent);
}
```

**Comparison of the two approaches:**

```
┌─────────────────────────────────────────────────────────────────┐
│       PROMPT-BASED VS EXTENSION-BASED AGENTS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROMPT-FILE AGENTS              EXTENSION AGENTS               │
│  ──────────────────              ────────────────               │
│  Just a .prompt.md file          Full VS Code extension         │
│  Zero code required              TypeScript/JavaScript code     │
│  Easy to share (commit to repo)  Needs publishing or sideload  │
│  Limited tool access             Full API access                │
│  Community agents possible       Full lifecycle control         │
│                                                                 │
│  BEST FOR:                       BEST FOR:                      │
│  • Team-specific workflows       • Complex integrations         │
│  • Simple specialized roles      • Custom UI elements           │
│  • Quick iteration               • Third-party API access       │
│  • Sharing via git               • Production distributions     │
│                                                                 │
│  RECOMMENDATION:                                                │
│  Start with prompt-file agents. Only build extensions when      │
│  you need capabilities that prompt files can't provide.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Agent Configuration with Prompt Files

**Deep-dive into the YAML front matter that powers prompt-file agents:**

```markdown
<!-- filepath: .github/prompts/security-auditor.prompt.md -->
---
mode: agent
description: "Security-focused code reviewer"
tools: ["codebase", "terminal", "fetch"]
---

# Security Auditor Agent

You are a senior application security engineer performing code review.

## Your Methodology (OWASP Top 10 Focused):
1. **Injection**: Check for SQL injection, command injection, XSS
2. **Broken Auth**: Review authentication and session management
3. **Data Exposure**: Find hardcoded secrets, log leaks, unencrypted data
4. **XXE**: Check XML parsing configurations
5. **Broken Access Control**: Verify authorization on all endpoints
6. **Misconfig**: Check security headers, CORS, error handling
7. **XSS**: Review output encoding and CSP headers
8. **Deserialization**: Check for unsafe deserialization
9. **Components**: Flag outdated dependencies with known CVEs
10. **Logging**: Verify security events are properly logged

## Output Format:
For each finding, report:
- **Severity**: Critical / High / Medium / Low / Informational
- **Location**: File and line number
- **Description**: What the vulnerability is
- **Impact**: What an attacker could do
- **Remediation**: Specific code fix
- **Reference**: OWASP/CWE identifier

## Tools Usage:
- Use `codebase` to search for patterns like hardcoded passwords
- Use `terminal` to check dependency versions against CVE databases
- Use `fetch` to look up latest security advisories
```

**The front matter fields explained:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AGENT FRONT MATTER FIELDS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FIELD         │ REQUIRED │ DESCRIPTION                         │
│  ─────         │ ──────── │ ───────────                         │
│                │          │                                     │
│  mode          │   YES    │ Must be "agent" to register as an   │
│                │          │ invocable agent in chat              │
│                │          │                                     │
│  description   │   YES    │ Short text shown in the agent       │
│                │          │ picker UI when user types @          │
│                │          │                                     │
│  tools         │   NO     │ Array of tool names the agent can   │
│                │          │ use: "codebase", "terminal",        │
│                │          │ "fetch", "githubRepo", etc.         │
│                │          │                                     │
│                │          │ If omitted, agent gets default      │
│                │          │ tools only                          │
│                │          │                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 Agent Tools and Capabilities

**What tools can you assign to an agent?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 AVAILABLE AGENT TOOLS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TOOL NAME       │ WHAT IT DOES                                 │
│  ─────────       │ ────────────                                 │
│                  │                                              │
│  codebase        │ Search and read files in the workspace,      │
│                  │ find symbol definitions, references           │
│                  │                                              │
│  terminal        │ Run commands in the integrated terminal,     │
│                  │ read terminal output                          │
│                  │                                              │
│  fetch           │ Make HTTP requests to external URLs          │
│                  │ (read documentation, check APIs)              │
│                  │                                              │
│  githubRepo      │ Search GitHub repositories, read issues,     │
│                  │ pull requests, and discussions                │
│                  │                                              │
│  useDiff         │ View and understand git diffs,               │
│                  │ staged/unstaged changes                       │
│                  │                                              │
│  editFiles       │ Make edits to files in the workspace         │
│                  │ (create, modify, delete)                      │
│                  │                                              │
│  MCP tools       │ Any tools provided by configured MCP         │
│                  │ servers (covered in Part 7)                   │
│                  │                                              │
└─────────────────────────────────────────────────────────────────┘
```

**Why tool assignment matters:**

> "Not every agent should have every tool. A documentation writer doesn't need terminal access. A security auditor doesn't need to edit files. Limiting tools prevents accidents and keeps agents focused on their role."

---

## 4.7 Real-World Agent Examples

**Example 1: API Documentation Generator**

```markdown
<!-- filepath: .github/prompts/api-docs.prompt.md -->
---
mode: agent
description: "Generates and updates API documentation"
tools: ["codebase", "fetch"]
---

# API Documentation Agent

You generate comprehensive API documentation by analyzing route handlers.

## Process:
1. Use `codebase` to find all API route files
2. For each route, extract: method, path, request body, query params, response
3. Generate OpenAPI 3.0 YAML
4. Generate human-readable Markdown docs

## Output format:
For each endpoint:
### `METHOD /path`
- **Description**: What this endpoint does
- **Auth required**: Yes/No (check for auth middleware)
- **Request body**: JSON schema with types
- **Query parameters**: Table of params
- **Response 200**: Success response example
- **Response 4xx/5xx**: Error response examples
- **Example cURL**: Working cURL command
```

**Example 2: Database Migration Specialist**

```markdown
<!-- filepath: .github/prompts/db-specialist.prompt.md -->
---
mode: agent
description: "Database schema design and migration expert"
tools: ["codebase", "terminal"]
---

# Database Specialist Agent

You are a PostgreSQL database administrator and Prisma ORM expert.

## Before suggesting changes:
1. Read the current schema: #{file:prisma/schema.prisma}
2. Understand existing relations and indexes
3. Check for existing migrations in prisma/migrations/

## When designing schema changes:
- Always ensure backward compatibility
- Add indexes for foreign keys and commonly queried fields
- Consider the data migration path for existing records
- Estimate the migration time for production data volume

## Standards:
- Table names: snake_case, plural (user_profiles)
- Column names: snake_case (created_at)
- Always include: id, created_at, updated_at
- Soft deletes: use deleted_at timestamp (not boolean)
- Use enums for fixed option sets

## After generating migration:
- Run `terminal` to execute: npx prisma migrate dev --name <name>
- Verify the migration SQL looks correct
- Generate updated Prisma client
```

**Example 3: Test Writer**

```markdown
<!-- filepath: .github/prompts/test-writer.prompt.md -->
---
mode: agent
description: "Writes comprehensive test suites"
tools: ["codebase"]
---

# Test Writer Agent

You write thorough test suites for TypeScript/React code.

## For every function/component, test:
1. **Happy path**: Expected inputs produce expected outputs
2. **Edge cases**: Empty inputs, null, undefined, boundary values
3. **Error cases**: Invalid inputs, network failures, timeouts
4. **Integration**: Component interactions with child components

## Testing tools:
- Framework: Vitest
- Component testing: React Testing Library
- Mocking: vi.mock() and vi.fn()
- HTTP mocking: MSW (Mock Service Worker)

## Test structure:
```typescript
describe('ComponentName', () => {
  describe('when [condition]', () => {
    it('should [expected behavior]', () => {
      // Arrange
      // Act
      // Assert
    });
  });
});
```

## Rules:
- Test behavior, not implementation
- Don't test library code (React, Prisma — they have their own tests)
- Each test should be independent (no shared mutable state)
- Use descriptive test names that read like documentation
- Aim for >80% branch coverage on critical paths
```

---

# PART 5: SKILLS (The Toolbox)

## 5.1 What Are Skills?

**Skills are specific capabilities that Copilot can use when responding to your queries.**

Think of skills as the tools in a toolbox. A carpenter has a hammer, saw, and drill. Copilot has code search, web fetch, terminal access, and more. The Skills menu lets you see and control which tools are available.

```
┌─────────────────────────────────────────────────────────────────┐
│                      SKILLS                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Individual capabilities that Copilot can invoke during  │
│         a conversation. Each skill performs a specific action    │
│         like searching code, running commands, or fetching URLs. │
│                                                                 │
│  WHERE: Listed in the Skills section of the Copilot settings    │
│         menu. Each can be enabled/disabled independently.       │
│                                                                 │
│  WHY:   Different tasks need different tools. You might want    │
│         Copilot to search your codebase but NOT run terminal    │
│         commands. Skills give you that control.                  │
│                                                                 │
│  ANALOGY: An employee's certifications and tool training.       │
│           "You're certified to use the forklift, the welder,    │
│            and the CNC machine. But your trainee can only       │
│            use the hand tools for now."                         │
│                                                                 │
│  RELATIONSHIP TO AGENTS:                                        │
│  Skills are the ATOMIC tools. Agents COMPOSE skills.            │
│  An agent = instructions + selected skills + persona.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Built-In Skills Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 BUILT-IN SKILLS CATALOG                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SKILL              │ DESCRIPTION              │ INVOKED WHEN   │
│  ─────              │ ───────────              │ ────────────   │
│                     │                          │                │
│  Code Search        │ Searches files and       │ You ask about  │
│  (codebase)         │ symbols across the       │ code in your   │
│                     │ entire workspace         │ project        │
│                     │                          │                │
│  File Reading       │ Reads and understands    │ You reference  │
│                     │ the full content of      │ specific files │
│                     │ specified files           │                │
│                     │                          │                │
│  Terminal           │ Suggests or runs         │ You ask "how   │
│                     │ terminal commands        │ do I..." tasks │
│                     │                          │                │
│  Web Fetch          │ Fetches content from     │ You provide    │
│  (fetch)            │ URLs for context         │ URLs or need   │
│                     │                          │ external docs  │
│                     │                          │                │
│  Code Editing       │ Makes changes to files   │ Agent mode or  │
│  (editFiles)        │ in the workspace         │ explicit edits │
│                     │                          │                │
│  Git Diff           │ Views staged and         │ You ask about  │
│  (useDiff)          │ unstaged changes         │ recent changes │
│                     │                          │                │
│  GitHub Repo        │ Searches GitHub repos,   │ You ask about  │
│  (githubRepo)       │ issues, PRs              │ repo history   │
│                     │                          │                │
│  Notebook           │ Creates and edits        │ Working with   │
│                     │ Jupyter notebooks        │ .ipynb files   │
│                     │                          │                │
│  Think              │ Extended reasoning for   │ Complex multi- │
│                     │ complex problems         │ step problems  │
│                     │                          │                │
│  Vision             │ Analyzes images and      │ You attach     │
│                     │ screenshots              │ images to chat │
│                     │                          │                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Enabling and Disabling Skills

**You can control which skills Copilot uses — here's how and why:**

**Through the Skills menu in Copilot Chat:**

When you click "Skills" in the settings dropdown, you see a list with toggles. Each skill can be turned on or off.

**Through settings.json:**

```json
{
    // Control which tools are available to Copilot
    "github.copilot.chat.agent.tools": {
        "terminal": true,
        "codebase": true,
        "fetch": true,
        "editFiles": true,
        "useDiff": true
    }
}
```

**When to disable skills:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHEN TO DISABLE SKILLS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DISABLE terminal WHEN:                                         │
│  • Working in a production environment                          │
│  • You don't want Copilot running commands automatically        │
│  • Security-sensitive operations                                │
│                                                                 │
│  DISABLE editFiles WHEN:                                        │
│  • You want review-only mode (discussion, not changes)          │
│  • Working on critical files you want to edit manually          │
│                                                                 │
│  DISABLE fetch WHEN:                                            │
│  • Behind a restrictive firewall                                │
│  • Working offline                                              │
│  • Sensitive codebase (don't want content leaving network)      │
│                                                                 │
│  KEEP codebase ALWAYS ENABLED:                                  │
│  • Almost never a reason to disable this                        │
│  • It's what makes @workspace useful                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 How Skills Interact with Agents

**Skills are the building blocks; Agents compose them:**

```
┌─────────────────────────────────────────────────────────────────┐
│           SKILLS + AGENTS = POWERFUL COMBOS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AGENT: @frontend-expert                                        │
│  SKILLS: [codebase, fetch]                                      │
│  WHY: Needs to read code and check React docs                   │
│       Does NOT need terminal or file editing                    │
│                                                                 │
│  AGENT: @devops                                                 │
│  SKILLS: [codebase, terminal, fetch]                            │
│  WHY: Needs to read configs, run kubectl/docker, check docs     │
│       Needs terminal for deployments                            │
│                                                                 │
│  AGENT: @code-reviewer                                          │
│  SKILLS: [codebase, useDiff]                                    │
│  WHY: Needs to read code and see what changed                   │
│       Explicitly NO editFiles — reviewers don't change code     │
│                                                                 │
│  AGENT: @scaffolder                                             │
│  SKILLS: [codebase, editFiles, terminal]                        │
│  WHY: Needs to create files, run generators, set up projects    │
│       Full creative control needed                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Skill Selection: Automatic vs Manual

**By default, Copilot auto-selects skills based on your query. But you can be explicit.**

```
┌─────────────────────────────────────────────────────────────────┐
│          AUTOMATIC VS MANUAL SKILL SELECTION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTOMATIC (default):                                           │
│  ─────────────────────                                          │
│  Copilot reads your question and decides which skills to use.   │
│                                                                 │
│  You: "How is authentication implemented in this project?"      │
│  Copilot auto-activates: codebase (to search your files)        │
│                                                                 │
│  You: "Fetch the latest React docs on Suspense"                 │
│  Copilot auto-activates: fetch (to read URLs)                   │
│                                                                 │
│  MANUAL (when auto fails or you want precision):                │
│  ──────────────────────────────────────────────                 │
│  You can hint or force specific tool usage:                     │
│                                                                 │
│  You: "Search the codebase for all usages of useAuth hook"      │
│  → Explicitly tells Copilot to use code search                  │
│                                                                 │
│  You: "#file:src/auth/login.ts explain this authentication"     │
│  → Explicitly provides a file reference as context              │
│                                                                 │
│  In agent mode, tools listed in the YAML front matter are       │
│  the only ones available — this is explicit tool scoping.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: HOOKS (The Automated Triggers)

## 6.1 What Are Copilot Hooks?

**Hooks are automated commands that run BEFORE or AFTER Copilot takes specific actions.**

Think of hooks as workflow automations. When Copilot generates code, you might want to automatically format it, lint it, or run tests. Hooks let you set this up as an automatic pipeline.

```
┌─────────────────────────────────────────────────────────────────┐
│                      COPILOT HOOKS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Shell commands or scripts that execute automatically    │
│         when certain Copilot events occur.                      │
│                                                                 │
│  WHERE: Configured in VS Code settings.json under               │
│         github.copilot.chat.hooks                               │
│                                                                 │
│  WHY:   Automate quality gates after AI-generated code.         │
│         Ensure generated code always passes your standards      │
│         WITHOUT manual intervention.                            │
│                                                                 │
│  ANALOGY: Company workflow automations.                         │
│           "After someone submits an expense report,             │
│            automatically email the manager for approval."       │
│           The employee doesn't have to remember the process.    │
│                                                                 │
│  THE BRILLIANT PART:                                            │
│  Copilot generates code → Hook auto-runs linter →              │
│  If linter fails → Copilot sees the errors →                   │
│  Copilot auto-fixes the code. A self-correcting loop!          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Hook Events: When Do They Fire?

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOOK EVENT TYPES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVENT                │ FIRES WHEN                              │
│  ─────                │ ──────────                              │
│                       │                                         │
│  postSaveFile         │ After Copilot saves/creates a file      │
│                       │ (agent mode or inline edit)              │
│                       │                                         │
│  postCreateFile       │ After Copilot creates a new file        │
│                       │                                         │
│  postEditFile         │ After Copilot edits an existing file    │
│                       │                                         │
│                                                                 │
│  LIFECYCLE VISUALIZATION:                                       │
│                                                                 │
│   User asks Copilot to create a component                       │
│       │                                                         │
│       ▼                                                         │
│   Copilot generates the code                                    │
│       │                                                         │
│       ▼                                                         │
│   Copilot writes the file to disk                               │
│       │                                                         │
│       ▼                                                         │
│   ┌────────────────────────┐                                    │
│   │ HOOK FIRES HERE        │ ← postSaveFile                    │
│   │ "npm run lint -- {file}"│                                   │
│   └────────────┬───────────┘                                    │
│                │                                                │
│       ┌────────┴────────┐                                       │
│       │                 │                                       │
│    PASS ✅           FAIL ❌                                     │
│       │                 │                                       │
│       ▼                 ▼                                       │
│    Done!           Copilot sees errors                           │
│                    and can auto-fix                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Configuring Hooks in settings.json

**Hooks are defined in your VS Code settings:**

```json
{
    "github.copilot.chat.hooks": {
        "postSaveFile": [
            {
                "command": "npm run lint -- ${file}",
                "description": "Run ESLint on saved file"
            }
        ],
        "postCreateFile": [
            {
                "command": "npx prettier --write ${file}",
                "description": "Format newly created files"
            }
        ]
    }
}
```

**Variables available in hook commands:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HOOK COMMAND VARIABLES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VARIABLE       │ RESOLVES TO                                   │
│  ────────       │ ───────────                                   │
│                 │                                               │
│  ${file}        │ Absolute path of the file that was            │
│                 │ created/saved/edited                           │
│                 │                                               │
│  ${workspaceFolder}                                             │
│                 │ Absolute path of the workspace root            │
│                 │                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Hook Commands and Scripts

**You can run any shell command or script:**

**Simple inline commands:**

```json
{
    "github.copilot.chat.hooks": {
        "postSaveFile": [
            {
                "command": "npx eslint --fix ${file}",
                "description": "Auto-fix ESLint issues"
            },
            {
                "command": "npx prettier --write ${file}",
                "description": "Auto-format with Prettier"
            }
        ]
    }
}
```

**Complex scripts:**

```bash
#!/bin/bash
# filepath: scripts/copilot-post-save.sh
# Run after Copilot saves a file

FILE="$1"
EXTENSION="${FILE##*.}"

echo "Post-save hook running on: $FILE"

# Run language-specific linting
case "$EXTENSION" in
    ts|tsx)
        npx eslint --fix "$FILE"
        npx tsc --noEmit "$FILE" 2>&1
        ;;
    py)
        python -m ruff check --fix "$FILE"
        python -m mypy "$FILE" 2>&1
        ;;
    css|scss)
        npx stylelint --fix "$FILE"
        ;;
esac

# Run prettier on everything
npx prettier --write "$FILE" 2>/dev/null

echo "Post-save hook complete"
```

```json
{
    "github.copilot.chat.hooks": {
        "postSaveFile": [
            {
                "command": "bash scripts/copilot-post-save.sh ${file}",
                "description": "Run comprehensive post-save checks"
            }
        ]
    }
}
```

---

## 6.5 Practical Hook Examples

**Example 1: Auto-format + Lint pipeline**

```json
{
    "github.copilot.chat.hooks": {
        "postSaveFile": [
            {
                "command": "npx prettier --write ${file}",
                "description": "Format code"
            },
            {
                "command": "npx eslint ${file} --max-warnings 0",
                "description": "Check for lint errors"
            }
        ]
    }
}
```

**Example 2: Auto-run tests for generated test files**

```json
{
    "github.copilot.chat.hooks": {
        "postCreateFile": [
            {
                "command": "if echo ${file} | grep -q '.test.'; then npx vitest run ${file}; fi",
                "description": "Run tests for newly created test files"
            }
        ]
    }
}
```

**Example 3: Python type checking**

```json
{
    "github.copilot.chat.hooks": {
        "postSaveFile": [
            {
                "command": "if echo ${file} | grep -q '.py$'; then python -m ruff check --fix ${file} && python -m mypy ${file} --ignore-missing-imports; fi",
                "description": "Ruff + mypy for Python files"
            }
        ]
    }
}
```

---

## 6.6 Security Considerations

```
┌─────────────────────────────────────────────────────────────────┐
│                HOOK SECURITY WARNINGS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⚠️  HOOKS RUN SHELL COMMANDS ON YOUR MACHINE                   │
│                                                                 │
│  RISKS:                                                         │
│  ─────                                                          │
│  1. A malicious repo could include workspace settings with      │
│     hooks that run arbitrary commands when you open it           │
│                                                                 │
│  2. Hook commands have the same permissions as your VS Code     │
│     process (your user account)                                 │
│                                                                 │
│  3. If a hook references ${file} and a filename is              │
│     maliciously crafted, it could enable injection              │
│                                                                 │
│  PROTECTIONS:                                                   │
│  ────────────                                                   │
│  1. VS Code PROMPTS you before running hooks from workspace     │
│     settings (Workspace Trust feature)                          │
│     → Always review before trusting a workspace                 │
│                                                                 │
│  2. Define hooks in USER settings for personal workflows        │
│     (won't be affected by cloned repos)                         │
│                                                                 │
│  3. Keep hook scripts simple and auditable                      │
│     Use well-known tools (eslint, prettier, ruff)               │
│     Avoid curl/wget in hooks (downloading and executing)        │
│                                                                 │
│  4. Review any .vscode/settings.json in PRs that modify hooks   │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  "If you wouldn't run this command manually, don't put          │
│   it in a hook."                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 7: MCP SERVERS (The External Specialists)

## 7.1 What Is MCP (Model Context Protocol)?

**MCP is a standard protocol that lets AI assistants talk to external tools and data sources.**

Imagine Copilot as a brilliant employee trapped inside a glass box. It can see your code (through file access), talk to you (through chat), and read the internet (through fetch). But it CAN'T directly query your database, read your Jira tickets, check your Figma designs, or access your internal documentation wiki.

**MCP breaks open that glass box.**

```
┌─────────────────────────────────────────────────────────────────┐
│              MODEL CONTEXT PROTOCOL (MCP)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  An open standard protocol (created by Anthropic) that   │
│         defines how AI models communicate with external tools   │
│         and data sources through a unified interface.           │
│                                                                 │
│  WHERE: MCP servers are configured in VS Code settings or       │
│         in .vscode/mcp.json in your workspace.                  │
│                                                                 │
│  WHY:   Copilot's built-in skills are limited. MCP lets you     │
│         connect Copilot to ANY external system — databases,     │
│         APIs, design tools, project management, monitoring.     │
│                                                                 │
│  ANALOGY: External consultants on speed-dial.                   │
│           Your employee (Copilot) can now CALL specialists:     │
│           "Hey database consultant, run this query."            │
│           "Hey Jira consultant, what are the open bugs?"        │
│           "Hey Figma consultant, what does this design say?"    │
│                                                                 │
│  THE PROTOCOL:                                                  │
│  MCP defines a standard way for tools to describe:              │
│  • What capabilities they offer (tool listing)                  │
│  • What inputs they need (parameter schemas)                    │
│  • What outputs they return (result formats)                    │
│  • How to communicate (JSON-RPC over stdio or HTTP)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 Why MCP Matters

**Before MCP: Every AI tool needed custom integrations.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM MCP SOLVES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE MCP (The N × M Problem):                                │
│  ─────────────────────────────────                              │
│                                                                 │
│  AI Tools          External Services                            │
│  ────────          ─────────────────                            │
│  Copilot ───────── PostgreSQL                                   │
│  Copilot ───────── MongoDB                                      │
│  Copilot ───────── Jira                                         │
│  Copilot ───────── Figma                                        │
│  Claude  ───────── PostgreSQL  (different integration!)         │
│  Claude  ───────── MongoDB     (different integration!)         │
│  Claude  ───────── Jira        (different integration!)         │
│  Claude  ───────── Figma       (different integration!)         │
│                                                                 │
│  N AI tools × M services = N×M custom integrations 😱           │
│                                                                 │
│                                                                 │
│  AFTER MCP (The N + M Solution):                                │
│  ────────────────────────────────                               │
│                                                                 │
│  AI Tools              MCP               External Services      │
│  ────────          ┌─────────┐           ─────────────────      │
│  Copilot ──────────│         │────────── PostgreSQL MCP         │
│  Claude  ──────────│   MCP   │────────── MongoDB MCP            │
│  ChatGPT ──────────│Protocol │────────── Jira MCP               │
│  Gemini  ──────────│         │────────── Figma MCP              │
│                    └─────────┘                                  │
│                                                                 │
│  N AI tools + M servers = N+M integrations 🎉                   │
│  Any AI tool works with any MCP server.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "MCP is like USB for AI. Before USB, every device had its own proprietary connector. After USB, any device works with any computer. MCP does the same — write one MCP server for PostgreSQL, and EVERY AI tool that speaks MCP can use it."

---

## 7.3 How MCP Servers Work

**The architecture of MCP communication:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 MCP ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────┐         │
│  │                  VS CODE                            │         │
│  │                                                    │         │
│  │  ┌──────────────────┐    ┌───────────────────┐     │         │
│  │  │  Copilot Chat    │    │   MCP Client      │     │         │
│  │  │                  │───▶│   (built into     │     │         │
│  │  │  "Query the      │    │    VS Code)       │     │         │
│  │  │   database for   │    │                   │     │         │
│  │  │   user count"    │    │  Translates       │     │         │
│  │  │                  │    │  Copilot's request │     │         │
│  │  └──────────────────┘    │  into MCP calls   │     │         │
│  │                          └─────────┬─────────┘     │         │
│  │                                    │               │         │
│  └────────────────────────────────────┼───────────────┘         │
│                                       │                         │
│              JSON-RPC over stdio      │  or HTTP/SSE            │
│              (standard protocol)      │                         │
│                                       ▼                         │
│  ┌────────────────────────────────────────────────────┐         │
│  │              MCP SERVER                             │         │
│  │              (External Process)                     │         │
│  │                                                    │         │
│  │  1. Receives tool call request                     │         │
│  │  2. Validates parameters                           │         │
│  │  3. Executes the operation                         │         │
│  │     (query DB, call API, read file, etc.)          │         │
│  │  4. Returns structured result                      │         │
│  │                                                    │         │
│  └────────────────────────────────────┬───────────────┘         │
│                                       │                         │
│                                       ▼                         │
│                              ┌──────────────┐                   │
│                              │  PostgreSQL   │                   │
│                              │  Jira API     │                   │
│                              │  Filesystem   │                   │
│                              │  Any service  │                   │
│                              └──────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The communication flow step by step:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MCP COMMUNICATION FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: DISCOVERY                                              │
│  ─────────────────                                              │
│  VS Code starts the MCP server process.                         │
│  Server reports: "I have these tools available:                  │
│    - query_database(sql: string) → results                      │
│    - list_tables() → table names                                │
│    - describe_table(name: string) → schema"                     │
│                                                                 │
│  STEP 2: USER REQUEST                                           │
│  ────────────────────                                           │
│  User asks in Copilot Chat:                                     │
│  "How many users signed up last month?"                         │
│                                                                 │
│  STEP 3: TOOL SELECTION                                         │
│  ─────────────────────                                          │
│  Copilot sees the available MCP tools and decides:              │
│  "I should use query_database to answer this."                  │
│                                                                 │
│  STEP 4: TOOL CALL                                              │
│  ────────────────                                               │
│  Copilot sends to MCP server:                                   │
│  {                                                              │
│    "method": "tools/call",                                      │
│    "params": {                                                  │
│      "name": "query_database",                                  │
│      "arguments": {                                             │
│        "sql": "SELECT COUNT(*) FROM users                       │
│                WHERE created_at >= '2026-01-01'"                │
│      }                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  STEP 5: EXECUTION & RESPONSE                                   │
│  ────────────────────────────                                   │
│  MCP server runs the query, returns:                            │
│  { "result": [{"count": 1247}] }                                │
│                                                                 │
│  STEP 6: NATURAL LANGUAGE ANSWER                                │
│  ────────────────────────────────                               │
│  Copilot receives the result and tells user:                    │
│  "1,247 users signed up last month (since January 1, 2026)."   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two transport mechanisms:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MCP TRANSPORT TYPES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STDIO (Standard Input/Output)                                  │
│  ──────────────────────────────                                 │
│  • MCP server runs as a LOCAL child process                     │
│  • Communicates via stdin/stdout (JSON-RPC)                     │
│  • Simplest setup — just specify the command to run             │
│  • Best for: Local tools, databases, file-based tools           │
│                                                                 │
│  Example: npx runs a Node.js MCP server locally                 │
│           python runs a Python MCP server locally               │
│                                                                 │
│                                                                 │
│  HTTP + SSE (Server-Sent Events)                                │
│  ────────────────────────────────                               │
│  • MCP server runs as a REMOTE HTTP service                     │
│  • Communicates over HTTP with SSE for streaming                │
│  • Can be shared across a team                                  │
│  • Best for: Shared services, cloud-hosted tools                │
│                                                                 │
│  Example: A centralized MCP server running on your              │
│           company's internal network                            │
│                                                                 │
│                                                                 │
│  WHICH TO USE:                                                  │
│  ┌────────────────────────┬──────────────────────────┐          │
│  │ Local / personal       │ Use stdio                │          │
│  │ Team-shared / cloud    │ Use HTTP + SSE           │          │
│  │ Quick experimentation  │ Use stdio                │          │
│  │ Production deployment  │ Use HTTP + SSE           │          │
│  └────────────────────────┴──────────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.4 Configuring MCP Servers in VS Code

**There are two main places to configure MCP servers:**

### Method 1: Workspace-Level Configuration (.vscode/mcp.json)

This is the **recommended** approach for project-specific MCP servers. It's version-controlled and shared with your team.

```json
// filepath: .vscode/mcp.json
{
    "servers": {
        "my-database": {
            "type": "stdio",
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-postgres",
                "postgresql://localhost:5432/mydb"
            ]
        },
        "filesystem": {
            "type": "stdio",
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "/path/to/allowed/directory"
            ]
        },
        "github-server": {
            "type": "stdio",
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-github"
            ],
            "env": {
                "GITHUB_PERSONAL_ACCESS_TOKEN": "${input:github_pat}"
            }
        }
    },
    "inputs": [
        {
            "id": "github_pat",
            "type": "promptString",
            "description": "GitHub Personal Access Token",
            "password": true
        }
    ]
}
```

### Method 2: User Settings (settings.json)

For personal MCP servers you want available in every workspace:

```json
// In your VS Code User settings.json
{
    "mcp": {
        "servers": {
            "my-personal-notes": {
                "type": "stdio",
                "command": "npx",
                "args": [
                    "-y",
                    "@modelcontextprotocol/server-filesystem",
                    "/home/me/notes"
                ]
            }
        }
    }
}
```

### Method 3: In-Chat Configuration (Quick Testing)

You can type MCP configuration directly in Copilot Chat for fast testing:

```
#mcp {"type":"stdio","command":"npx","args":["-y","@modelcontextprotocol/server-memory"]}
```

**Configuration field reference:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MCP SERVER CONFIGURATION FIELDS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FIELD     │ REQUIRED │ DESCRIPTION                             │
│  ─────     │ ──────── │ ───────────                             │
│            │          │                                         │
│  type      │   YES    │ "stdio" or "sse"                        │
│            │          │ How VS Code communicates with the server │
│            │          │                                         │
│  command   │   YES    │ The command to launch the server         │
│  (stdio)   │          │ e.g., "npx", "python", "node"          │
│            │          │                                         │
│  args      │   NO     │ Arguments to pass to the command        │
│            │          │ e.g., ["-y", "server-package", "db-url"]│
│            │          │                                         │
│  url       │   YES    │ The HTTP endpoint of the MCP server     │
│  (sse)     │          │ e.g., "http://localhost:3001/mcp"       │
│            │          │                                         │
│  env       │   NO     │ Environment variables to set            │
│            │          │ e.g., {"API_KEY": "xxx"}                │
│            │          │                                         │
│  envFile   │   NO     │ Path to a .env file to load             │
│            │          │ e.g., ".env.local"                      │
│            │          │                                         │
└─────────────────────────────────────────────────────────────────┘
```

**Using input variables for secrets:**

> "NEVER hardcode API keys or passwords in your MCP configuration. Use input variables with `password: true` — VS Code will prompt you securely at startup."

```json
{
    "servers": {
        "my-api": {
            "type": "stdio",
            "command": "node",
            "args": ["mcp-server.js"],
            "env": {
                "DB_PASSWORD": "${input:db_password}",
                "API_KEY": "${input:api_key}"
            }
        }
    },
    "inputs": [
        {
            "id": "db_password",
            "type": "promptString",
            "description": "Database password",
            "password": true
        },
        {
            "id": "api_key",
            "type": "promptString",
            "description": "API key for external service",
            "password": true
        }
    ]
}
```

---

## 7.5 Available MCP Servers

**The MCP ecosystem is growing rapidly. Here are well-known servers:**

```
┌─────────────────────────────────────────────────────────────────┐
│              POPULAR MCP SERVERS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CATEGORY: DATABASES                                            │
│  ──────────────────                                             │
│                                                                 │
│  @modelcontextprotocol/server-postgres                          │
│  • Query PostgreSQL databases                                   │
│  • List tables, describe schemas                                │
│  • Run read-only or read-write queries                          │
│                                                                 │
│  @modelcontextprotocol/server-sqlite                            │
│  • Query SQLite databases                                       │
│  • Lightweight, no server required                              │
│                                                                 │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  CATEGORY: FILE SYSTEMS & SEARCH                                │
│  ──────────────────────────────                                 │
│                                                                 │
│  @modelcontextprotocol/server-filesystem                        │
│  • Read/write files in specified directories                    │
│  • Search files, create directories                             │
│  • Sandboxed to allowed paths                                   │
│                                                                 │
│  @modelcontextprotocol/server-brave-search                      │
│  • Web search via Brave Search API                              │
│  • Return structured search results                             │
│                                                                 │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  CATEGORY: DEVELOPMENT TOOLS                                    │
│  ────────────────────────────                                   │
│                                                                 │
│  @modelcontextprotocol/server-github                            │
│  • Create/read issues and PRs                                   │
│  • Search repositories                                          │
│  • Manage branches and files                                    │
│                                                                 │
│  @modelcontextprotocol/server-gitlab                            │
│  • Similar to GitHub server, for GitLab                         │
│                                                                 │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  CATEGORY: KNOWLEDGE & MEMORY                                   │
│  ────────────────────────────                                   │
│                                                                 │
│  @modelcontextprotocol/server-memory                            │
│  • Persistent memory for Copilot across sessions                │
│  • Store and retrieve key-value knowledge                       │
│  • "Remember that our API uses v2 auth"                         │
│                                                                 │
│  @modelcontextprotocol/server-puppeteer                         │
│  • Browser automation                                           │
│  • Take screenshots                                             │
│  • Navigate and interact with web pages                         │
│                                                                 │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  CATEGORY: CLOUD & INFRASTRUCTURE                               │
│  ────────────────────────────────                               │
│                                                                 │
│  Docker MCP Server                                              │
│  • Manage containers, images, volumes                           │
│  • Run docker commands through Copilot                          │
│                                                                 │
│  Kubernetes MCP Server                                          │
│  • Query cluster state                                          │
│  • Describe pods, services, deployments                         │
│                                                                 │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  CATEGORY: DESIGN & PRODUCT                                     │
│  ──────────────────────────                                     │
│                                                                 │
│  Figma MCP Server                                               │
│  • Read design files                                            │
│  • Extract component specs                                      │
│  • Convert designs to code requirements                         │
│                                                                 │
│  Linear / Jira MCP Server                                       │
│  • Read issues and tickets                                      │
│  • Create and update tickets from chat                          │
│  • Query sprint/project status                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Finding more MCP servers:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHERE TO FIND MCP SERVERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Official MCP Repository                                     │
│     https://github.com/modelcontextprotocol/servers             │
│     → Maintained by Anthropic, high quality                     │
│                                                                 │
│  2. MCP Registry / Community Lists                              │
│     https://mcp.so                                              │
│     → Community-curated directory of MCP servers                │
│                                                                 │
│  3. npm/PyPI Search                                             │
│     Search "mcp-server" on npm or PyPI                          │
│     → Growing ecosystem of third-party servers                  │
│                                                                 │
│  4. GitHub Topics                                               │
│     Search "mcp-server" topic on GitHub                         │
│     → Open-source implementations                              │
│                                                                 │
│  EVALUATION CRITERIA:                                           │
│  • Is it actively maintained? (check last commit)               │
│  • Does it have a clear README with setup instructions?         │
│  • Does it handle authentication securely?                      │
│  • Is it from a trusted source?                                 │
│  • Does it support the transport type you need?                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.6 Building Your Own MCP Server (Conceptual)

**You can build custom MCP servers for your team's specific needs.**

The MCP SDK is available in multiple languages. Here's the conceptual structure:

**Python MCP Server Example:**

```python
# filepath: my-mcp-server/server.py
# A simple MCP server that provides company-specific tools

from mcp.server import Server
from mcp.types import Tool, TextContent
import json

# Create an MCP server instance
server = Server("my-company-tools")

# Define available tools
@server.list_tools()
async def list_tools():
    """Tell Copilot what tools we offer."""
    return [
        Tool(
            name="get_team_members",
            description="Get the list of team members and their roles",
            inputSchema={
                "type": "object",
                "properties": {
                    "team": {
                        "type": "string",
                        "description": "Team name (e.g., 'frontend', 'backend')"
                    }
                },
                "required": ["team"]
            }
        ),
        Tool(
            name="get_coding_standards",
            description="Get the coding standards for a specific language",
            inputSchema={
                "type": "object",
                "properties": {
                    "language": {
                        "type": "string",
                        "description": "Programming language"
                    }
                },
                "required": ["language"]
            }
        ),
        Tool(
            name="query_internal_docs",
            description="Search the company's internal documentation wiki",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query"
                    }
                },
                "required": ["query"]
            }
        )
    ]

# Implement the tools
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """Execute a tool call from Copilot."""
    
    if name == "get_team_members":
        team = arguments["team"]
        # In reality, query your HR system or team database
        teams = {
            "frontend": [
                {"name": "Alice", "role": "Senior React Developer"},
                {"name": "Bob", "role": "UI/UX Engineer"},
            ],
            "backend": [
                {"name": "Charlie", "role": "Python Lead"},
                {"name": "Diana", "role": "Database Architect"},
            ]
        }
        members = teams.get(team, [])
        return [TextContent(
            type="text",
            text=json.dumps(members, indent=2)
        )]
    
    elif name == "get_coding_standards":
        language = arguments["language"]
        # In reality, fetch from your internal wiki
        standards = {
            "python": "Use ruff for linting. Black for formatting. "
                      "Type hints required on all public functions.",
            "typescript": "Use ESLint + Prettier. Strict mode enabled. "
                          "Functional components only in React."
        }
        return [TextContent(
            type="text",
            text=standards.get(language, f"No standards found for {language}")
        )]
    
    elif name == "query_internal_docs":
        query = arguments["query"]
        # In reality, search your Confluence/Notion/wiki
        return [TextContent(
            type="text",
            text=f"Documentation search results for '{query}':\n"
                 f"1. Architecture Decision Record: {query}\n"
                 f"2. How-To Guide: Working with {query}\n"
                 f"3. FAQ: Common {query} questions"
        )]
    
    else:
        return [TextContent(type="text", text=f"Unknown tool: {name}")]

# Run the server
if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server
    
    async def main():
        async with stdio_server() as (read_stream, write_stream):
            await server.run(read_stream, write_stream)
    
    asyncio.run(main())
```

**Node.js MCP Server Example:**

```typescript
// filepath: my-mcp-server/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
    ListToolsRequestSchema,
    CallToolRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
    { name: "my-company-tools", version: "1.0.0" },
    { capabilities: { tools: {} } }
);

// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
        {
            name: "search_codebase_patterns",
            description: "Search for architectural patterns used in our codebase",
            inputSchema: {
                type: "object" as const,
                properties: {
                    pattern: {
                        type: "string",
                        description: "Pattern name (e.g., 'repository', 'factory', 'observer')"
                    }
                },
                required: ["pattern"]
            }
        }
    ]
}));

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    
    if (name === "search_codebase_patterns") {
        // Your implementation here
        return {
            content: [{
                type: "text",
                text: `Found pattern: ${args?.pattern}\nUsed in: src/repositories/, src/services/`
            }]
        };
    }
    
    throw new Error(`Unknown tool: ${name}`);
});

// Start the server
const transport = new StdioServerTransport();
server.connect(transport);
```

**Configuring your custom server in VS Code:**

```json
// filepath: .vscode/mcp.json
{
    "servers": {
        "my-company-tools": {
            "type": "stdio",
            "command": "python",
            "args": ["my-mcp-server/server.py"],
            "env": {
                "INTERNAL_API_URL": "https://api.internal.company.com",
                "API_TOKEN": "${input:internal_api_token}"
            }
        }
    },
    "inputs": [
        {
            "id": "internal_api_token",
            "type": "promptString",
            "description": "Internal API token",
            "password": true
        }
    ]
}
```

**The development workflow:**

```
┌─────────────────────────────────────────────────────────────────┐
│           BUILDING A CUSTOM MCP SERVER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: IDENTIFY THE NEED                                      │
│  "What external data/actions does my team repeatedly need       │
│   when working with Copilot?"                                   │
│  • "We always need to check our internal wiki"                  │
│  • "We constantly look up deployment configurations"            │
│  • "We need to query our analytics database"                    │
│                                                                 │
│  STEP 2: CHOOSE YOUR LANGUAGE                                   │
│  Python: pip install mcp                                        │
│  Node.js: npm install @modelcontextprotocol/sdk                 │
│                                                                 │
│  STEP 3: DEFINE YOUR TOOLS                                      │
│  List the tools with clear names, descriptions, and schemas.    │
│  Good descriptions help Copilot know WHEN to use each tool.     │
│                                                                 │
│  STEP 4: IMPLEMENT THE HANDLERS                                 │
│  Write the actual logic behind each tool.                       │
│  Handle errors gracefully — return error messages, don't crash. │
│                                                                 │
│  STEP 5: CONFIGURE IN VS CODE                                   │
│  Add to .vscode/mcp.json and test in Copilot Chat.             │
│                                                                 │
│  STEP 6: SHARE WITH TEAM                                        │
│  Commit mcp.json to the repo.                                   │
│  Package the server or host it centrally.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.7 MCP vs Extensions vs Skills

**Understanding which tool to use for what:**

```
┌─────────────────────────────────────────────────────────────────┐
│            MCP vs EXTENSIONS vs SKILLS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    SKILLS                                        │
│                    ──────                                        │
│  WHAT: Built-in capabilities in Copilot                         │
│  EXAMPLES: codebase search, terminal, fetch                     │
│  SCOPE: Pre-defined by GitHub, cannot add new ones              │
│  EFFORT: Zero — just toggle on/off                              │
│  USE WHEN: The built-in capability is sufficient                │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│                    VS CODE EXTENSIONS                            │
│                    ──────────────────                            │
│  WHAT: Plugins that extend VS Code itself                       │
│  EXAMPLES: ESLint extension, GitLens, Prettier                  │
│  SCOPE: Can add UI, commands, languages, debug adapters         │
│  EFFORT: Medium (install from marketplace) or high (build)      │
│  USE WHEN: You need VS Code UI integration, language support,   │
│            or functionality beyond AI chat                       │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│                    MCP SERVERS                                   │
│                    ───────────                                   │
│  WHAT: External tool servers that Copilot can call              │
│  EXAMPLES: Database query, Jira integration, wiki search        │
│  SCOPE: Any external system, any language                       │
│  EFFORT: Low (use existing) or medium (build your own)          │
│  USE WHEN: Copilot needs access to external data/systems        │
│            that aren't in your workspace files                  │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│  DECISION TREE:                                                 │
│                                                                 │
│  Need external data in chat? ──YES──▶ MCP Server                │
│         │ NO                                                    │
│         ▼                                                       │
│  Need VS Code UI changes? ──YES──▶ VS Code Extension            │
│         │ NO                                                    │
│         ▼                                                       │
│  Built-in skill exists? ──YES──▶ Use the Skill                  │
│         │ NO                                                    │
│         ▼                                                       │
│  Consider building an MCP server or extension                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**They can work together:**

> "These aren't competing approaches — they're complementary layers. A VS Code extension might install an MCP server. An agent's prompt file might reference skills AND MCP tools. The magic is in combining them."

```
┌─────────────────────────────────────────────────────────────────┐
│              COMBINING ALL THREE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO: Full-stack development with Copilot                  │
│                                                                 │
│  SKILLS (built-in):                                             │
│  ├─ codebase → Search project files                             │
│  ├─ terminal → Run npm/docker commands                          │
│  └─ fetch → Read external documentation                         │
│                                                                 │
│  MCP SERVERS (external):                                        │
│  ├─ PostgreSQL → Query the development database                 │
│  ├─ GitHub → Check open issues for context                      │
│  └─ Memory → Remember project-specific knowledge                │
│                                                                 │
│  EXTENSIONS:                                                    │
│  ├─ ESLint → Real-time linting UI                               │
│  ├─ Prisma → Schema syntax highlighting                         │
│  └─ GitLens → Git blame and history UI                          │
│                                                                 │
│  CUSTOM AGENT tying it together:                                │
│  @fullstack-dev with tools: [codebase, terminal,                │
│                              mcp-postgres, mcp-github]          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 8: TOOL SETS (The Equipment Assignments)

## 8.1 What Are Tool Sets?

**Tool Sets let you group and organize tools into named collections that can be assigned to agents or enabled/disabled together.**

Think of tool sets like equipment kits given to different roles at a company. The IT department gets network tools, the developers get coding tools, the designers get design tools. Nobody gets everything — that would be chaotic and potentially dangerous.

```
┌─────────────────────────────────────────────────────────────────┐
│                      TOOL SETS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Named collections of tools (skills + MCP tools)         │
│         that can be referenced and assigned as a group.         │
│                                                                 │
│  WHERE: Defined in VS Code settings or .vscode/mcp.json,       │
│         and referenced in agent prompt files.                   │
│                                                                 │
│  WHY:   Managing individual tools per agent gets messy.         │
│         Tool sets let you say "this agent gets the 'backend     │
│         toolkit'" instead of listing 15 individual tools.       │
│                                                                 │
│  ANALOGY: Equipment kits at a construction site.                │
│           "Electricians get Kit A (multimeter, wire strippers,  │
│            voltage tester). Plumbers get Kit B (pipe wrench,    │
│            soldering iron, pipe cutter). Nobody gets the        │
│            other team's specialized tools."                     │
│                                                                 │
│  KEY VALUE:                                                     │
│  ├─ ORGANIZATION: Group related tools with semantic names       │
│  ├─ REUSABILITY: Define once, assign to multiple agents         │
│  ├─ SAFETY: Restrict tools to appropriate contexts              │
│  └─ MAINTENANCE: Update the set, all agents get the change     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.2 Defining Tool Sets

**Tool sets are defined in your VS Code configuration:**

```json
// filepath: .vscode/settings.json
{
    "github.copilot.chat.toolSets": [
        {
            "name": "backend-toolkit",
            "description": "Tools for backend development tasks",
            "tools": [
                "codebase",
                "terminal",
                "editFiles",
                "mcp:my-database:query_database",
                "mcp:my-database:list_tables",
                "mcp:my-database:describe_table"
            ]
        },
        {
            "name": "frontend-toolkit",
            "description": "Tools for frontend development tasks",
            "tools": [
                "codebase",
                "fetch",
                "editFiles",
                "mcp:figma-server:get_design",
                "mcp:figma-server:export_assets"
            ]
        },
        {
            "name": "review-toolkit",
            "description": "Tools for code review (read-only, no editing)",
            "tools": [
                "codebase",
                "useDiff",
                "fetch"
            ]
        },
        {
            "name": "devops-toolkit",
            "description": "Tools for infrastructure and deployment",
            "tools": [
                "codebase",
                "terminal",
                "editFiles",
                "mcp:docker-server:list_containers",
                "mcp:docker-server:container_logs",
                "mcp:k8s-server:get_pods",
                "mcp:k8s-server:describe_service"
            ]
        }
    ]
}
```

**Anatomy of a tool set definition:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TOOL SET STRUCTURE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  {                                                              │
│    "name": "backend-toolkit",                                   │
│     │                                                           │
│     └─ Unique identifier. Used to reference this set            │
│        in agent configurations.                                 │
│                                                                 │
│    "description": "Tools for backend development",              │
│     │                                                           │
│     └─ Human-readable text shown in the UI.                     │
│        Helps you and your team understand the purpose.          │
│                                                                 │
│    "tools": [                                                   │
│     │                                                           │
│     ├─ "codebase"              ← Built-in skill                 │
│     ├─ "terminal"              ← Built-in skill                 │
│     ├─ "editFiles"             ← Built-in skill                 │
│     └─ "mcp:server-name:tool"  ← MCP server tool               │
│         │    │           │                                      │
│         │    │           └─ Specific tool from that server      │
│         │    └─ Name of the MCP server (from mcp.json)          │
│         └─ Prefix indicating it's an MCP tool                   │
│    ]                                                            │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.3 Assigning Tools to Agents

**Tool sets create a clean separation between tool management and agent design:**

**Direct tool assignment (without tool sets) — gets messy:**

```markdown
<!-- filepath: .github/prompts/backend-expert.prompt.md -->
---
mode: agent
description: "Backend development specialist"
tools: ["codebase", "terminal", "editFiles", "mcp:my-database:query_database", "mcp:my-database:list_tables", "mcp:my-database:describe_table"]
---
# Backend Expert
...
```

```markdown
<!-- filepath: .github/prompts/api-designer.prompt.md -->
---
mode: agent
description: "API design specialist"
tools: ["codebase", "terminal", "editFiles", "mcp:my-database:query_database", "mcp:my-database:list_tables", "mcp:my-database:describe_table"]
---
# API Designer
...
```

**Notice the duplication? Now imagine you add a new database tool — you'd have to update every agent file.**

**With tool sets — clean and DRY:**

```markdown
<!-- filepath: .github/prompts/backend-expert.prompt.md -->
---
mode: agent
description: "Backend development specialist"
toolSets: ["backend-toolkit"]
---
# Backend Expert
...
```

```markdown
<!-- filepath: .github/prompts/api-designer.prompt.md -->
---
mode: agent
description: "API design specialist"
toolSets: ["backend-toolkit"]
---
# API Designer
...
```

**Now add a new tool to `backend-toolkit` once → both agents get it automatically.**

**Combining tool sets:**

```markdown
<!-- filepath: .github/prompts/fullstack-dev.prompt.md -->
---
mode: agent
description: "Full-stack development with access to everything"
toolSets: ["backend-toolkit", "frontend-toolkit"]
---
# Full-Stack Developer Agent
You have access to both backend and frontend tools.
...
```

**Visualization of tool set assignment:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TOOL SETS → AGENTS MAPPING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TOOL SETS                        AGENTS                        │
│  ─────────                        ──────                        │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │ backend-toolkit  │────────────▶ @backend-expert              │
│  │ • codebase       │────────────▶ @api-designer                │
│  │ • terminal       │────────┐                                  │
│  │ • editFiles      │        │                                  │
│  │ • mcp:database   │        │                                  │
│  └─────────────────┘         ├──▶ @fullstack-dev               │
│                              │                                  │
│  ┌─────────────────┐         │                                  │
│  │ frontend-toolkit │────────┘                                  │
│  │ • codebase       │────────────▶ @frontend-expert             │
│  │ • fetch          │                                           │
│  │ • editFiles      │                                           │
│  │ • mcp:figma      │                                           │
│  └─────────────────┘                                            │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │ review-toolkit   │────────────▶ @code-reviewer               │
│  │ • codebase       │────────────▶ @security-auditor            │
│  │ • useDiff        │                                           │
│  │ • fetch          │                                           │
│  │ (NO editFiles!)  │ ← Intentionally read-only                │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.4 Tool Permissions and Safety

**Tool sets are also a SECURITY mechanism. Control what each agent can do:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TOOL SAFETY THROUGH TOOL SETS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRINCIPLE OF LEAST PRIVILEGE:                                  │
│  ────────────────────────────                                   │
│  Give each agent ONLY the tools it needs. Nothing more.         │
│                                                                 │
│  EXAMPLE: Why the @code-reviewer should NOT have editFiles:     │
│                                                                 │
│  You ask: "@code-reviewer review my PR"                         │
│                                                                 │
│  WITH editFiles: The reviewer might "helpfully" auto-fix        │
│  issues — changing your code unexpectedly. Dangerous.           │
│                                                                 │
│  WITHOUT editFiles: The reviewer can only READ and COMMENT.     │
│  You decide what to change. Safe.                               │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  EXAMPLE: Why @security-auditor should NOT have terminal:       │
│                                                                 │
│  You don't want the security auditor agent accidentally         │
│  running exploit-testing commands on your system.               │
│  Read-only access is sufficient for audit.                      │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  EXAMPLE: Why @scaffolder SHOULD have editFiles + terminal:     │
│                                                                 │
│  Its entire job is creating files and running generators.       │
│  Without these tools, it can't do its job at all.              │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  BEST PRACTICES:                                                │
│  ├─ Start restrictive, add tools as needed                      │
│  ├─ Read-only agents should NEVER have editFiles or terminal    │
│  ├─ Review which MCP tools each agent can access                │
│  ├─ Use descriptive tool set names so permissions are obvious   │
│  └─ Document WHY each tool set has the tools it does            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tool set permission matrix example:**

```
┌─────────────────────────────────────────────────────────────────┐
│           PERMISSION MATRIX                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TOOL            │ backend │ frontend│ review │ devops │ docs   │
│  ────            │ ─────── │ ────────│ ────── │ ────── │ ────   │
│                  │         │         │        │        │        │
│  codebase        │   ✅    │   ✅    │  ✅    │  ✅    │  ✅    │
│  editFiles       │   ✅    │   ✅    │  ❌    │  ✅    │  ✅    │
│  terminal        │   ✅    │   ❌    │  ❌    │  ✅    │  ❌    │
│  fetch           │   ❌    │   ✅    │  ✅    │  ✅    │  ✅    │
│  useDiff         │   ❌    │   ❌    │  ✅    │  ❌    │  ❌    │
│  mcp:database    │   ✅    │   ❌    │  ❌    │  ❌    │  ❌    │
│  mcp:figma       │   ❌    │   ✅    │  ❌    │  ❌    │  ❌    │
│  mcp:docker      │   ❌    │   ❌    │  ❌    │  ✅    │  ❌    │
│  mcp:k8s         │   ❌    │   ❌    │  ❌    │  ✅    │  ❌    │
│                  │         │         │        │        │        │
│  RISK LEVEL      │ Medium  │  Low    │ Lowest │ High   │ Low    │
│                  │         │         │        │        │        │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 9: DIAGNOSTICS (The Health Check)

## 9.1 What Does Diagnostics Show?

**Diagnostics is the health check panel for your Copilot installation.**

When things aren't working — Copilot isn't responding, agents aren't appearing, MCP servers fail to connect — Diagnostics is where you go first.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DIAGNOSTICS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  A diagnostic panel that shows the health and status     │
│         of your GitHub Copilot installation, authentication,    │
│         active features, and configuration.                     │
│                                                                 │
│  WHERE: Accessed from the Copilot Chat settings menu →          │
│         "Diagnostics", or via Command Palette:                  │
│         Ctrl+Shift+P → "GitHub Copilot: Diagnostics"           │
│                                                                 │
│  WHY:   When something breaks, you need to know WHERE.         │
│         Diagnostics tells you the exact state of every          │
│         component in the Copilot ecosystem.                     │
│                                                                 │
│  ANALOGY: The dashboard on your car.                            │
│           Check engine light? Low oil? Battery dying?           │
│           You don't pop the hood and guess — you check          │
│           the dashboard indicators first.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.2 Troubleshooting Common Issues

**When Copilot isn't working, use this systematic approach:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TROUBLESHOOTING FLOWCHART                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYMPTOM: Copilot Chat not responding at all                    │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  Step 1: Check Diagnostics                                      │
│          Ctrl+Shift+P → "GitHub Copilot: Diagnostics"          │
│          │                                                      │
│          ▼                                                      │
│  ┌──────────────────────┐                                       │
│  │ Authentication OK?   │                                       │
│  └──────────┬───────────┘                                       │
│        │         │                                              │
│       YES        NO → Sign in again:                            │
│        │               Ctrl+Shift+P → "GitHub: Sign In"        │
│        ▼                                                        │
│  ┌──────────────────────┐                                       │
│  │ Subscription active? │                                       │
│  └──────────┬───────────┘                                       │
│        │         │                                              │
│       YES        NO → Check github.com/settings/copilot        │
│        │               Verify your subscription is active       │
│        ▼                                                        │
│  ┌──────────────────────┐                                       │
│  │ Extension enabled?   │                                       │
│  └──────────┬───────────┘                                       │
│        │         │                                              │
│       YES        NO → Extensions panel → Enable Copilot        │
│        │                                                        │
│        ▼                                                        │
│  ┌──────────────────────┐                                       │
│  │ Network connectivity?│                                       │
│  └──────────┬───────────┘                                       │
│        │         │                                              │
│       YES        NO → Check proxy/firewall settings             │
│        │               copilot-proxy.githubusercontent.com      │
│        │               must be accessible                       │
│        ▼                                                        │
│  Try: Ctrl+Shift+P → "Developer: Reload Window"                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│              MORE TROUBLESHOOTING SCENARIOS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYMPTOM: Custom agent not appearing in chat                    │
│  ─────────────────────────────────────────                      │
│  • Check: Is the file in .github/prompts/ directory?            │
│  • Check: Does the filename end with .prompt.md?                │
│  • Check: Does it have `mode: agent` in YAML front matter?     │
│  • Check: Is the YAML valid? (no syntax errors)                │
│  • Try:   Reload window (Ctrl+Shift+P → Reload Window)         │
│                                                                 │
│  SYMPTOM: MCP server not connecting                             │
│  ───────────────────────────────────                            │
│  • Check: Output panel → "MCP" channel for error messages      │
│  • Check: Is the command installed? (e.g., `npx`, `python`)    │
│  • Check: Are all required env variables set?                   │
│  • Check: Can you run the server command manually in terminal?  │
│  • Try:   Ctrl+Shift+P → "MCP: List Servers" to see status     │
│                                                                 │
│  SYMPTOM: Instructions not being followed                       │
│  ─────────────────────────────────────────                      │
│  • Check: Is .github/copilot-instructions.md at the right      │
│           path? (workspace root, not subdirectory)              │
│  • Check: Are settings-based instructions properly formatted?   │
│  • Check: Is the file too long? (keep under ~100 lines)         │
│  • Try:   Make instructions more specific and concrete          │
│                                                                 │
│  SYMPTOM: Hooks not running                                     │
│  ─────────────────────────────                                  │
│  • Check: Is workspace trusted? (Trust prompt on first open)    │
│  • Check: Is the hook command valid? Test it in terminal.       │
│  • Check: Is the hook event name correct?                       │
│           (postSaveFile, postCreateFile, postEditFile)          │
│  • Check: Output panel → "GitHub Copilot" for error logs       │
│                                                                 │
│  SYMPTOM: Skills not available                                  │
│  ────────────────────────────                                   │
│  • Check: Are they enabled in the Skills settings menu?         │
│  • Check: Does the current model support the skill?             │
│  • Check: Are you in the right mode (chat vs. agent)?           │
│           Some tools only work in agent mode.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.3 Checking Authentication & Entitlements

**Understanding what your Copilot subscription gives you:**

```
┌─────────────────────────────────────────────────────────────────┐
│           COPILOT SUBSCRIPTION TIERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIER                  │ FEATURES                               │
│  ────                  │ ────────                               │
│                        │                                        │
│  Copilot Free          │ • Limited chat & completions           │
│                        │ • Basic models only                    │
│                        │ • Limited monthly usage                │
│                        │ • Basic skills                         │
│                        │                                        │
│  Copilot Pro           │ • Unlimited chat & completions         │
│  ($10/month)           │ • Multiple models (GPT-4o, Claude,    │
│                        │   Gemini, o1, etc.)                    │
│                        │ • Agent mode                           │
│                        │ • MCP server support                   │
│                        │ • Custom agents & prompt files         │
│                        │                                        │
│  Copilot Business      │ • Everything in Pro                    │
│  ($19/user/month)      │ • Organization-wide policies           │
│                        │ • Audit logs                           │
│                        │ • IP indemnity                         │
│                        │ • Content exclusions                   │
│                        │                                        │
│  Copilot Enterprise    │ • Everything in Business               │
│  ($39/user/month)      │ • Knowledge bases                      │
│                        │ • Fine-tuned models                    │
│                        │ • @github agent with org search        │
│                        │ • Bing web search in chat              │
│                        │                                        │
│                                                                 │
│  TO CHECK YOUR TIER:                                            │
│  1. Open Diagnostics (Ctrl+Shift+P → Copilot: Diagnostics)     │
│  2. Look for "Entitlements" section                             │
│  3. Or visit: https://github.com/settings/copilot              │
│                                                                 │
│  NOTE: Some features in this lecture (MCP servers, agent mode,  │
│  multi-model) require Copilot Pro or higher.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to verify authentication is working:**

```bash
# In VS Code terminal, check if the GitHub CLI is authenticated
gh auth status

# Expected output:
# ✓ Logged in to github.com account YourUsername
# ✓ Token: gho_****
# ✓ Token scopes: copilot, ...
```

**Common authentication issues:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTHENTICATION ISSUES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ISSUE: "Not authorized" errors                                 │
│  FIX:   Sign out and back in:                                   │
│         Ctrl+Shift+P → "GitHub Copilot: Sign Out"              │
│         Ctrl+Shift+P → "GitHub: Sign In"                       │
│                                                                 │
│  ISSUE: "You don't have access to Copilot"                      │
│  FIX:   • Verify subscription at github.com/settings/copilot   │
│         • If org-managed, ask admin to add you                  │
│         • Check if your org allows Copilot for your team        │
│                                                                 │
│  ISSUE: Token expired                                           │
│  FIX:   Tokens refresh automatically, but if stuck:             │
│         • Clear VS Code's stored credentials                    │
│         • Ctrl+Shift+P → "Clear Authentication Session"        │
│         • Sign in again fresh                                   │
│                                                                 │
│  ISSUE: Behind corporate proxy/VPN                              │
│  FIX:   Configure VS Code proxy settings:                       │
│         "http.proxy": "http://proxy.company.com:8080"          │
│         Ensure these domains are allowed:                       │
│         • github.com                                            │
│         • api.github.com                                        │
│         • copilot-proxy.githubusercontent.com                   │
│         • *.githubcopilot.com                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.4 Reading Diagnostic Logs

**Where to find logs when things go wrong:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DIAGNOSTIC LOG LOCATIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOCATION 1: Output Panel → "GitHub Copilot Chat"               │
│  ─────────────────────────────────────────────────              │
│  View → Output (Ctrl+Shift+U)                                  │
│  Select "GitHub Copilot Chat" from the dropdown                 │
│                                                                 │
│  Shows: Chat request/response logs, errors, tool invocations   │
│                                                                 │
│                                                                 │
│  LOCATION 2: Output Panel → "GitHub Copilot"                    │
│  ────────────────────────────────────────────                   │
│  Shows: General Copilot extension logs (completions, status)   │
│                                                                 │
│                                                                 │
│  LOCATION 3: Output Panel → "MCP"                               │
│  ─────────────────────────────────                              │
│  Shows: MCP server startup, tool discovery, errors              │
│  Critical for debugging MCP server connection issues            │
│                                                                 │
│                                                                 │
│  LOCATION 4: Developer Tools Console                            │
│  ───────────────────────────────────                            │
│  Help → Toggle Developer Tools (Ctrl+Shift+I)                  │
│  Console tab shows low-level errors                             │
│                                                                 │
│                                                                 │
│  LOCATION 5: Copilot Diagnostics Panel                          │
│  ─────────────────────────────────────                          │
│  Ctrl+Shift+P → "GitHub Copilot: Diagnostics"                  │
│  Shows: Authentication state, editor info, network reachability │
│                                                                 │
│                                                                 │
│  EXPERT TIP: Enable verbose logging                             │
│  ──────────────────────────────────                             │
│  settings.json:                                                 │
│  {                                                              │
│    "github.copilot.advanced": {                                 │
│      "debug.overrideLogLevels": {                               │
│        "*": "DEBUG"                                             │
│      }                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  ⚠️  Turn this off after debugging — produces massive logs      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Reading a diagnostic output example:**

```
┌─────────────────────────────────────────────────────────────────┐
│           SAMPLE DIAGNOSTIC OUTPUT (Annotated)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GitHub Copilot Diagnostics                                     │
│  ─────────────────────────                                      │
│                                                                 │
│  Version: 1.250.0                 ← Extension version           │
│  Build: production                ← Release or Insiders         │
│  VS Code: 1.100.0                 ← Editor version              │
│  OS: Linux x64 6.5.0             ← Operating system             │
│                                                                 │
│  Authentication:                                                │
│    Status: Authenticated ✅       ← Good                        │
│    User: JIMMY-1732               ← Your GitHub username        │
│    Plan: copilot_pro              ← Subscription tier           │
│                                                                 │
│  Entitlements:                                                  │
│    Chat: enabled ✅               ← Can use chat                │
│    Completions: enabled ✅        ← Can use inline completions  │
│    Agent Mode: enabled ✅         ← Can use agent mode          │
│    MCP: enabled ✅                ← Can use MCP servers         │
│                                                                 │
│  Network:                                                       │
│    Proxy: none                    ← Direct connection           │
│    Reachable: yes ✅              ← Can reach Copilot servers   │
│                                                                 │
│  Active Configuration:                                          │
│    Instructions file: found ✅    ← .github/copilot-            │
│                                      instructions.md exists     │
│    Prompt files: 3 found          ← 3 .prompt.md files found   │
│    MCP servers: 1 running ✅      ← 1 MCP server active        │
│    Hooks: 2 configured            ← 2 hook events configured   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What to look for:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DIAGNOSTIC RED FLAGS 🚩                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Authentication: Status: Not authenticated ❌                   │
│  → Action: Sign in again                                        │
│                                                                 │
│  Plan: copilot_free (some features limited)                     │
│  → Action: Upgrade if you need agent mode, MCP, etc.           │
│                                                                 │
│  Network: Reachable: no ❌                                      │
│  → Action: Check proxy, firewall, VPN settings                 │
│                                                                 │
│  MCP servers: 0 running (1 configured) ⚠️                      │
│  → Action: Check MCP output panel for startup errors           │
│                                                                 │
│  Instructions file: not found ⚠️                                │
│  → Action: Create .github/copilot-instructions.md              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 10: CHAT SETTINGS (The Control Panel)

## 10.1 Overview of Chat Settings

**Chat Settings is the control panel for how Copilot Chat behaves on a day-to-day basis.**

While Instructions control WHAT Copilot says and Agents control WHO Copilot acts as, Chat Settings control the underlying ENGINE — which model, how much context, and how the interface behaves.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHAT SETTINGS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT:  Configuration options that control the fundamental      │
│         behavior of Copilot Chat — the model used, context      │
│         management, code generation preferences, and UI         │
│         behavior.                                               │
│                                                                 │
│  WHERE: The "Chat Settings" item in the Copilot settings menu,  │
│         or directly in VS Code settings.json under              │
│         github.copilot.chat.* keys.                             │
│                                                                 │
│  WHY:   Different tasks need different configurations.          │
│         A quick question needs a fast model. A complex          │
│         architecture discussion needs the most capable model.   │
│         Privacy-sensitive work needs local models.              │
│                                                                 │
│  ANALOGY: The control panel on a machine.                       │
│           Speed dial, precision knob, mode selector.            │
│           Same machine, different settings for different jobs.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.2 Model Selection

**One of the most impactful settings: which AI model powers your Copilot.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    MODEL SELECTION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Copilot Pro/Business/Enterprise users can choose from          │
│  multiple models. Each has different strengths:                  │
│                                                                 │
│  MODEL                │ STRENGTHS           │ BEST FOR           │
│  ─────                │ ─────────           │ ────────           │
│                       │                     │                    │
│  GPT-4o               │ Fast, well-rounded  │ General coding,    │
│                       │ Good at code gen    │ daily tasks        │
│                       │                     │                    │
│  GPT-4o-mini          │ Very fast, cheaper  │ Quick questions,   │
│                       │ Good enough for     │ simple completions │
│                       │ simple tasks        │                    │
│                       │                     │                    │
│  Claude 3.5 Sonnet    │ Excellent reasoning │ Complex logic,     │
│  / Claude Opus 4      │ Strong analysis     │ architecture,      │
│                       │ Nuanced responses   │ code review        │
│                       │                     │                    │
│  Gemini 2.0 Flash     │ Fast, large context │ Large file         │
│                       │ window              │ analysis,          │
│                       │                     │ multi-file tasks   │
│                       │                     │                    │
│  o1 / o3-mini         │ Deep reasoning      │ Complex algorithms,│
│                       │ "Thinking" models   │ math, debugging    │
│                       │ More deliberate     │ tricky bugs        │
│                       │                     │                    │
│  GPT-5.3-Codex       │ Optimized for code  │ Agent tasks,       │
│  (shown in your       │ generation and      │ multi-step code    │
│   screenshot)         │ agentic workflows   │ generation         │
│                       │                     │                    │
│                                                                 │
│  HOW TO SWITCH:                                                 │
│  • Click the model name at the bottom of the chat panel         │
│  • Or: Ctrl+Shift+P → "Chat: Select Model"                     │
│  • Or: Click the model selector dropdown in the chat input      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Model selection strategy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO USE WHICH MODEL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Write a simple utility function"                              │
│  → GPT-4o-mini (fast, simple task)                              │
│                                                                 │
│  "Generate a full CRUD API with tests"                          │
│  → GPT-4o or Claude Sonnet 4 (balanced capability)              │
│                                                                 │
│  "Review this complex authentication flow for security issues"  │
│  → Claude Opus 4 or o1 (deep reasoning needed)                  │
│                                                                 │
│  "Analyze these 5 large files and explain the data pipeline"    │
│  → Gemini 2.0 Flash (large context window)                      │
│                                                                 │
│  "Debug why this recursive algorithm produces wrong results"    │
│  → o1 or o3-mini (deliberate reasoning for tricky logic)        │
│                                                                 │
│  "Execute an agent workflow: create files, run tests, fix bugs" │
│  → GPT-5.3-Codex (optimized for multi-step agent tasks)        │
│                                                                 │
│  GENERAL RULE:                                                  │
│  Start with a fast model (GPT-4o). If the response isn't       │
│  good enough, switch to a more capable model and retry.         │
│  Don't use the "biggest" model for every simple question.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Setting a default model in settings:**

```json
{
    // Set your preferred default model
    "github.copilot.chat.defaultModel": "gpt-4o",
    
    // For inline completions (separate from chat)
    "github.copilot.advanced": {
        "model": "gpt-4o"
    }
}
```

---

## 10.3 Context Window and Token Limits

**Understanding context is critical for getting good results:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTEXT WINDOW EXPLAINED                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT IS A CONTEXT WINDOW?                                      │
│  ──────────────────────────                                     │
│  The "memory" of the AI model for a single conversation.        │
│  Everything the model can "see" at once:                        │
│  • Your system instructions                                     │
│  • copilot-instructions.md content                              │
│  • Active prompt file content                                   │
│  • File contents you've included                                │
│  • Chat history (your questions + Copilot's answers)            │
│  • MCP tool results                                             │
│                                                                 │
│  Think of it as a DESK. The context window is the desk size.    │
│  If you pile too many papers on it, older papers fall off.      │
│                                                                 │
│                                                                 │
│  MODEL CONTEXT WINDOWS (approximate):                           │
│  ─────────────────────────────────────                          │
│  GPT-4o:              128K tokens (~100K words)                 │
│  GPT-4o-mini:         128K tokens                               │
│  Claude Sonnet 4:     200K tokens (~150K words)                 │
│  Gemini 2.0 Flash:    1M tokens (~750K words)                   │
│  o1:                  200K tokens                               │
│                                                                 │
│                                                                 │
│  WHY IT MATTERS:                                                │
│  If your instructions + file contents + chat history exceed     │
│  the context window, OLDER messages get truncated.              │
│  The model literally FORGETS earlier parts of the conversation. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical implications:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTEXT MANAGEMENT TIPS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. KEEP INSTRUCTIONS CONCISE                                   │
│     Your copilot-instructions.md counts toward the context.     │
│     A 500-line instructions file eats into your available       │
│     space for actual conversation. Keep it 50-100 lines.        │
│                                                                 │
│  2. START NEW CHATS FOR NEW TOPICS                              │
│     A long conversation accumulates context. When you switch    │
│     topics, start a new chat to get a fresh context window.     │
│     Ctrl+L or click "New Chat" (+) button.                      │
│                                                                 │
│  3. BE SELECTIVE WITH FILE INCLUDES                             │
│     Don't include entire large files if you only need a         │
│     specific function. Use #selection or reference specific     │
│     sections.                                                   │
│                                                                 │
│  4. USE @workspace WISELY                                       │
│     @workspace searches and includes relevant snippets,         │
│     not entire files. It's context-efficient by design.         │
│                                                                 │
│  5. MONITOR CONTEXT USAGE                                       │
│     Some models show token usage. If responses start getting    │
│     degraded or "forgetting" things you said earlier, you've    │
│     likely hit context limits. Start a new chat.                │
│                                                                 │
│  6. CHOOSE MODELS BY CONTEXT NEEDS                              │
│     Working with many files? → Gemini (1M tokens)               │
│     Simple question? → GPT-4o-mini is plenty                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.4 Code Generation Preferences

**Control how Copilot generates code by default:**

```json
// filepath: .vscode/settings.json (workspace settings)
{
    // Instructions for code generation specifically
    "github.copilot.chat.codeGeneration.instructions": [
        { "text": "Always generate TypeScript, never JavaScript." },
        { "text": "Include comprehensive error handling." },
        { "text": "Add JSDoc comments for all exported functions." },
        { "file": ".github/copilot-instructions.md" }
    ],
    
    // Instructions for test generation specifically
    "github.copilot.chat.testGeneration.instructions": [
        { "text": "Use Vitest as the test runner." },
        { "text": "Use React Testing Library for component tests." },
        { "text": "Follow Arrange-Act-Assert pattern." },
        { "text": "Test edge cases and error scenarios." }
    ],
    
    // Instructions for code review specifically
    "github.copilot.chat.reviewSelection.instructions": [
        { "text": "Check for security vulnerabilities (OWASP Top 10)." },
        { "text": "Flag any hardcoded secrets or API keys." },
        { "text": "Suggest performance improvements." },
        { "text": "Verify error handling is comprehensive." }
    ],
    
    // Instructions for commit message generation
    "github.copilot.chat.commitMessageGeneration.instructions": [
        { "text": "Use Conventional Commits format." },
        { "text": "Prefix with type: feat, fix, docs, style, refactor, test, chore." },
        { "text": "Keep the first line under 72 characters." },
        { "text": "Include a body if the change is non-trivial." }
    ]
}
```

**The different instruction scopes:**

```
┌─────────────────────────────────────────────────────────────────┐
│           INSTRUCTION SCOPES IN SETTINGS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETTING KEY                              │ APPLIES TO           │
│  ───────────                              │ ──────────           │
│                                           │                      │
│  chat.codeGeneration.instructions         │ When Copilot         │
│                                           │ generates new code   │
│                                           │                      │
│  chat.testGeneration.instructions         │ When Copilot         │
│                                           │ generates tests      │
│                                           │ (via "Generate tests │
│                                           │  for this code")     │
│                                           │                      │
│  chat.reviewSelection.instructions        │ When Copilot         │
│                                           │ reviews selected     │
│                                           │ code                 │
│                                           │                      │
│  chat.commitMessageGeneration.            │ When Copilot         │
│      instructions                         │ generates commit     │
│                                           │ messages             │
│                                           │                      │
│  .github/copilot-instructions.md          │ ALL interactions     │
│  (file-based)                             │ (always included)    │
│                                           │                      │
│                                                                 │
│  LAYERING: All applicable scopes are combined.                  │
│  For code generation, Copilot reads:                            │
│  1. .github/copilot-instructions.md (always)                    │
│  2. chat.codeGeneration.instructions (for this task type)       │
│  3. The agent's own instructions (if using an agent)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.5 Privacy and Data Settings

**Understanding what data Copilot uses and how to control it:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PRIVACY & DATA SETTINGS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT DATA COPILOT ACCESSES:                                    │
│  ────────────────────────────                                   │
│  • Active editor content (the file you're looking at)           │
│  • Open tabs (for additional context)                           │
│  • Files you explicitly include (#file references)              │
│  • @workspace search results                                    │
│  • MCP server responses                                         │
│  • Your chat messages                                           │
│  • .github/copilot-instructions.md content                      │
│                                                                 │
│  WHAT DATA IS SENT TO GITHUB SERVERS:                           │
│  ─────────────────────────────────────                          │
│  • Code snippets (for completion/chat)                          │
│  • Prompts and context                                          │
│  • Telemetry data (if enabled)                                  │
│                                                                 │
│  DATA RETENTION (varies by plan):                               │
│  ─────────────────────────────────                              │
│  • Copilot Individual: Prompts may be retained for improvement  │
│  • Copilot Business: No prompt retention by default             │
│  • Copilot Enterprise: No prompt retention, additional controls │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  KEY PRIVACY SETTINGS:                                          │
│                                                                 │
│  // Disable telemetry collection                                │
│  "telemetry.telemetryLevel": "off"                              │
│                                                                 │
│  // Control whether Copilot uses your data for improvements     │
│  // (managed at https://github.com/settings/copilot)            │
│  //  "Allow GitHub to use my code snippets for product          │
│  //   improvements" → Uncheck for maximum privacy               │
│                                                                 │
│  // Exclude specific files/patterns from Copilot                │
│  // (Copilot Business/Enterprise)                               │
│  // Managed in organization settings                            │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  FILE EXCLUSIONS (what Copilot won't read):                     │
│  ───────────────────────────────────────────                    │
│  Configure in VS Code settings:                                 │
│                                                                 │
│  "github.copilot.advanced": {                                   │
│      "excludeFiles": [                                          │
│          "**/.env",                                             │
│          "**/.env.*",                                           │
│          "**/secrets/**",                                       │
│          "**/credentials/**"                                    │
│      ]                                                          │
│  }                                                              │
│                                                                 │
│  These glob patterns tell Copilot to completely IGNORE          │
│  matching files — they won't be sent as context, won't          │
│  appear in completions, won't be searchable via @workspace.    │
│                                                                 │
│  ALWAYS EXCLUDE:                                                │
│  ├─ .env files (API keys, database passwords)                   │
│  ├─ Private key files (*.pem, *.key)                            │
│  ├─ Credential stores                                           │
│  ├─ Patient/customer data files                                 │
│  └─ Any file you wouldn't share publicly                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Additional useful Chat Settings:**

```json
{
    // Experimental: use context from open editors
    "github.copilot.chat.useProjectContext": true,
    
    // Control inline chat behavior
    "github.copilot.chat.localeOverride": "en",
    
    // Control the chat panel location
    "github.copilot.chat.terminalChatLocation": "chatView",
    
    // Enable/disable specific Copilot features
    "github.copilot.enable": {
        "*": true,
        "plaintext": false,
        "markdown": true,
        "scminput": true
    },
    
    // Disable Copilot for specific file types
    // Useful for sensitive file types
    "github.copilot.enable": {
        "yaml": true,
        "json": true,
        "env": false
    }
}
```

