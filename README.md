# Blyze Labs Skills

A library of [Claude skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that productize Blyze Labs' fractional data-analytics methodology. Each skill packages a
repeatable workflow — an audit, a build process, a recurring deliverable — so it can be
run consistently across clients inside Claude (Cowork, Claude Code, or the desktop app).

**Maintained by:** Blyze Labs · [blyzelabs.co](https://blyzelabs.co) · agus@blyzelabs.co

---

## What's a skill?

A skill is a folder containing a `SKILL.md` file (instructions Claude loads on demand)
plus optional reference files, scripts, and assets. When a skill is installed, Claude
reads its name and description, and pulls the full instructions into context whenever a
task matches. This turns our internal playbooks into something Claude can execute
repeatably — same methodology, same output structure, every client.

See Anthropic's [Agent Skills documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
for the underlying mechanics.

---

## Skills in this repo

| Skill | Purpose | Status |
|---|---|---|
| [`data-architecture-audit`](./data-architecture-audit) | Full data-stack audit for retail/e-commerce brands — scans dbt, ETL, BI, and warehouse; produces a scored report with a prioritized action-item roadmap. | ✅ v1.0 |

> More skills (data-quality review, client onboarding, recurring reporting) will be added
> here over time. One repo, one install, the whole library.

---

## Repository structure

```
blyzelabs-skills/
├── README.md                      ← you are here
├── data-architecture-audit/
│   ├── SKILL.md                   ← skill entrypoint (name + description + workflow)
│   ├── references/                ← detailed docs Claude loads as needed
│   └── assets/                    ← templates and example deliverables
└── <future-skill>/
    └── SKILL.md
```

Each skill is self-contained in its own folder. Conventions we follow:

- **Folder name = skill name** (kebab-case), matching the `name:` in its `SKILL.md` frontmatter.
- **`SKILL.md` stays lean** (a quick-start + principles). Heavy detail goes in `references/`.
- **`assets/`** holds output templates and anonymized example deliverables.
- **No client data in this repo.** Methodology only — never a client's actual repo, data,
  or credentials.

---

## Using a skill

### In Claude Cowork / Claude Desktop
1. Clone or download this repo.
2. Add the skill folder to your Cowork capabilities (Settings → Capabilities → Skills),
   or point your project at this repo.
3. Start a session and describe the task — Claude loads the matching skill automatically.

### In Claude Code
Place the skill folder under your project's skills directory (or a path Claude Code is
configured to read), then invoke Claude normally on a matching task.

### Quick start: running an audit
```
Open a Cowork session with the data-architecture-audit skill available, then:
"I'm starting a data architecture audit for [client]. Here's read-only access
 to their dbt repo / warehouse / BI tool — let's begin with stack discovery."
```

---

## Contributing a new skill

1. Create a new folder: `your-skill-name/`.
2. Add a `SKILL.md` with `name` and `description` frontmatter. Make the description
   specific about **when** to trigger — that's what Claude matches against.
3. Keep `SKILL.md` under ~500 lines; push detail into `references/`.
4. Add an entry to the **Skills in this repo** table above.
5. Test it on a real task before merging. Iterate on the description if it under-triggers.

For building and refining skills, use Anthropic's skill-creator tooling.

---

## Conventions

- **Branching:** feature branches off `main`; PR + review before merge.
- **Versioning:** bump the `Version` line in a skill's `SKILL.md` on meaningful changes.
- **Privacy:** this repo is **private**. It encodes Blyze Labs IP. Never commit client
  data, credentials, or proprietary client logic.

---

*© Blyze Labs. Internal methodology — confidential.*
