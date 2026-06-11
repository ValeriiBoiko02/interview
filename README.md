# Senior Test Automation Engineer — Interview Workbook

A focused, Markdown-based study workbook for preparing for **Senior Test Automation Engineer** interviews. Primary stack: **Playwright + TypeScript**.

The content is organized **by theme**. Each theme is a folder; each concept within a theme is a single Markdown file structured in whatever shape fits it best — a concept explainer, a step-by-step guide, or a quick-reference cheat-sheet — so it reads naturally whether you're learning it deeply or skimming it minutes before the interview.

## How it's organized

```
theme-name/
├── README.md          # overview + ordered topic list for the theme
└── NN-topic-name.md   # one concept, fit-for-purpose structure
```

Filenames carry a **two-digit reading-order prefix** (`01-`, `02-`…) so the file list reads in order; each theme's `README.md` lists the same order.

Structure is **fit-for-purpose**, not fixed, but every file meets the same quality bar:

- a short, standalone **recap** near the top for last-minute skimming;
- senior-level depth — the **why** and the trade-offs, not just the what;
- **interview Q&A** and **gotchas** wherever the topic could come up in an interview;
- realistic, idiomatic Playwright + TypeScript code;
- a few canonical reference links.

## How to use it

- **Learning a topic:** read top to bottom.
- **Day-of review:** read just the **recap / TL;DR** at the top of each topic in the themes you're shaky on.
- **Self-testing:** cover the answers in the **Interview Q&A** and respond out loud first.

## Themes index

| Theme | Description | Topics |
|-------|-------------|--------|
| [Playwright](./playwright/) | The primary stack, end to end: setup & config, the runner, locators, auto-waiting, web-first assertions, fixtures, page objects & architecture, network mocking, auth/storage state, API testing, parallelism/sharding, debugging & trace viewer, reporters & CI, flakiness. | 14 |
| [Git](./git/) | Version control: publishing to GitHub, fundamentals, branching/merging, undoing changes, remotes & collaboration, workflows & branch protection. | 6 |
