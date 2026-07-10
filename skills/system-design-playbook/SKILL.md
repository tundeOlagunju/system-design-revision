---
name: system-design-playbook
description: Fill a one-page system-design revision note for a given system (or a pasted breakdown of one) using a fixed template. Use when the user asks to "make a playbook entry", "fill the system design template", or names a system to study (e.g. "URL shortener", "design Dropbox"). Produces terse study notes, NOT interview scripts or documentation. Do not redesign the template — apply it exactly.
---

# System Design Playbook

Apply the template below **exactly** as specified. When the user names a system (or
pastes a written breakdown of one), fill this template. Do NOT redesign the template
or propose alternatives — just apply it. Output terse, one-page, revision-note style.

Input is usually a LINK to a system-design breakdown. When the user posts a URL:
fetch it with WebFetch (or an available fetch tool) and convert the page into the
template. If fetching fails (e.g. internet mode disabled, auth wall, JS-rendered
page), say so explicitly and ask the user to paste the page content — do NOT
reconstruct the breakdown from memory and present it as the source's design.

When converting a fetched or pasted breakdown, map: their FR/NFR → Requirements,
their high-level design → v1 + first `➕` moves, their deep-dives → the `❌` breaks.
Stay faithful to their tech choices; don't substitute your own.

## Workflow (follow in ORDER — do not skip step 2)

1. **Fetch/read the source** (URL or pasted text) so you have the breakdown text.
2. **ASK THE USER TO PASTE THE DIAGRAMS FIRST — before you generate ANY of the
   note.** Do not output the filled template, not even the text sections, until the
   user has pasted the diagrams. Request **at least two** (v1/simple and
   final/deep-dive; optional third key-idea) per the "Diagrams" section below. If the
   user pastes only one, ask for the other. If the user explicitly says there are no
   diagrams / to skip them, proceed and mark the diagram sections
   `<no diagram provided>`.
3. **Generate the full note** (all template sections + the pasted diagrams converted
   to ASCII).
4. **SAVE the note to a file** (see "Saving the note" below) — always, without being
   asked. Then show the note in the reply and state the saved path.

DIAGRAMS ARE DIFFERENT: never generate them yourself. Ask the user to paste the
diagrams (see "Diagrams" section below) and convert those to ASCII. The text of the
breakdown can come from a link/paste; the diagrams must come from the user.

## Saving the note (always do this)

Every completed note is saved as a Markdown file — do it automatically, don't wait
to be asked.

- **Base folder = the current working directory's repo root** (never a hardcoded
  personal path). Find it by running `git rev-parse --show-toplevel`; if that fails
  (not a git repo), fall back to the current working directory. This is wherever the
  user cloned/opened their system-design notes repo.
- **Per-question subfolder:** one folder per system design question, named in
  kebab-case (e.g. `whatsapp/`, `url-shortener/`). Create it if missing (`mkdir -p`) —
  a question can hold multiple notes over time.
- **Filename:** `<question>_<resource>.md`, snake_case, e.g.
  `whatsapp_hello_interview.md`, `url_shortener_bytebytego.md`. `<question>` matches
  the folder name; `<resource>` is where the breakdown came from (site/author); if
  unknown, use `notes`.
- Full path example: `<repo-root>/whatsapp/whatsapp_hello_interview.md`
- `mkdir -p` the subfolder, then write the exact note (including the ASCII diagrams)
  to that file with the Write tool, then tell the user the path. If a file with that
  name already exists, pick a non-colliding name (append `_2`, etc.) rather than
  overwriting, unless the user asked to update it.

## The template

