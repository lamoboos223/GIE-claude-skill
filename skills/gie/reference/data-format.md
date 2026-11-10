# Data format for template/explainer.html

Everything sits between `/* DATA START */` and `/* DATA END */` in the template's single `<script>`. Six constants. Strings marked (html) may contain inline HTML such as `<code>`, `<b>`; everything else is escaped.

## PRODUCT

```js
const PRODUCT = {
  title: 'Page title (h1 + document.title)',
  subtitle: 'One-line summary under the h1',          // (html)
  device: 'phone',        // 'phone' | 'web' | null. Default device for scenarios.
  actor: 'Alex',          // whose screen the mockup shows
  url: 'app.example.com', // web device only: text in the browser address bar
  helpText: '...',        // (html) "How to read this" in the scenario rail
  concept: [ { h2: 'Heading', html: '<p>...</p>' } ],  // Core idea prose blocks
  layers: [ { name: 'Main DB', html: '<code>Users</code>, ...' } ], // "Where each component lives"
  archCaption: 'Click a component to see what it talks to.'  // (html)
};
```

## ARCH (component map, Core idea tab)

```js
const ARCH = {
  nodes: [
    { id: 'api',             // unique, referenced by edges
      label: 'Orders API',
      kind: 'backend',       // frontend | backend | database | cache | queue | external (sets border color)
      row: 1,                // vertical layer, 0 = top; nodes in a row spread evenly
      note: '...' }          // (html) shown in the detail card when clicked
  ],
  edges: [
    { from: 'app', to: 'api', label: 'POST /orders' }  // animated arrow; label optional
  ]
};
```

Clicking a node highlights it, its neighbors, and its edges; everything else dims. Arrows animate always (disabled under prefers-reduced-motion).

## TABLES_DATA (ERD, Data model tab) and ERD_CAPTION

```js
const ERD_CAPTION = '...'; // (html) paragraph above the ERD
const TABLES_DATA = [
  { name: 'Orders',                 // must match DB_BASE key and lane name
    note: 'one row per order',      // small header caption
    rel: ['Users', 'OrderEvents'],  // highlighted when this table is clicked
    cols: [
      ['PK', 'OrderId', 'integer'],   // badge: 'PK' | 'FK' | 'UQ' | ''
      ['FK', 'UserId', '→ Users'],
      ['',   'Status', 'placed | paid']
    ],
    why: [ 'bullet one ...', 'bullet two ...' ] }  // (html) rationale bullets in the detail card
];
```

## DB_BASE (board baseline)

```js
const DB_BASE = {
  Users: { cols: ['UserId', 'Name'],                      // column order on the board
           rows: [ { UserId: '1001', Name: 'Alex' } ] },  // rows before step 1; all values strings
  Orders: { cols: ['OrderId', 'UserId', 'Status'], rows: [] }
};
```

Keys of `DB_BASE` define which lane names render as blue table lanes and which cards the board shows. Missing/empty cell renders as muted `null`.

## SCENARIOS

```js
const SCENARIOS = [
  {
    section: 'Orders',        // rail group heading
    title: 'Place an order',
    desc: '...',              // (html) summary of the whole flow and outcome
    review: false,            // true shows the "Needs review" flag
    device: undefined,        // omit = PRODUCT.device; 'phone' | 'web' | null (null hides the mockup)
    actor: 'Alex',            // optional override of PRODUCT.actor
    lanes: ["Alex's app", 'Orders API', 'Redis', 'Orders', 'Outbox', 'RMQ'],
    seed: { Orders: [ {...} ] },   // optional: replaces DB_BASE rows for this scenario only
    steps: [ /* see below */ ]
  }
];
```

### Step

```js
{ f: "Alex's app",        // from lane (exact lane string)
  t: 'Orders API',        // to lane; or self: true for a self-call
  label: 'POST /orders',  // text on the arrow (plain)
  d: '...',               // (html) explanation in the step card: why, and what does NOT happen
  code: 'POST /orders\n{ "id": 1 }',  // literal payload: HTTP, SQL, Redis cmd, or response; rendered in <pre>, escaped
  async: true,            // dashed arrow: returns, pushes, queue hops, worker activity
  self: true,             // self-loop on lane f (omit t)
  txStart: true, txLabel: 'db transaction',  // opens a dashed box around the table lanes
  txEnd: true,            // closes it (a single-step tx sets both)

  // Board ops, applied cumulatively in step order. Flags on the CURRENT step color rows:
  db: [
    { table: 'Orders', op: 'insert', row: { OrderId: '7001', Status: 'placed' } },        // green
    { table: 'Orders', op: 'update', match: { OrderId: '7001' }, set: { Status: 'paid' } }, // yellow
    { table: 'Orders', op: 'read',   match: { OrderId: '7001' } },                          // blue, this step only
    { table: 'Orders', op: 'delete', match: { OrderId: '7001' } }
  ],
  redis: [
    { op: 'set', key: 'order:1001:idem:91', value: 'received', ttl: 'config' },  // insert or update
    { op: 'get', key: 'order:1001:idem:91' },   // blue read flag
    { op: 'del', key: 'order:1001:idem:91' }
  ],
  seedRedis: [ { key: '...', value: '...' } ],  // keys that exist before step 1 (any step may declare them)

  // Device screen. If present, the mockup lights up and renders these items;
  // if absent, the mockup keeps the previous screen and dims.
  ui: {
    mode: 'foreground',   // 'foreground' | 'background' (dim caption) | 'away' (lock screen; banner items still show)
    items: [
      { type: 'header', title: 'Checkout', sub: '1 item', back: true },
      { type: 'row', av: 'A', color: '#7c3aed', title: 'Weekend trip', sub: 'meet at 6', badge: '2', on: true },
      { type: 'bubble', text: 'hello', out: true, sys: false, meta: '10:21 ✓✓' },
      { type: 'note', text: 'muted helper text' },
      { type: 'btn', label: 'Place order', primary: true, soft: false },
      { type: 'input', text: 'Type a message', send: false, locked: false },
      { type: 'banner', app: 'Orderly', text: 'Order confirmed' },  // notification banner
      { type: 'toast', text: 'Saved' }
    ]
  }
}
```

### Board semantics

- State replays from `DB_BASE` (or scenario `seed`) through every step's ops up to the selected step, so clicking any arrow shows the exact rows as of that moment.
- `match` compares with `String(...)` equality on every named column.
- The Redis card appears only if any step in the scenario has `redis` or `seedRedis`.
- A card gets the accent outline when the current step's ops touch it or the current arrow's lane is that table.
