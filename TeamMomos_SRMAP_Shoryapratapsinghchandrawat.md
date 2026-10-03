# FlipTrace

A decision reversal detector that lives in one HTML file.

You know how someone says "let's go with React" on Monday, "actually, Vue" on
Wednesday, and by next week they're back on React and nobody remembers agreeing
to any of it? FlipTrace eats your chat logs, email threads and task board
exports, pulls out every decision, and flags the things that keep getting
un-decided. It gives each topic a 0–100 Flip-Flop Score and each person a
stability profile, which is either useful analytics or a weaponised guilt trip
depending on your team culture.

No server. No account. No network calls. Open the file, it works. Close it,
your data is still there (IndexedDB, with a localStorage fallback for browsers
that lock storage down on `file://` origins — more on that below).

## Quick start

1. Open `index.html` in a modern browser (Chrome, Firefox, Edge — anything
   from the last few years).
2. Click **Load demo data**. Three people, three months, four sources, plenty
   of regret. Watch the backend console while it ingests if you don't believe
   it's doing real work.
3. Or go to **Import** and paste a WhatsApp export, an email thread, a task
   CSV, or just lines like `Sam: let's do React`. Format detection is automatic.
4. Press `Ctrl+K` (or `Cmd+K`) to jump anywhere. Press `?` for all shortcuts.

## What it actually does

The pipeline, in order:

**Ingest.** Parsers for WhatsApp-style txt exports, Slack-style JSON, raw
`.eml`/mbox text, and CSV/JSON task board dumps. Email parsing strips quoted
replies (lines starting with `>`, the "On ... wrote:" headers) and signatures.
Task history events — status changes, due date moves, delete-and-recreate —
become decision records, because moving a due date four times absolutely is a
decision you keep reversing.

**Extract.** Each message gets split into sentences. Every sentence is scored
against a phrase lexicon: commitment phrases ("locked in", "final answer",
"we're going with"), reversal phrases ("actually", "scratch that", "back to"),
rejection phrases, and hedges ("maybe", "probably") that pull confidence down.
A pile of regexes then pulls out the *option* being chosen — "React" out of
"we're going with React for the frontend". The output is a decision: who, when,
polarity (commit / switch / reject / revert), option, confidence 0–1, and the
exact phrases that triggered it. You can edit every phrase in Settings and the
whole corpus re-scores on the spot.

