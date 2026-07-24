# IP Investment Readiness Tool — Demo Technical Stack

This document describes the technology choices behind `IP_Readiness_Demo.html` and its companion test suite `IP_Readiness_Demo_Tests.html`.

## 1. Overall architecture

The demo is a **single self-contained HTML file** — no server, no build step, no package manager, no internet connection required. Opening the file directly in any modern browser (double-click, or `file://` path) runs the entire application. This was a deliberate choice for a sponsor-facing prototype: it can be emailed, dropped on a USB stick, or hosted on any static file host with zero configuration, and it will behave identically everywhere.

There is no backend. All scoring logic, state, and rendering happen client-side, in the browser's JavaScript engine. Nothing is sent over the network and no data persists after the tab is closed (no cookies, no `localStorage`).

## 2. Languages and layers

| Layer | Technology | Notes |
|---|---|---|
| Structure | HTML5 | A single `<div id="app">` root; every screen is generated as an HTML string and swapped in wholesale. |
| Styling | CSS3 (custom properties, Flexbox, CSS Grid) | Dark theme driven entirely by `:root` CSS variables (one accent color per dimension), so re-theming means editing a handful of variables. Responsive via `@media` breakpoints and `grid-template-columns`. No CSS framework (no Bootstrap/Tailwind) and no CSS preprocessor. |
| Behavior | Vanilla JavaScript (ES6+) | Template literals, arrow functions, array methods (`map`/`filter`/`reduce`/`find`), destructuring, spread. No transpilation needed — every feature used is supported natively by current evergreen browsers. |

No external libraries, no CDN `<script src>` tags, no npm dependencies, no React/Vue/jQuery. Everything the page needs is inline in the one file.

## 3. Application pattern: data model + pure engine + dumb renderer

The script is organized into three clearly separated concerns, in this order:

1. **Data model** — plain JavaScript object literals (`D1`, `D2`, `D3`, `D4`, `D5`) describing each dimension's framing line, questions, answer options, point values/weights, evidence requirements, hard-blocker caps, and band thresholds. This is content, not code — it mirrors the Field 1–8 structure from the module template and the D3/D4 weighted-formula spec directly, so updating a question's wording or point value never touches any logic.
2. **Scoring engine** — a set of pure functions (`computeD1D2D5`, `computeD3`, `computeD4`, `overallBand`, `evidenceChecklist`, `improvementList`) that take the current answers and return numbers/bands. These functions read from a single mutable `state` object but do not touch the DOM at all, which is what makes them independently testable (see §5).
3. **Renderer** — a `render()` function that reads `state.screen` and calls the matching `renderX()` function, each of which returns an HTML string. `render()` sets `app.innerHTML` once per state change. There is no virtual DOM and no diffing: every click handler mutates `state` and calls `render()`, which redraws the whole screen from scratch. For an app of this size (a handful of screens, no animation-heavy transitions) this "re-render everything" pattern is simpler and more robust than hand-rolled DOM patching, at the cost of not being suitable for a much larger production app.

## 4. State management and events

- All mutable state lives in one plain object, `state` (current screen, current dimension/question index, all answers, all D6 evidence choices, debug-toggle flag).
- Event handling uses **inline `onclick="functionName(...)"` HTML attributes** generated as part of the template strings, rather than `addEventListener` calls. This keeps each render self-contained (no need to re-attach listeners after replacing `innerHTML`) and keeps the code short, at the cost of putting some logic into HTML attribute strings — an acceptable trade-off at this scale, but not a pattern that scales to a large app.
- Navigation (question skip/jump logic for D1's co-creator branch and D2's revenue-split branch) is resolved by `activeQuestions()`, which walks each dimension's question list following `jump`/`requiresJumpFrom` declarations in the data model — so the branching logic is driven by data, not hard-coded `if` statements per dimension.

## 5. Testing approach

`IP_Readiness_Demo_Tests.html` is a second, independent single-file HTML page. It intentionally **duplicates** the data model and scoring-engine functions from the main demo (rather than loading the demo file via `<script src>`), because browsers block `fetch()`/module loading between two local files opened via `file://` (a CORS restriction with no server). Duplication trades a small sync burden (if the engine changes, the test file's copy needs updating too) for zero-configuration portability — you can open the test file the same way you open the demo, with no local server.

The test file implements a minimal assertion harness (`test()`, `eq()`, `close()`) and runs 30 test cases across six categories: per-dimension additive scoring (D1/D2/D5), branch/skip logic, the D6 evidence-discount math, D3's weighted-formula engine (including its two hard caps and the single-work-dependency rule), D4's weighted-formula engine (including the non-stacking anomaly penalty), the framework_v4.0 overall-aggregation rules (band-index averaging, the ".5 rounds up" rule, and the weakest-link cap — including the Technical Spec's own worked example, reproduced as a test), and the two-tier Critical/Recommended improvement-list classification. Results render as a pass/fail report directly in the browser.

**Verification note:** the sandbox this was built in has no outbound internet access, so headless-browser tools (Puppeteer, Playwright) and even lightweight DOM libraries (jsdom) could not be installed to click through the actual rendered UI end-to-end. To compensate, the scoring engine — the part of the app where a bug would actually produce a wrong investment-readiness answer — was verified two ways: (1) Node's built-in `vm` module to execute the real engine code in isolation and run all 30 assertions (this is what `IP_Readiness_Demo_Tests.html` also does, in-browser, for you), and (2) a syntax check (`node --check`) of the full demo file. The click-through navigation and CSS layout were verified by careful manual code review rather than an automated headless click-test, since that tooling wasn't installable here — worth a quick manual click-through on your end before presenting to sponsors.

## 6. Why this stack (given the goal)

The brief was an MBTI-style, clickable, sponsor-facing demo — not a production system. Given that:

- **Zero dependencies / zero build step** means anyone can open it immediately, on any machine, with no setup risk during a live demo.
- **Single file** means it's trivial to share (one email attachment) and trivial to version (one file to diff).
- **Pure-function scoring engine, separated from rendering,** means the investment-readiness math can be tested and trusted independently of how pretty the UI is — important given the scoring rules come from several source documents with real formulas (weighted composites, hard caps, evidence discounts) that need to compute correctly, not just look right.
- **No backend** was a deliberate scope match to the framework specs, which explicitly place this tool in the "assessment, not certification" category — there's no user data to protect because none leaves the browser.

If this moves from demo to production, the natural next steps would be: introducing a small framework (React/Vue) once the number of screens and interaction states grows, moving the data model into a CMS or JSON files editable by non-developers, and adding a real backend if document upload/verification (D6) needs to persist across sessions.
