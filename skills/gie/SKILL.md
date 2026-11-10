---
name: gie
description: Use when the user wants an interactive HTML explainer page for a product, feature, or system design, or asks for "gie" / "GIE" (Generate Interactive Explainer). The page is a single file with three tabs (Core idea, Data model, Scenarios), an animated clickable component map, a clickable ERD, and step-through sequence diagrams with a device mockup and a live database/Redis board. Triggers include "explain my product as HTML", "architecture explainer", "interactive design doc", "sequence diagram page".
---

# GIE (Generate Interactive Explainer)

Generates a single self-contained HTML file that explains a product the way an engineer walks a colleague through a design: three tabs, each interactive.

1. **Core idea**: prose, a "where each component lives" list, and a component map whose arrows animate. Clicking a component dims everything unrelated and shows its note and links.
2. **Data model**: a clickable ERD. Clicking a table highlights the tables it links to and shows per-table design rationale.
3. **Scenarios**: a left rail of scenarios grouped by section. Each scenario is a step-through sequence diagram (clickable numbered arrows, dashed async arrows, dashed transaction boxes), a phone or web mockup that follows the steps, a step-detail card with the SQL/HTTP/Redis payload, and a database plus Redis board that replays row state per step (insert green, update yellow, read blue).

## Workflow

1. **Gather the product.** From the user's description collect: entities/tables (columns, keys, relations, rationale), the flows worth showing as scenarios, the infrastructure actually mentioned (database, Redis, RMQ/Kafka, push, external APIs; only what the product uses), and whether the product is mobile (`device: 'phone'`) or web (`device: 'web'`). Ask only if the data model or flows are genuinely unknowable from the prompt; otherwise infer a sensible design and say so.
2. **Copy the template.** Copy `template/explainer.html` from this skill's directory to the output location. All content lives in one JS data block between `/* DATA START */` and `/* DATA END */`. Never rewrite the machinery below the markers.
3. **Replace the data block.** The template ships with a small sample product (Orderly) that demonstrates every field. Replace `PRODUCT`, `ARCH`, `ERD_CAPTION`, `TABLES_DATA`, `DB_BASE`, and `SCENARIOS` with the user's product. Field-by-field schema: read `reference/data-format.md` in this skill's directory before writing the data.
4. **Verify.** Extract the script and run `node --check`. Then render headless if Chrome is available:
   `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --dump-dom --virtual-time-budget=3000 "file://<path>"` and confirm the step count, ERD tables, and arch nodes appear. Open the file for the user.

## Quality bar (what makes the output good)

The example this skill is modeled on is dense. Match it:

- **Every table** gets real columns with `PK`/`FK`/`UQ` badges, a `rel` list, and 2 to 5 `why` bullets that explain design decisions (sharding, indexes, idempotency, why a column exists), not column restatements.
- **Every scenario** gets a `desc` that summarizes the whole flow and its outcome, and steps where `d` explains *why* the call happens and what it does NOT do, and `code` shows the literal payload: the HTTP request with JSON body, the SQL statement, the Redis command, or the response status with body.
- **Transactions**: wrap multi-write commits in `txStart`/`txEnd` with a `txLabel`. Separate databases get separate boxes.
- **Async**: returns, worker publishes, queue consumes, and pushes are `async: true` (dashed).
- **The board must move**: every step that writes or reads data carries `db:` / `redis:` ops so the board animates. A scenario where the board never changes is underspecified.
- **The device must follow**: steps where the user sees something carry `ui:` with concrete screen items (what the user actually taps and reads). Backend-only scenarios set `device: null` on the scenario.
- Use consistent concrete ids across the whole page (the same user id, room/order id, and seq in prose, ERD rationale, scenarios, and board rows) so the reader can follow one story.
- Lanes: person's app first, then APIs, then Redis/cache, then tables in touch order, then queue/external. 5 to 9 lanes per scenario; make a lane only for things that send or receive arrows.
- Include failure and edge scenarios (duplicate submit, permission denied, retry), not only happy paths. Mark speculative ones with `review: true`.

## Common mistakes

- Editing the machinery instead of the data block. Don't; it is verified as-is.
- Table names in `lanes` must exactly match `DB_BASE` keys, or lanes lose their blue table styling and the board focus.
- `db` op `match` values are compared as strings; keep board row values strings.
- Forgetting `seed`/`seedRedis` in scenarios that assume earlier state, so the board starts empty where the desc says a row exists.
- HTML is allowed in `d`, `why`, `desc`, `note`, and `html` fields (use `<code>`), but labels and `code` are escaped as plain text.
