# GAIuS Skills

Claude Code skills from the [GAIuS](https://www.gaius.fyi) team.

## gaius-fit-check

Run an honest, evidence-grounded assessment of whether the GAIuS cognitive engine fits
*your* project — inside your own repo, with no GAIuS install required.

Point Claude Code at your project and run it: the skill explores your codebase, asks a
handful of targeted questions (one at a time, each with a recommended answer), and
delivers a verdict-first assessment — where GAIuS would genuinely help, where it
wouldn't, what adoption would take, and a ready-to-send outreach message if there's a
fit. Its knowledge of GAIuS travels with it: benchmark numbers, shipped use cases, and
the honest no-gos included.

### Install

**Option A — plugin install** (recommended):

```
/plugin marketplace add gaius-fyi/gaius-skills
/plugin install gaius-fit-check@gaius-skills
```

**Option B — copy into a project:**

```bash
git clone https://github.com/gaius-fyi/gaius-skills /tmp/gaius-skills
mkdir -p .claude/skills
cp -r /tmp/gaius-skills/gaius-fit-check/skills/gaius-fit-check .claude/skills/
```

(Copy to `~/.claude/skills/` instead to install for all your projects.)

### Run

In Claude Code, inside the project you want assessed, just ask:

> is Gaius a fit for this project?

or invoke it directly: `/gaius-fit-check:gaius-fit-check` (plugin install) or
`/gaius-fit-check` (copied install).

### What it will and won't do

- Runs inside your existing Claude Code session, so it's meant for repos where you
  already use Claude Code. It sends nothing to us, and nothing beyond what your Claude
  Code session already sends to Anthropic, or to the cloud provider your organization
  runs Claude through.
- If your code is controlled (for example CUI or export-controlled), run it only where
  your organization's Claude arrangement is approved for that code.
- The skill is four Markdown files (plus two small JSON plugin manifests): no scripts, no
  binaries, nothing that executes. It never sends anything on your behalf; if there's a
  fit, it drafts a message for you to send yourself.
- It is instructed to ground every claim about GAIuS in its bundled reference files, and
  to say so when a question falls outside them instead of inventing a capability.
- A clear "not a fit, because X" is a supported outcome. The honesty is the point.
