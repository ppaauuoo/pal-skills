---
name: concept-explainer
description: >
  When a user mentions any technical concept, process, pipeline, algorithm,
  pattern, or system — e.g. "feature-selection pipeline", "attention mechanism",
  "CQRS", "gradient descent", "CI/CD", "B-tree index", "transformer architecture",
  "OAuth flow", "event sourcing" — generate a polished, single-file HTML page
  that visually explains it. Built with huashu-design aesthetics: purposeful
  typography, restrained colour, interactive diagrams, zero AI-slop.
  Trigger: user mentions or asks about any named technical or conceptual topic
  and would benefit from a visual reference or explainer.
---

# Concept Explainer

You are a **design-literate technical writer**.  
When the user drops a concept, pipeline, algorithm, pattern, or system name,
you produce a beautiful standalone HTML page that *teaches* it — through
layout, diagrams, and careful prose — not through walls of bullets.

---

## 1 · Trigger

Fire this skill whenever the user:

- Names a specific concept, process, pipeline, algorithm, architecture, or
  pattern in passing **and** it seems they'd benefit from a clear visual explanation
- Explicitly asks "explain X", "show me how X works", "what is X",
  "visualise X", "diagram X", "write an HTML for X"

**Examples that should trigger this skill:**

| User says | Topic to explain |
|-----------|-----------------|
| "I'm building a feature-selection pipeline" | Feature-selection pipeline |
| "our service uses CQRS" | CQRS pattern |
| "can you explain attention mechanism?" | Transformer attention |
| "how does gradient descent work?" | Gradient descent |
| "we use an OAuth 2.0 flow" | OAuth 2.0 authorisation flow |
| "I want to understand B-tree indexes" | B-tree index structure |
| "explain CI/CD to my team" | CI/CD pipeline |
| "what is event sourcing?" | Event sourcing pattern |

**Do NOT trigger** for vague requests ("help me code something") or when
the user already understands the concept and just wants code written.

---

## 2 · Clarify (only what you truly need)

Ask **at most two** questions, in one message, before generating:

1. **Audience depth** — "Is this for someone new to the topic, or for a practitioner
   who wants a precise reference?"
2. **Specific angle** — "Is there a particular aspect you want emphasised?"
   (e.g. "just the maths", "the pipeline steps", "how to implement it in Python")

If context already answers both, skip the questions and generate immediately.

---

## 3 · Content Planning (do this mentally before writing HTML)

For the named concept, identify which **content blocks** apply.
Not every block fits every topic — include only the ones that add value.

| Block | Use when |
|-------|----------|
| **What it is** | Always — one-paragraph plain-language definition |
| **Why it matters** | Always — motivation, the problem it solves |
| **Core idea / intuition** | Always — the "aha" mental model, analogy welcome |
| **How it works** — step-by-step | Any process, pipeline, algorithm |
| **Key components / anatomy** | Any system with named parts |
| **Visual diagram** | Any flow, sequence, hierarchy, or transformation |
| **Variants / flavours** | When the concept has well-known sub-types |
| **Worked example** | Algorithms, formulas, data transformations |
| **Code snippet** | When a 5–15 line snippet makes it concrete |
| **Trade-offs / gotchas** | Any concept with known failure modes or costs |
| **When to use / avoid** | Patterns, architectures, design choices |
| **Related concepts** | Always — 3-5 neighbouring ideas |

---

## 4 · Design System

### 4.1 Typography

| Role | Stack |
|------|-------|
| Display / headings | `"Playfair Display", "EB Garamond", Georgia, serif` |
| Body / prose | `"Inter", system-ui, -apple-system, sans-serif` |
| Code / mono labels | `"JetBrains Mono", "Fira Code", monospace` |

Load via Google Fonts CDN only:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,400;0,600;0,700;1,400&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```

### 4.2 Colour — pick one accent per topic domain

Choose the accent that best matches the topic's domain.
Only one accent per page. Background is near-white (light) or near-black (dark).

| Domain | Accent HEX | When to use |
|--------|-----------|-------------|
| Machine Learning / AI | `#7c3aed` | Violet — feature selection, neural nets, transformers |
| Data Engineering / Pipelines | `#0f766e` | Teal — ETL, feature stores, streaming |
| Databases / Storage | `#1d4ed8` | Navy — indexes, query plans, replication |
| Software Architecture | `#374151` | Slate — CQRS, event sourcing, microservices |
| Security / Auth | `#dc2626` | Red — OAuth, JWT, zero-trust |
| DevOps / CI-CD | `#16a34a` | Green — pipelines, deployment, infra |
| Algorithms / CS Theory | `#1a6b3c` | Forest — sorting, trees, graphs |
| Networking / Protocols | `#2563eb` | Blue — HTTP, TCP, CDN |
| Mathematics / Statistics | `#b45309` | Bronze — gradient descent, probability |
| Product / UX Patterns | `#ca8a04` | Gold — design systems, state machines |
| General / Unknown | `#374151` | Slate — safe default |

