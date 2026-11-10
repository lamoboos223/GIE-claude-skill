# GIE (Generate Interactive Explainer)

A Claude Code skill that turns a product or system description into a single, self-contained, interactive HTML explainer. Describe your product in a prompt; Claude loads the `gie` skill and generates a page an engineer can step through like a guided design review.

## What the generated page looks like

Three tabs, all interactive, one HTML file, no dependencies:

### 1. Core idea
- Prose blocks that explain the design.
- A "where each component lives" layer list.
- A component map with animated (moving-dash) arrows. Click a component: everything unrelated dims, its neighbors and edges highlight, and a detail card shows its note and links.

### 2. Data model
- A clickable ERD: every table with its columns and `PK` / `FK` / `UQ` badges.
- Click a table: the tables it links to highlight, and a detail card shows the design rationale (indexes, sharding, idempotency, why each column exists).

### 3. Scenarios
- A left rail of scenarios grouped by section (foldable).
- Each scenario is a step-through sequence diagram: clickable numbered arrows, dashed async arrows, amber dashed transaction boxes, blue lanes for database tables.
- A phone or web-browser mockup that follows the steps and shows what the user sees; it dims when the current step is not on that screen.
- A step-detail card with the literal payload: HTTP request/response, SQL statement, or Redis command.
- A **Database and Redis board** that replays row state at every step: insert is green, update is yellow, read is blue.

## Install

Copy the skill folder into your user skills directory:

```bash
cp -R gie ~/.claude/skills/gie
```

New Claude Code sessions list it automatically.

## Use

In any Claude Code session, describe the product and ask for the explainer:

> Use the gie skill. My product is a food-delivery app: customers place orders, a courier accepts, payments go through Stripe. We use Postgres, Redis for idempotency, and RabbitMQ for courier dispatch. Generate the explainer HTML.

Claude will:
1. Derive the tables, component map, and scenarios from your description (only the infrastructure you actually mention).
2. Copy `template/explainer.html` and replace the single data block between `/* DATA START */` and `/* DATA END */`.
3. Verify the result (`node --check` plus a headless Chrome render) and open it.

The more you describe (entities, flows, failure cases, external dependencies, mobile vs web), the denser the page. The skill also pushes Claude to add edge-case scenarios (duplicate submit, permission denied, retry) and to keep one consistent story: the same user ids and record ids across prose, ERD, diagrams, and board rows.

## Repository layout

```
gie/
  SKILL.md                  # skill entry point: workflow + quality bar
  template/explainer.html   # the full page machinery, data-driven; ships with a sample product
  reference/data-format.md  # field-by-field schema for the data block
README.md
```

## How it works

All page content lives in six JS constants (`PRODUCT`, `ARCH`, `ERD_CAPTION`, `TABLES_DATA`, `DB_BASE`, `SCENARIOS`) inside one marked data block. The rendering machinery below the markers never changes, so generation is reliable: Claude writes data, not UI code. Open `gie/template/explainer.html` directly in a browser to see the built-in sample product (Orderly) that exercises every feature.

Supports light and dark mode, reduced motion, keyboard navigation on clickable diagram elements, and works offline.
