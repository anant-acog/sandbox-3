---
name: aganitha-eda-live-viz
description: Guides an exploratory data analysis (EDA) pass over a tabular dataset (CSV, TSV, Parquet, Excel — any domain, no chemistry or other domain assumptions) that ends in a live local web page the user can browse and interact with, not a static report file. Use this whenever someone hands over a dataset and asks to explore it, profile it, "see what's in this data," get an overview, or wants EDA — and especially when they want to look at it themselves in a browser rather than just read a text summary. Also trigger on requests like "visualize this dataset," "spin up a dashboard for this CSV," "what does this data look like," or follow-up requests to drill into a specific column or filter of a dataset already being explored this way. Covers profiling (dtypes, missingness, cardinality, summary stats), choosing appropriate plots per column type, generating the plotting and serving code fresh at runtime (never from a bundled script), starting a localhost server, and iterating live as the user asks follow-up questions.
---

# EDA with a live localhost view

The deliverable is a **running local web page**, not a PDF, a markdown writeup,
or a single static HTML file the user opens once and throws away. The user
should be able to leave it open in a browser tab, come back after asking a
follow-up question, and see it change. That distinction drives every choice
below — pick tools and structures that stay alive and cheap to update, not
ones that are fast to render once.

This skill is deliberately code-free: it teaches the process, not a script.
Every dataset has a different shape, so write the profiling and plotting code
fresh each time, tailored to the columns actually present, instead of running
a fixed template that was built for a different dataset.

## Workflow

### 1. Load and profile before deciding anything

Don't jump to plots. First get a factual read on the data — this is what
tells you which plots make sense in step 2:

- Shape (rows, columns), and whether the full file is small enough to load
  entirely into memory or needs sampling (see Practical notes).
- Per-column dtype, and *inferred* semantic type where it disagrees with the
  literal dtype — a column called `zip_code` stored as an int64 is
  categorical, not numeric; a string column that parses cleanly as dates is
  a datetime.
- Missingness per column (count and %).
- Cardinality per column — distinguish low-cardinality categorical (a
  handful of distinct values, good for bar charts), high-cardinality
  categorical (hundreds/thousands of distinct values — an ID column, not a
  chart candidate), and continuous numeric.
- Basic numeric summary stats (min/max/mean/median/std/quartiles) for
  numeric columns.
- A handful of sample rows, printed or logged, so you're looking at real
  values, not just dtype labels.

Do this profiling step in whatever language you're about to build the page
in, so the output is already in a form the plotting code can reuse (a
DataFrame, an array of objects) instead of parsed twice.

### 2. Decide what to plot from the profile, not from habit

Let the actual columns and their relationships drive the choice. A rough
decision guide:

| Column shape | Reasonable plots |
|---|---|
| Single numeric | Histogram (with a sensible bin count for the data's range), box/violin for spotting outliers |
| Single low-cardinality categorical | Bar chart of counts, or a table if there are too many categories for a legible chart |
| Single high-cardinality categorical / ID-like | Skip charting it directly — report cardinality and a few example values instead |
| Two numeric | Scatter plot; correlation heatmap once there are 3+ numeric columns worth comparing pairwise |
| Numeric × categorical | Grouped box/violin (distribution of the numeric column per category) |
| Datetime present | Time series / trend line of relevant numeric columns over time, resampled if the range is long |
| Any dataset with missing data | A missingness matrix or per-column missing-% bar chart, so gaps are visible before anyone trusts a summary stat |
| Many numeric columns | Scatter matrix / pairplot, but only for a handful of columns at a time — it gets unreadable past ~6 |

Not every dataset needs every plot in this table — pick what the profile
actually supports, and skip anything degenerate (a histogram of a column
that's 95% one value isn't worth a full chart; say so in text instead).

### 3. Pick the lightest stack that's actually available

Before writing anything, check what's already in the project: is there a
`pyproject.toml`/`requirements.txt` with pandas/plotly already a dependency?
A `package.json`? An existing venv or node_modules? Match the project's
existing language and libraries rather than introducing a new stack for one
task — that keeps the generated code something the user can actually run
again later.

If there's nothing to match (a bare dataset, no project), default to
whichever of these you're confident is available in the environment, roughly
in this order of preference:

- **Python + pandas + Plotly, served with a tiny stdlib `http.server` or
  Flask app.** Plotly figures export to interactive HTML/JS directly
  (`fig.to_html(full_html=False)`), so you can compose several into one page
  without extra frontend work, and Plotly's zoom/hover/pan makes the
  "browse it yourself" part of the deliverable actually worth having over a
  static image.
- **Node/Bun + a minimal HTTP server, charting via Plotly.js or Chart.js
  loaded from a CDN in the served HTML**, if the project is JS/TS-first per
  its existing tooling. Follow this repo/org's standard stack conventions
  (config, logging, etc.) if this is a real project rather than a scratch
  task.
- **Streamlit**, if it's already installed or the user clearly wants an app
  they can keep extending interactively — it gets you widgets (column
  selectors, filters) with very little code, at the cost of being its own
  process model.

Whatever you pick, the point is interactivity in the browser, not a
particular library — don't fight the environment to force a preference from
this list.

### 4. Generate the code, don't template it

Write an actual script/app tailored to this dataset's real columns and the
plot choices from step 2 — literal column names, sensible titles, axis
labels that use the data's actual units where known. A page full of generic
"Column 1 vs Column 2" placeholders defeats the purpose; the user should
recognize their own data on the page.

Structure it so follow-ups (step 6) are cheap:
- One route/section per plot or logical group of plots, not one monolithic
  function that regenerates everything from scratch.
- Keep the loaded dataset (or a cached/sampled version of it) in memory
  across requests rather than re-reading the file on every page load.

### 5. Start the server and hand the user a URL

- Bind to `127.0.0.1` (localhost), not `0.0.0.0` — this is for the user
  sitting at this machine, not a public listener.
- Pick a port unlikely to collide (check what's already listening if you're
  unsure), and say which port you chose.
- Run the server as a background/detached process so it keeps running after
  you report back — the conversation shouldn't block on it, and the user
  needs it alive to actually look at it.
- If a browser preview tool is available in this environment, open the URL
  yourself and check the page actually rendered (no stack trace, charts
  visible) before telling the user it's ready — catch broken imports or bad
  column references before they do. If no browser tool is available, give
  the user the exact URL to open and ask them to confirm it loaded.

### 6. Iterate without starting over

Treat the server as a running session, not a one-shot artifact. When the
user asks a follow-up — "what about column X," "filter to rows where Y,"
"break that down by region" — regenerate just the affected route/section and
let the server pick it up (reload the page or the specific route), rather
than tearing down and rebuilding the whole app. If the server framework
supports hot reload, rely on it; if not, restart quickly and say so.

## Practical notes

- **Large files:** if a full load is going to be slow or memory-heavy,
  profile on a random sample (and say so on the page — a histogram silently
  built from 1% of the data is misleading) or use a library that supports
  lazy/chunked reads.
- **Process cleanup:** note the process/port you started so it can be killed
  later instead of leaking background servers across a long session.
- **Don't ship a static-report fallback.** If a static HTML file feels
  easier than wiring up a server, that's a sign to simplify the server, not
  to drop the live-page requirement — the ability to leave it open and get
  it to update is the actual ask.