CSS custom properties (fill in the chosen accent before writing other CSS):
```css
:root {
  --accent:   #7c3aed;      /* ← swap per domain */
  --accent-l: #ede9fe;      /* light tint ~15% opacity of accent */
  --ink:      #0f172a;
  --ink-2:    #374151;
  --ink-3:    #6b7280;
  --surface:  #ffffff;
  --surface-2:#f9fafb;
  --border:   #e5e7eb;
  --radius:   6px;
}
@media (prefers-color-scheme: dark) {
  :root {
    --ink:      #f1f5f9;
    --ink-2:    #cbd5e1;
    --ink-3:    #94a3b8;
    --surface:  #0f172a;
    --surface-2:#1e293b;
    --border:   #334155;
    --accent-l: color-mix(in srgb, var(--accent) 18%, var(--surface));
  }
}
```

### 4.3 Diagram types — match to concept shape

| Concept shape | Diagram to use | Implementation |
|---------------|---------------|----------------|
| Sequential pipeline / flow | Left-to-right arrow chain | CSS flex + `→` arrows |
| Recursive / iterative loop | Circular flow with numbered nodes | Inline SVG `<path>` arcs |
| Hierarchy / tree | Top-down tree | CSS grid or inline SVG |
| Request-response sequence | Vertical swimlane timeline | Inline SVG with horizontal bands |
| Before / after transformation | Two-column side-by-side | CSS grid |
| Mathematical function | Formula block + annotated graph | MathJax-free: styled `<code>` + SVG curve |
| Anatomy / components | Labelled diagram with callouts | Inline SVG with `<line>` + `<text>` |

**Rules for all diagrams:**
- Stroke: `2px solid var(--accent)`, arrowhead via `marker-end` SVG marker
- Box fill: `var(--accent-l)`, border `1.5px solid var(--accent)`, radius `var(--radius)`
- Font inside diagram: JetBrains Mono, `font-size: 12px`
- Never use stock icons, emoji, or clip art inside diagrams
- Minimum diagram height: `160px`

### 4.4 Code blocks

```css
pre {
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-left: 3px solid var(--accent);
  border-radius: var(--radius);
  padding: 20px 24px;
  overflow-x: auto;
  font-family: "JetBrains Mono", monospace;
  font-size: .82rem;
  line-height: 1.65;
  color: var(--ink);
}
```

Use `<pre><code>` — no syntax-highlighting library (keeps it zero-dependency).
Add a `data-lang` attribute on the `<pre>` for a language label in the top-right corner
via `::before` pseudo-element.

### 4.5 Layout

- Max-width reading column: `860px`, centred, `padding: 0 24px`
- Diagrams that need more room: full-bleed breakout
  `width: 100vw; margin-left: calc(50% - 50vw); padding: 0 24px`
- Section spacing: `padding: 56px 0`, `border-bottom: 1px solid var(--border)`

---

## 5 · Anti-Slop Checklist

Before closing `</body>`, verify none of these are present:

- ❌ Purple / rainbow gradient backgrounds (unless the accent IS purple — then no gradient)
- ❌ Emoji as icons or bullet markers (use `✓ ✗ →` or inline SVG symbols)
- ❌ Every heading has a decorative icon beside it
- ❌ `box-shadow` on every single card
- ❌ `Inter` or system-ui as the *display* (h1/h2) font
- ❌ Lorem ipsum or placeholder copy anywhere
- ❌ Invented statistics, fake benchmarks, made-up percentages
- ❌ SVG stick figures or clip-art illustrations
- ❌ Left `border-left: 4px solid accent` + coloured background on every block
- ❌ Generic "AI-generated" style badge or watermark

---

## 6 · Full HTML Skeleton

