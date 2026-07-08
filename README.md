# claw-kit

**Reusable skill sets for [Claude Code](https://claude.com/claude-code).**

A growing collection of drop-in skills. Each skill lives under `skills/<name>/`
and installs into your global `~/.claude/skills/` directory. Nothing here is
tied to a specific company or repo — you supply your own project context.

> Inspired by the "batteries-included preset" idea (à la LazyVim), but for
> Claude Code's skills system. Own your tooling, keep your secrets local.

---

## Included skills

| Skill | What it does | Roles |
|-------|--------------|-------|
| **orchestrate** | Hierarchical orchestration for **non-dev** work (research, analysis, docs, decisions). Planner → parallel Workers → Reviewer. | 4 (research / analysis / writing / review) |
| **orchestrate-dev** | Hierarchical orchestration for **dev** work (code, debug, test, refactor, review). Same Planner → Workers → Reviewer loop, dev-tuned roles. | 6 (research / code / debug / test / review / refactor) |

Both skills share the same architecture:

```
/orchestrate <project> <task>
        │
        ▼
   Planner (1 agent)  ── decomposes task into N subtasks + parallel groups
        │
        ▼
   Workers (N agents) ── run in parallel groups, each injected with
        │                PROJECT_CONTEXT + its ROLE definition
        ▼
   Reviewer (1 agent) ── merges, verifies, scores → final markdown report
        │
        ▼
   saved to ./orchestration-output/
```

### Design principles

- **Project context is data, not code.** Skills are the engine; your project
  facts live in `projects/<name>.md` (git-ignored by default).
- **Roles are single-responsibility.** A `research` worker only gathers facts;
  it never proposes fixes (that's `analysis`/`code`). This keeps outputs clean
  and prevents scope bleed.
- **Governance built in.** Absolute rules: no external AI-API fallback, no
  writes to production DBs, no file edits / commits / deploys without explicit
  approval. Tune these in each `SKILL.md`.
- **Parallel by default.** Independent subtasks run as concurrent agents.

---

## Install

Skills install into your **global** Claude Code skills directory
(`~/.claude/skills/` — on Windows `C:\Users\<You>\.claude\skills\`).

```bash
git clone https://github.com/bigcomma-kong/claw-kit.git
cd claw-kit

# copy the skill(s) you want
cp -r skills/orchestrate      ~/.claude/skills/orchestrate
cp -r skills/orchestrate-dev  ~/.claude/skills/orchestrate-dev
```

On Windows (PowerShell):

```powershell
Copy-Item skills\orchestrate      "$env:USERPROFILE\.claude\skills\orchestrate"     -Recurse
Copy-Item skills\orchestrate-dev  "$env:USERPROFILE\.claude\skills\orchestrate-dev" -Recurse
```

No restart needed — Claude Code picks up skills on next invocation.

---

## Set up a project

Each skill loads context from `projects/<project-name>.md`, where the
**file name is the `<project-name>` argument**.

```bash
cd ~/.claude/skills/orchestrate/projects
cp example.md myapp.md      # then edit myapp.md with your real project facts
```

Now run:

```
/orchestrate myapp 최근 리서치 자료 정리해서 요약 보고서 만들어줘
/orchestrate-dev myapp UserService NPE 원인 추적하고 수정안 제시
```

> ⚠️ **Never commit real `projects/*.md` files.** They contain server
> addresses, credentials, and known-issue locations. `.gitignore` already
> excludes everything under `projects/` except `README.md` and `example.md`.

---

## Add your own skill

Drop a new folder under `skills/`, following the same shape:

```
skills/<your-skill>/
├── SKILL.md          # frontmatter: name, description, allowed-tools
├── roles/*.md        # optional: role definitions
└── projects/         # optional: per-project context (git-ignored)
    ├── README.md
    └── example.md
```

---

## License

MIT — see [LICENSE](./LICENSE).