**Group.** Decisions made weeks apart need to land in the same topic. Each one
becomes a TF-IDF vector (tokenised, stop-worded, lightly stemmed — the stemmer
is four if-statements, not Porter, and that's deliberate). New decisions join a
topic when cosine similarity plus Jaccard overlap plus an "option vocabulary"
affinity clear a threshold. The vocabulary matters: React/Vue/Svelte and
keto/vegan/carnivore are strong hints that two decisions belong together.
Grouping wrong? Merge or split topics by hand. Both are undoable.

**Detect.** Within a topic, decisions are ordered by time. A reversal fires
when the chosen option changes, or a commit is followed by a rejection, inside
a configurable window (45 days by default). A **boomerang** — A → B → A, going
back to something already tried — is the loudest signal and gets its own type.
Task progress (backlog → in progress → done) is filtered out on purpose;
that's work, not wobble.

**Score.** Flip count (30%), boomerangs (25%), flip speed (15%), oscillation
(15%), decision entropy (10%) and confidence drift (5%) blend into the
Flip-Flop Score:

    0–19   Stable
    20–39  Wobbly
    40–59  Restless
    60–79  Chaotic
    80+    Full Pendulum

**Alert.** When a topic crosses its Nth reversal (default: 2nd), an alert
fires with the source sentence highlighted, who said it, when, and which rule
matched. Thresholds are in Settings.

## The "backend" situation

There's no server, but the file is organised like there is one, and the UI is
held to it strictly — no page ever touches the database directly:

- **Database layer** — IndexedDB wrapper with seven stores (sources, messages,
  decisions, topics, reversals, alerts, settings), indexes on topic/timestamp/
  person, a versioned migration hook, and a localStorage fallback for the
  browsers that refuse IndexedDB on `file://`.
- **API layer** — a tiny router that mimics REST: `api.get('/topics')`,
  `api.post('/ingest')`, `api.put('/settings')`. All UI traffic goes through
  it.
- **Worker** — the analysis engine runs in a Web Worker built from a Blob URL,
  streaming progress events back so the ingest screen shows live progress.
  If the environment won't give us a worker, the same code runs on the main
  thread through a `new Function` shim. Identical behaviour, slightly worse
  frame rates.
- **Services** — IngestService, ExtractionService (lives in the engine),
  TopicService, ReversalService, ScoringService, AlertService, ExportService.
  This is where the business rules live.
- **Event bus** — plain pub/sub so pages refresh themselves when data lands.
- **Backend console** — bottom-right button. Every API call, DB write, worker
  message and bus event, timestamped, filterable. Open it during a demo load
  and watch the whole pipeline work.

## The pages

Eleven of them: hero, dashboard (stat counters, an instability "ECG" whose
beat tracks the data, leaderboard), a zoomable/pannable timeline where
reversals swing as animated arcs, topic explorer (decision chain, animated
state diagram, score breakdown), a force-directed pattern map with hand-rolled
physics (unstable topics physically fidget), a GitHub-style heatmap, person
profiles with a radial gauge and behaviour radar ("Night-time flipper",
"Monday reverser" — those tags are computed from actual flip timestamps), the
import center, the alert feed, a "How it works" page with four scripted
canvas "clips" driven by a small keyframe engine, and settings.

Also in there: command palette with fuzzy search, an onboarding tour,
undo/redo for topic merges, three themes (Midnight / Aurora / Ember), a
live-feed simulator that pipes scripted messages through the real pipeline
every few seconds so you can watch alerts fire, and export to JSON, CSV, or a
printable report. `prefers-reduced-motion` is respected — things calm down
rather than break.

## How it was made

Honestly: not in one giant file. The single-file constraint is brutal for
maintainability, so the project is split into readable modules in `src/` —

    src/engine.js    the analysis engine (pure functions, no DOM)
    src/demo.js      the hand-written demo dataset
    src/worker.js    worker glue around the engine
    src/style.css    the design system
    src/body.html    the page skeleton
    src/app1..6.js   utils → bus/console → db → api/services → ui kit →
                     charts/particles/clips → pages → router → boot
    src/test.js      engine tests against the demo corpus
    build.js         concatenates everything into index.html

`build.js` inlines the CSS and the modules into one file, and embeds the
engine + worker glue as a JSON string that becomes the Blob worker. Running
`node build.js` rebuilds `index.html` and syntax-checks the assembled script.
`node src/test.js` runs the pipeline on the demo corpus and asserts the
important detections (framework choice must come out Chaotic, the job-offer
thread must produce a boomerang, hosting must stay Stable, and so on).

The engine being DOM-free is what makes it testable — it runs identically in
the worker, in the main-thread fallback, and in Node for the test suite.

The demo data was written by hand, sentence by sentence, with deliberate
patterns baked in: Sam can't pick a framework or a price, Marcus cycles
through diets, Priya's job-offer saga goes accept → decline → accept. Roughly
half the sentences are noise ("Good luck with that") because an extractor that
only works on clean input is a demo, not a tool.

## Honest limitations

- The phrase lexicons are English-only and tuned on the demo corpus plus
  typical Slack/WhatsApp phrasing. Your mileage will vary. That's why the
  lexicon editor exists.
- "Slack export" means the copy/paste you get from a Slack channel and simple
  JSON arrays — not the full zip export format.
- Person names are matched as literal strings. "Sam" and "Sam Rivera" are
  different people as far as the engine knows; the demo data is consistent on
  purpose.
- Everything runs on-device, so a decade of chat history will be slow on a
  decade-old phone. It streams progress, at least.
- The timeline's pendulum animation is honestly just cosmetic. It looks great
  though.

## Privacy

There is nothing to phone home with. No analytics, no fonts required at
runtime (Google Fonts are loaded but system fallbacks cover offline use), no
network requests anywhere in the code. All data stays in your browser's
storage until you delete it or export it yourself.

## Licence

Do whatever you like with it. If you catch a teammate's framework boomerang
with it, that's on you.