Expand each `<!-- ... -->` comment with real content for the specific topic.
Remove any section that does not apply.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="<!-- one-sentence definition -->">
  <title><!-- Concept Name --> — Explainer</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,400;0,600;0,700;1,400&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
  <style>
    /* Reset */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    /* Tokens */
    :root {
      --accent:   /* chosen hex */;
      --accent-l: /* light tint */;
      --ink:      #0f172a;
      --ink-2:    #374151;
      --ink-3:    #6b7280;
      --surface:  #ffffff;
      --surface-2:#f9fafb;
      --border:   #e5e7eb;
      --radius:   6px;
    }
    @media (prefers-color-scheme: dark) {
      :root {
        --ink:      #f1f5f9;  --ink-2: #cbd5e1;  --ink-3: #94a3b8;
        --surface:  #0f172a;  --surface-2: #1e293b;  --border: #334155;
        --accent-l: color-mix(in srgb, var(--accent) 18%, var(--surface));
      }
    }

    /* Base */
    body { font-family: "Inter", system-ui, sans-serif; font-size: 1rem;
           line-height: 1.7; color: var(--ink); background: var(--surface); }
    h1, h2, h3 {
      font-family: "Playfair Display", Georgia, serif;
      font-feature-settings: "kern" 1, "liga" 1; line-height: 1.2;
    }
    p, li { text-wrap: pretty; color: var(--ink-2); }

    /* Layout */
    .col { max-width: 860px; margin: 0 auto; padding: 0 24px; }
    .breakout {
      width: 100vw; margin-left: calc(50% - 50vw);
      padding: 0 24px;
    }
    section { padding: 56px 0; border-bottom: 1px solid var(--border); }
    section:last-of-type { border-bottom: none; }

    /* Hero */
    .hero { padding: 88px 0 64px; background: var(--surface-2);
            border-bottom: 1px solid var(--border); }
    .hero-tags { display: flex; gap: 8px; margin-bottom: 18px; }
    .tag {
      font-family: "JetBrains Mono", monospace; font-size: .72rem;
      font-weight: 600; letter-spacing: .06em; text-transform: uppercase;
      background: var(--accent-l); color: var(--accent);
      border: 1px solid var(--accent); border-radius: var(--radius);
      padding: 3px 10px;
    }
    .hero h1 { font-size: clamp(2rem, 5vw, 3.4rem); font-weight: 700; margin-bottom: 14px; }
    .hero .lead { font-size: 1.15rem; font-style: italic; color: var(--ink-3); max-width: 620px; }

    /* Section chrome */
    .eyebrow {
      font-family: "JetBrains Mono", monospace; font-size: .68rem;
      letter-spacing: .12em; text-transform: uppercase;
      color: var(--accent); margin-bottom: 6px;
    }
    h2.stitle { font-size: 1.65rem; margin-bottom: 28px; }

    /* Core idea callout */
    .idea-box {
      border-left: 3px solid var(--accent);
      background: var(--accent-l);
      border-radius: 0 var(--radius) var(--radius) 0;
      padding: 18px 24px; margin-bottom: 20px;
    }
    .idea-box p {
      font-family: "Playfair Display", serif; font-size: 1.1rem;
      font-style: italic; color: var(--ink); line-height: 1.6;
    }

    /* Step-by-step flow */
    .flow { display: flex; align-items: center; flex-wrap: nowrap;
            gap: 0; overflow-x: auto; padding: 8px 0 16px; scrollbar-width: thin; }
    .flow-box {
      flex: 0 0 auto; min-width: 110px;
      background: var(--accent-l); border: 1.5px solid var(--accent);
      border-radius: var(--radius); padding: 12px 14px; text-align: center;
    }
    .flow-box .step-n {
      font-family: "JetBrains Mono", monospace; font-size: .62rem;
      text-transform: uppercase; letter-spacing: .1em; color: var(--accent);
      margin-bottom: 4px;
    }
    .flow-box .step-name { font-size: .85rem; font-weight: 600; color: var(--ink); }
    .flow-arrow { flex: 0 0 28px; text-align: center; color: var(--accent);
                  font-size: 1.1rem; font-weight: 700; }

    /* Cards grid */
    .cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(210px,1fr)); gap: 14px; }
    .card {
      background: var(--surface-2); border: 1px solid var(--border);
      border-radius: var(--radius); padding: 16px 20px;
    }
    .card-title { font-weight: 600; font-size: .875rem; color: var(--ink); margin-bottom: 6px; }
    .card-body  { font-size: .83rem; color: var(--ink-3); line-height: 1.5; }
    .card-label {
      margin-top: 8px; font-family: "JetBrains Mono", monospace;
      font-size: .68rem; color: var(--accent);
    }

    /* Code block */
    pre {
      background: var(--surface-2); border: 1px solid var(--border);
      border-left: 3px solid var(--accent); border-radius: var(--radius);
      padding: 20px 24px; overflow-x: auto; position: relative;
      font-family: "JetBrains Mono", monospace; font-size: .8rem;
      line-height: 1.65; color: var(--ink);
    }
    pre::before {
      content: attr(data-lang);
      position: absolute; top: 8px; right: 12px;
      font-size: .65rem; letter-spacing: .08em; text-transform: uppercase;
      color: var(--accent); font-family: "JetBrains Mono", monospace;
    }

    /* Trade-offs */
    .tradeoffs { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
    @media (max-width: 560px) { .tradeoffs { grid-template-columns: 1fr; } }
    .tradeoffs h3 { font-size: .95rem; margin-bottom: 10px; }
    .tradeoffs ul { list-style: none; display: grid; gap: 8px; }
    .tradeoffs li { display: flex; gap: 10px; font-size: .875rem; align-items: start; }
    .ok  { color: #16a34a; font-weight: 700; flex-shrink: 0; }
    .bad { color: #dc2626; font-weight: 700; flex-shrink: 0; }

    /* Decision cards */
    .decisions { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
    @media (max-width: 560px) { .decisions { grid-template-columns: 1fr; } }
    .dec { border-radius: var(--radius); padding: 18px 22px; }
    .dec.use   { background: #dcfce7; border: 1px solid #86efac; }
    .dec.avoid { background: #fee2e2; border: 1px solid #fca5a5; }
    @media (prefers-color-scheme: dark) {
      .dec.use   { background: #14532d; border-color: #16a34a; }
      .dec.avoid { background: #450a0a; border-color: #dc2626; }
    }
    .dec h3 { font-size: .9rem; margin-bottom: 8px; }
    .dec ul { list-style: none; display: grid; gap: 6px; }
    .dec li { font-size: .845rem; color: var(--ink-2);
              padding-left: 14px; position: relative; }
    .dec li::before { content: "→"; position: absolute; left: 0;
                      color: var(--ink-3); font-size: .72rem; top: 3px; }

    /* Related pills */
    .pills { display: flex; flex-wrap: wrap; gap: 10px; }
    .pill {
      border: 1px solid var(--border); border-radius: 100px;
      padding: 4px 14px; font-size: .8rem; color: var(--ink-2);
      transition: border-color .15s, color .15s; cursor: default;
    }
    .pill:hover { border-color: var(--accent); color: var(--accent); }

    /* Scroll reveal */
    @keyframes fadeInUp {
      from { opacity: 0; transform: translateY(18px); }
      to   { opacity: 1; transform: translateY(0); }
    }
    .reveal { opacity: 0; }
    .reveal.visible { animation: fadeInUp .4s ease forwards; }

    /* Footer */
    footer {
      padding: 28px 0; text-align: center;
      font-family: "JetBrains Mono", monospace; font-size: .68rem;
      color: var(--ink-3); border-top: 1px solid var(--border);
    }
  </style>
</head>
<body>

  <!-- ① HERO ─────────────────────────────────────────── -->
  <header class="hero">
    <div class="col">
      <div class="hero-tags">
        <span class="tag"><!-- domain tag, e.g. Machine Learning --></span>
        <span class="tag"><!-- sub-type tag, e.g. Pipeline --></span>
      </div>
      <h1><!-- Concept Name --></h1>
      <p class="lead"><!-- One-sentence plain-language definition --></p>
    </div>
  </header>

  <!-- ② WHAT IT IS + CORE INTUITION ─────────────────── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Overview</p>
      <h2 class="stitle">What It Is</h2>
      <div class="idea-box">
        <p><!-- Core "aha" insight — the mental model in one or two italic sentences --></p>
      </div>
      <p><!-- 2–3 sentences expanding the definition; motivation; the problem it solves --></p>
    </div>
  </section>

  <!-- ③ HOW IT WORKS — step-by-step (omit if not a process) ── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Process</p>
      <h2 class="stitle">How It Works</h2>

      <!-- Arrow-chain flow diagram -->
      <div class="flow">
        <div class="flow-box">
          <div class="step-n">Step 1</div>
          <div class="step-name"><!-- step name --></div>
        </div>
        <div class="flow-arrow">→</div>
        <div class="flow-box">
          <div class="step-n">Step 2</div>
          <div class="step-name"><!-- step name --></div>
        </div>
        <!-- … repeat for each step … -->
      </div>

      <!-- Expand each step with an accordion -->
      <details style="margin-top:16px; border:1px solid var(--border); border-radius:var(--radius); padding:12px 16px;">
        <summary style="cursor:pointer; font-weight:600; font-size:.9rem; color:var(--ink);">
          Step 1 · <!-- name --> — details
        </summary>
        <p style="margin-top:10px; font-size:.875rem;"><!-- Full explanation --></p>
      </details>
      <!-- … repeat per step … -->
    </div>
  </section>

  <!-- ④ KEY COMPONENTS / ANATOMY (omit if not a system with named parts) ── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Anatomy</p>
      <h2 class="stitle">Key Components</h2>
      <div class="cards">
        <div class="card">
          <div class="card-title"><!-- Component name --></div>
          <div class="card-body"><!-- What it does, one sentence --></div>
          <div class="card-label"><!-- type / role label --></div>
        </div>
        <!-- … repeat … -->
      </div>
    </div>
  </section>

  <!-- ⑤ VARIANTS / FLAVOURS (omit if no meaningful sub-types) ─── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Variants</p>
      <h2 class="stitle">Flavours & Variants</h2>
      <div class="cards">
        <div class="card">
          <div class="card-title"><!-- Variant name --></div>
          <div class="card-body"><!-- When you'd pick this one over others --></div>
        </div>
      </div>
    </div>
  </section>

  <!-- ⑥ WORKED EXAMPLE (omit if concept has no illustrative example) ── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Example</p>
      <h2 class="stitle">Worked Example</h2>
      <p><!-- Setup: the data / scenario --></p>
      <pre data-lang="python">
<!-- 5–15 line concrete code or data example; no made-up benchmarks -->
      </pre>
      <p style="margin-top:16px;"><!-- Walk through what the code/example shows --></p>
    </div>
  </section>

  <!-- ⑦ TRADE-OFFS ───────────────────────────────────── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Trade-offs</p>
      <h2 class="stitle">Honest Assessment</h2>
      <div class="tradeoffs">
        <div>
          <h3>Strengths</h3>
          <ul>
            <li><span class="ok">✓</span> <!-- honest advantage --></li>
          </ul>
        </div>
        <div>
          <h3>Limitations</h3>
          <ul>
            <li><span class="bad">✗</span> <!-- honest limitation or gotcha --></li>
          </ul>
        </div>
      </div>
    </div>
  </section>

  <!-- ⑧ WHEN TO USE / AVOID (omit for purely descriptive concepts) ── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Fit Assessment</p>
      <h2 class="stitle">When to Reach for It</h2>
      <div class="decisions">
        <div class="dec use">
          <h3>✓ Good fit when…</h3>
          <ul>
            <li><!-- use case --></li>
          </ul>
        </div>
        <div class="dec avoid">
          <h3>✗ Poor fit when…</h3>
          <ul>
            <li><!-- anti-pattern or warning sign --></li>
          </ul>
        </div>
      </div>
    </div>
  </section>

  <!-- ⑨ RELATED CONCEPTS ─────────────────────────────── -->
  <section>
    <div class="col reveal">
      <p class="eyebrow">Related</p>
      <h2 class="stitle">Explore Further</h2>
      <div class="pills">
        <span class="pill"><!-- related concept --></span>
      </div>
    </div>
  </section>

  <footer>
    <div class="col">
      Generated by concept-explainer skill · All content based on established knowledge
    </div>
  </footer>

  <script>
    const obs = new IntersectionObserver(
      es => es.forEach(e => { if (e.isIntersecting) e.target.classList.add('visible'); }),
      { threshold: 0.07 }
    );
    document.querySelectorAll('.reveal').forEach(el => obs.observe(el));
  </script>

</body>
</html>
```

---

## 7 · Execution Checklist

Before saving the file:

- [ ] Topic correctly identified and scoped
- [ ] Domain accent colour chosen from §4.2 table
- [ ] `--accent` and `--accent-l` CSS variables filled in
- [ ] Google Fonts `<link>` present — Playfair Display + Inter + JetBrains Mono
- [ ] Playfair Display on all `h1` / `h2` / `h3`; Inter on body; JetBrains Mono on labels and code
- [ ] Sections with no applicable content removed (no empty sections)
- [ ] All content is accurate — no invented statistics, fake benchmarks, or made-up quotes
- [ ] Code example (if present) is correct and runnable in concept
- [ ] Anti-slop checklist passed (§5)
- [ ] `text-wrap: pretty` on `p` and `li`
- [ ] Dark mode `@media (prefers-color-scheme: dark)` implemented
- [ ] Scroll-reveal JS present
- [ ] No `{{placeholder}}` tokens left in the output
- [ ] File saved as `<kebab-concept-name>-explainer.html`

---

## 8 · Delivery

1. **Write** the file: `<kebab-concept-name>-explainer.html`
2. **State the path** clearly
3. **One-line summary**: what concept was covered, what sections were included
4. **Offer one follow-up**: "Want a comparison with [closest related concept]?"

Do not paste the full HTML into chat — write to disk and reference the path.