```
# <System>                                    source: <> · date: <>

## Key constraint 🔑
The ONE sentence everything hangs off. Pick the fact that forces the biggest
decision (read:write skew, latency floor, large blobs, fan-out, or an invariant
that must never break).

## Requirements
FR — what it does:
- <core feature — the one that defines the system>
- <secondary features>              (parked: <out of scope>)
NFR — the pressures, with numbers:
- <driver>: <target>   ← the one that shapes the design
- scale / latency / consistency / availability: <>

## Diagram — v1 (simple)          ← ASK USER TO PASTE; convert to ASCII faithfully
<the simple/v1 diagram the user pasted, converted to ASCII — never invented>

## Evolve it   (text only — the moves, v1 → final)
- v1:  <the simple design above>
- ➕ <feature>  → <how the design changes>      (build up the FRs)
- ❌ <break>    → <response>                     (answer a pressure)
- stop when: <all FRs built + NFR targets met>

## Diagram — key idea    (OPTIONAL — only if the user pasted a before→after)
<the pasted key-idea diagram, converted to ASCII>

## Diagram — final               ← ASK USER TO PASTE; convert to ASCII faithfully
<the full/final diagram the user pasted, converted to ASCII — never invented>

## Decisions (choice · why · alt rejected)
- <decision> → <X> · <why chosen> · <alt: short kill> · <alt: short kill>

## Reusable pattern ♻️   (fill LAST — what transfers to the NEXT question)
- when I see <shape>, reach for <move>

## Gotchas   (only what's NOT already obvious above; each solved or parked)
- …
```

## Rules (load-bearing — follow them)

1. **FR → v1, NFR → the breaks.** v1 is the dumbest thing that makes the CORE feature
   work; ignore scale. Every FR must be representable in the evolution — if a feature
   can't appear, it's out of scope (park it).

2. **"Evolve it" grows the design on TWO axes:**
   - `➕` = add a capability (build up remaining FRs)
   - `❌` = something pushes back. Exactly THREE kinds of break — LABEL each:
     - `load`  → too slow      → cache / shard / replicate / CDN / async
     - `crash` → a box dies    → replicate / failover / lease / degrade gracefully
     - `race`  → data corrupts → lock / idempotency / version / sequence numbers
   - "Failure modes" are just the `❌ (crash)` steps — inline, NOT a separate section.

3. **A `❌` can only break a box an EARLIER step introduced.** If a break references a
   component never drawn, add the missing box or the break is invalid. Same on the
   input side: name the genuinely dumbest naïve choice (a local counter variable, bytes
   through the app server), and let a break pull in the real component. Never import the
   scaled answer into v1 (no CDN / Redis / Cassandra in v1).

4. **Tech-choice timing.** In v1, name a box generically ("a store", "a queue"). COMMIT
   the specific tech (Cassandra vs Postgres, SSE vs WebSocket) at the step where a
   pressure makes the choice obvious — usually the shard/scale break. You may commit the
   moment you can point at the break that eliminates the alternative, not before. Flag
   the access pattern early, commit late.

5. **Decisions = the DEFENSE** (why this, not the alternative, + the cost). Keep it
   COMPACT: `choice · why · alt: short kill · alt: short kill`. Only decisions with 3+
   real contenders get alternatives spelled out (transport, SQL-vs-NoSQL,
   optimistic-vs-pessimistic locking, sync-vs-async). Obvious moves (add a cache, add an
   LB) don't need defending. Every Decision must trace back to a `➕`/`❌` move.

6. **Gotchas:** each item is SOLVED above (tag "(solved)") or PARKED. Nothing dangles.
   Only list traps not already obvious from the evolution.

