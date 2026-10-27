# Landing page skills (Claude Code)

A 10-phase workflow for building a landing page that converts, plus the
marketing skills it calls at each phase.

The idea: conversion decides, design only finishes. Design never adds elements.

## Install

Copy the folders into your project:

```bash
cp -R skills/* /path/to/your-project/.claude/skills/
```

Then, in that project, run `/landing-page`. It loads the other skills phase by
phase.

Works the same in `~/.claude/skills/` if you want it available everywhere.

## What is in here

`skills/landing-page` is the orchestrator: the phase order, the rules that
override taste, the red-flag table, and `checklist.md` with a gate per phase.

The ones it calls:

| Phase | Skills |
|---|---|
| 1 Context | `product-marketing` |
| 2 Offer and funnel | `offers`, `free-tools`, `paywalls`, `pricing` |
| 3 Structure | `cro` |
| 4 Copy | `copywriting`, `marketing-psychology`, `copy-editing` |
| 5 Proof and media | `image`, `video` |
| 6 Visual identity | *(yours, see below)* |
| 7 SEO and AI search | `seo-audit`, `schema`, `ai-seo`, `site-architecture` |
| 8 Measurement | `analytics`, `attribution`, `ads` |

Plus `ad-creative` and `social`, outside the landing-page flow but the same
pack: ad creative, short-form video formats, static ad templates.

## Phase 6 is the gap you fill

Phase 6 originally pointed at project-specific skills holding design tokens,
components and frontend conventions. Those were stripped, they were useless
outside the project they came from.

Point phase 6 at your own equivalent, or at a general design skill. The rule
that survives either way is the order: design runs last, and it never adds an
element that a conversion phase did not already ask for.

## Where the mockup and safe-area rules live

Three different things, easy to mix up:

**Vertical video safe zone** — `ad-creative/references/short-form-video-specs.md`.
A 720x1200 centered band (y=220 to y=1420) that the TikTok/Reels/Shorts UI does
not cover. Captions and load-bearing visuals go inside it. Cross-platform worst
case, re-verify against current app UI.

**Device mockups with real captures** — the `image` skill, enforced by phase 5
of `landing-page`. Real screenshots only, never AI-generated UI, real device
proportions, a callout per mockup pointing at the feature it sells.

**Reading the actual pixels behind text** — not in this repo. That is
`contrast-report.mjs` from HyperFrames (below): it hides each text element,
re-screenshots to see what was really behind the glyphs, scores WCAG contrast
and emits an overlay PNG marking every failure. Broadcast action-safe (90%) and
title-safe (80%) boxes live there too.

## Related, not included

[HyperFrames](https://github.com/heygen-com/hyperframes) (HeyGen, Apache-2.0)
for programmatic video: animation, keyframes, audio ducking, safe-zone overlays,
and the contrast audit described above. Install from upstream, those skills are
coupled to its runtime and do nothing on their own.

## Credits and license

The marketing skills come from
[coreyhaines31/marketingskills](https://github.com/coreyhaines31/marketingskills)
by Corey Haines, MIT. `LICENSE-marketingskills` is that project's license file,
unmodified. Check upstream for newer versions.

`skills/landing-page` is mine, same MIT terms, do what you want with it.
