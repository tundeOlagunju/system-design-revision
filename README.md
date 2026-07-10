# System Design Revision

One-page, self-testable revision notes for system design questions. Each note is a
single portable Markdown file with the diagrams rendered as ASCII, generated from a
published breakdown using the `system-design-playbook` Claude Code skill.

## Structure

```
.
├── .claude/
│   └── skills/
│       └── system-design-playbook/
│           └── SKILL.md          # the playbook skill (template + rules)
├── <question>/                   # one folder per system design question
│   └── <question>_<resource>.md  # one note per resource/breakdown
└── README.md
```

- **`.claude/skills/`** — Claude Code skills used to produce the notes. Holds
  `system-design-playbook`, which fills a fixed one-page revision template.
- **One folder per question** — e.g. `whatsapp/`. A question can hold several notes as
  it's studied from different sources.
- **One Markdown file per resource** — named `<question>_<resource>.md`, e.g.
  `whatsapp/whatsapp_hello_interview.md` (the WhatsApp breakdown from Hello Interview).

Every note follows the same shape: **Key constraint → Requirements → v1 diagram →
Evolve it (the `➕`/`❌` moves) → final diagram → Decisions → Reusable pattern →
Gotchas**. Study by self-testing: cover the Decisions and final diagram, then re-derive
them from v1 + the moves.

## Usage

### Generate a note with the skill

1. Clone the repo and start Claude Code inside it. The `system-design-playbook` skill is
   available as a project skill.
2. Invoke it with a breakdown link (or a pasted breakdown):
   ```
   /system-design-playbook https://www.hellointerview.com/learn/system-design/problem-breakdowns/whatsapp
   ```
3. The skill reads the source, then asks you to paste the source diagrams — it never
   invents diagrams. Paste the v1/simple and final/deep-dive images or Excalidraw SVGs.
4. It converts the diagrams to ASCII, fills the template, and writes the note to
   `<question>/<question>_<resource>.md`, creating the question folder if needed.

To use the skill across all your projects, copy it into your personal skills directory:

```bash
cp -r .claude/skills/system-design-playbook ~/.claude/skills/
```

### Read the notes

Open any `.md` under a question folder — the ASCII diagrams render in any Markdown
viewer or plain text.

## Questions covered

- **[WhatsApp](whatsapp/whatsapp_hello_interview.md)** — source: Hello Interview