7. **Reusable pattern: fill LAST.** Transferable lessons ("when I see shape X, reach for
   move Y"). This is the point of the playbook — patterns transfer, specific systems don't.

8. **Keep Evolve it (moves) and Decisions (defense) SEPARATE.** Do not merge the
   reasoning into the moves — separation lets the user self-test (cover Decisions,
   re-derive the "why" from the moves).

## Diagrams — NEVER invent them; convert what the user pastes

**Hard rule: do NOT generate, imagine, or design diagrams on your own.** The diagrams
must come from the user. **Ask for the diagrams FIRST — before generating any part of
the note (see Workflow step 2), not after.** ASK the user to paste **at least two**
diagrams from the source (image or Excalidraw SVG — SVG is text and can be read
directly):

1. the **simple / v1** diagram (the naïve or high-level design), and
2. the **full / final** diagram (the fully evolved / deep-dive design).

(An optional third — the key-idea before→after — only if the source has one.)

Do not proceed to draw the diagram sections until the user has pasted them. If the
user pastes only one, ask for the other. Never substitute a diagram you made up.

To convert a pasted diagram: read it (for an SVG file, Read the file — parse the
box labels, the arrow endpoints/directions, dashed containers, and overlapping/stacked
boxes), then reproduce it **faithfully** in ASCII. Preserve every box, every arrow and
its direction, double-headed arrows, dashed groupings, and stacked boxes (render stacks
as `×N`). Reproduce text labels verbatim. You may straighten a spaghetti layout into a
clean left→right / top→bottom flow, but never add, drop, or invent a box or edge.

### Node shapes — everything is a box; brokers are a long/wide box

```
Regular node (service, DB, cache, client)   Message broker / queue / stream
┌──────────┐   ┌──────────┐                 ┌──────────────────────────┐
│   App    │   │ Database │                 │          Kafka           │
└──────────┘   └──────────┘                 └──────────────────────────┘
 normal-width box for services,              SAME box style, just drawn
 stores, caches, clients, actors            long/wide (a bar) to signal
                                            a broker/queue/stream
```

- **Every node is a square-cornered box** (`┌─┐ │ └─┘`) — services, DBs, caches,
  clients, actors, and brokers alike.
- **Message brokers / queues / streams** (Kafka, Kinesis, SQS, pub/sub) are the SAME
  box, just drawn **long/wide** (a bar-shaped box) so they stand out as the messaging
  layer. Still a full box with borders — not open rails.

```
arrows:  →  sync/request      ══▶ / ⇒  pushed/async     ◄──►  double-headed
label the arrow with what flows ("publish", "poll", "read", "write", "fix records")
```

### STRICT ASCII rules (the conversion MUST follow all of these)
- **Monospace alignment only.** What you type is what renders — align by hand.
- **Arrows connect box EDGES, centred on the box** (a horizontal arrow leaves the
  right edge of one box and enters the left edge of the next at the box's mid-height
  row; a vertical arrow leaves the bottom/top at the box's horizontal centre).
- **One arrow = one clean line** — `─`/`│` for sync, `═`/`║` (`══▶`) for async — with
  a single head (`►`/`▼`/`▲`/`◄`). Double-headed = a head on each end.
- **Arrow labels go ABOVE a horizontal arrow or to the RIGHT of a vertical arrow** —
  short (1–3 words). Never let a label overlap a box or another arrow.
- **Keep box labels short**; wrap onto a second line inside the box rather than widening.
- **Boxes on the same row share the same height**; leave ≥2 spaces of gap between a box
  and any arrow/label so nothing touches.
- **Long explanations go in a `>` note line under the diagram**, never inside a box or
  on an arrow.
- **Group labels** (dashed containers) become a titled box `┌─ Title ──┐ … └────────┘`
  with the title ≤ 2–3 words; put contained boxes inside it.
- Draw at most THREE diagrams: v1, optional key-idea, final. Never one per step.

Before returning, re-read your ASCII: every arrow must start and end on a box edge,
nothing may overlap, and the picture must match the pasted source box-for-box.

## Style

- Terse, one page, revision-note style. For self-study, NOT an interview script or docs.
- The Key constraint is the fact that forces the biggest decision (e.g. for FB Live
  Comments it's the read/write IMBALANCE — which is what makes SSE beat WebSocket — not
  just "fan-out").
- The end goal is a note the user can self-test from: v1 + the moves are the prompt, the
  final diagram + Decisions are the answer to reproduce from memory.
