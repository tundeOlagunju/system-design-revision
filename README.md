# System Design Revision

One-page, self-testable revision notes for system design questions — generated from
published breakdowns using a Claude Code skill, with the diagrams converted to ASCII so
each note is a single portable Markdown file.

## Structure

```
.
├── skills/                       # reusable Claude Code skills
│   └── system-design-playbook/
│       └── SKILL.md              # the playbook skill (template + rules)
├── <question>/                   # one folder per system design question
│   └── <question>_<resource>.md  # one note per resource/breakdown
└── README.md
```

- **`skills/`** — the Claude Code skills used to produce the notes. Currently holds
  `system-design-playbook`, which fills a fixed one-page revision template.
- **One folder per question** — e.g. `whatsapp/`. A question can have several notes
  over time as it's studied from different sources.
- **One Markdown file per resource** — named `<question>_<resource>.md`, e.g.
  `whatsapp/whatsapp_hello_interview.md` (the WhatsApp breakdown from Hello Interview).

Each note follows the same shape: **Key constraint → Requirements → v1 diagram →
Evolve it (the `➕`/`❌` moves) → final diagram → Decisions → Reusable pattern →
Gotchas**. The idea is to self-test: cover the Decisions and final diagram, then
re-derive them from v1 + the moves.

## Usage

### Use the skill in Claude Code

1. Copy the skill into your Claude Code skills directory:
   ```bash
   cp -r skills/system-design-playbook ~/.claude/skills/
   ```
   (Drop the whole `system-design-playbook/` folder into `.claude/skills/`.)
2. In Claude Code, invoke it with a breakdown link or a pasted breakdown:
   ```
   /system-design-playbook https://www.hellointerview.com/learn/system-design/problem-breakdowns/whatsapp
   ```
3. The skill fetches the source text, then **asks you to paste the source diagrams**
   (it never invents diagrams — paste the v1/simple and final/deep-dive images or
   Excalidraw SVGs). It converts those to ASCII, fills the template, and saves the note.

> **Where notes are saved:** the skill writes each finished note into the **repo root
> of your current working directory** as `<question>/<question>_<resource>.md`, creating
> the question folder if it doesn't exist. Run Claude Code from inside your clone of this
> repo and notes land in the right place automatically — no path editing needed.

### Just read the notes

Browse the per-question folders and open any `.md` — the ASCII diagrams render in any
Markdown viewer or plain text.

## Questions covered

- **[WhatsApp](whatsapp/whatsapp_hello_interview.md)** — source: Hello Interview
