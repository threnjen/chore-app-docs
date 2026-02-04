#### Tests
Write tests BEFORE implementation
Confirm tests fail (avoid mock implementations)
Commit tests separately
Implement until tests pass
Do NOT modify tests during implementation
Implement your plan and make sure your new tests pass.
Always run tests to make sure you didn't break anything else.
Always run prettier on newly created files.
Always run turbo typecheck lint.

#### Hooks
Use hooks to enforce quality automatically:

TypeScript/linter checks after every edit
Build validation before commits
Test execution on file changes
Formatting automation (though see caveat below)
Hook Example (from 6_months_hardcore_use):

// Stop hook: Runs when Claude finishes responding
1. Read edit logs to find modified repos
2. Run build scripts on each affected repo
3. Check for TypeScript errors
4. If <5 errors: Show them to Claude
5. If ≥5 errors: Recommend auto-error-resolver agent
6. Log everything

#### Self Review

review your own code using subagents or fresh context
Have one Claude write, another review (fresh context = better critique)

What to Look For:

Spaghetti code (hard to follow logic)
Substantial API/backend changes
Unnecessary imports, functions, comments
Missing error handling
Security vulnerabilities

#### Commits

Commit early and often with meaningful messages

Use Conventional Commits format
Each commit should compile and pass tests
Avoid references to “Claude” or “AI-generated” in messages
Commit in stages tied to plan/task checkpoints
Example from Building_AI_Factory:

"One important instruction is to have claude write commits
as it goes for each task step. This way either claude or I
can revert to a previous state if something goes wrong."

#### Context Clearing

Clear at 60k tokens or 30% context (don’t wait for limits)
Use /clear + /catchup pattern for simple restart
Use “Document & Clear” for complex tasks
Document & Clear Pattern:

Have Claude write progress to .md file
/clear the context
Start fresh session reading the .md file
Continue work

#### Documentation Systems

Dev Docs System (from 6_months_hardcore_use, Design_Partner, Building_AI_Factory):

The Three-File Pattern:

~/docs/[task-name]/
├── [task-name]-plan.md      # The accepted plan
├── [task-name]-context.md   # Key files, decisions
└── [task-name]-tasks.md     # Checklist of work

Living Document Approach (from Design_Partner):

Update plan during implementation
Plan documents reveal changed requirements
Check plan is up-to-date before each commit

#### Explore, Plan, Code, Commit
Explore: Read relevant files, images, URLs (explicitly tell it NOT to code yet)
Plan: Use subagents to verify details, create plan with “think” mode
Code: Implement with explicit verification steps
Commit: Update READMEs/changelogs, create PR

#### Specification Documents

Write clear specs before starting

#### Subagents

- Let main agent use Task(...) to spawn clones of itself
- Agent manages own orchestration dynamically
- Avoids gatekeeping context


