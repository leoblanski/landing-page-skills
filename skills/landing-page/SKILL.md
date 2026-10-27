---
name: landing-page
description: Use FIRST, before any other skill, whenever creating, redesigning, polishing or reviewing a landing page, sales page, ad landing page or any public page whose job is to convert or rank, and when preparing a page for paid ads. Also use when tempted to add decoration, background art, animation, an extra section or a visual flourish to such a page, or when a brief says "make it pretty", "add some art", "add animation", "improve the LP".
---

# Landing page that converts

## Overview

A landing page exists to make one person take one action. Every element either
moves that action or is removed. Conversion skills decide; design only polishes
what they already decided. **Violating the letter of a phase gate is violating
its spirit.**

## The workflow

Run the phases in order. Each phase has a gate in `checklist.md`; do not start
the next phase with an open gate. Load each named skill with the Skill tool
when its phase starts, never from memory.

| # | Phase | Skills that decide | Output |
|---|---|---|---|
| 1 | Context | `product-marketing` | `.agents/product-marketing.md` current |
| 2 | Offer and funnel | `offers`, `free-tools`, `paywalls`, `pricing` | one action, where money is asked, what is free |
| 3 | Structure | `cro` (+ its `references/form.md`) | section order, above-the-fold contents |
| 4 | Copy | `copywriting`, `marketing-psychology`, `copy-editing` | every string, sourced claims list |
| 5 | Proof and media | `image`, `video` | real captures in real device mockups |
| 6 | Visual identity | your project's own style/UI-pattern skills, then a general design skill for finishing only | tokens applied, nothing added |
| 7 | SEO and AI search | `seo-audit`, `schema`, `ai-seo`, `site-architecture` | prerendered HTML checked |
| 8 | Measurement | `analytics`, `attribution`, `ads` | events, attribution, consent gates |
| 9 | QA | this skill's `checklist.md` | screenshots 1440/1024/768/430/390/360, CLS, LCP |
| 10 | Review | code review agent + per-skill compliance report | PASS before the owner sees it |

Phase 6 is the one you have to fill in yourself: point it at whatever skill
holds your project's design tokens, components and frontend conventions. The
rule that matters is the order, design last and additive never.

## Rules that override taste

- **Justify every element.** One line: which skill rule it serves and which
  funnel step it moves (attention, paste/act, see value, pay). No line, no
  element. Decoration near the primary action is presumed harmful.
- **Motion only shows the product delivering value.** No attention pulses,
  decorative loops, or examples animating on every load.
- **Every claim is true and sourced** (database query, backend constant).
  Superlatives need data.
- **Paid-traffic page: single action, no exits** above the fold.
- **The owner reviews last, never first.** Present only after phase 10, with
  a rule-by-rule compliance report, not "it looks nice".

## Red flags: stop and go back to the phase

| Thought | Reality |
|---|---|
| "A bit of art will make it feel premium" | Phase 6 never adds elements. Back to phase 3 with a conversion reason or drop it. |
| "The skill says X, but this looks better" | Conversion skill wins over taste. Report the conflict, don't override. |
| "Generic phone frame is fine" | `image` requires real device mockups with real captures. |
| "I'll check SEO after it looks right" | Prerender, title, h1, FAQ answers in HTML are part of done. |
| "The number sounds right" | Query it. Unsourced claims shipped to ads are false advertising. |
| "Tests are green, so it's done" | Green tests once missed a bot-check action that would have blocked every preview in production. Run the real path. |
