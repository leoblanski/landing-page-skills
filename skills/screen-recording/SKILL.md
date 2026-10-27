---
name: screen-recording
description: Use when a landing page, ad or social post needs video of a real product flow instead of static screenshots, and when a hand-recorded screen capture keeps coming out shaky, off-pace, too heavy, or impossible to re-record identically. Covers driving the app with a headless browser, a virtual camera applied after capture, captions, pacing, and encoding to a byte budget. Also use when a recorded demo has to be regenerated after a UI change without looking different.
---

# Recording a product demo you can regenerate

## Overview

A screenshot shows that a product exists. A short video shows it working, and
on a landing page that difference converts. But a demo recorded by hand rots:
the UI changes, nobody remembers the exact path that was clicked, and the
re-record has different timing, different mouse jitter and a different file
size.

The fix is to treat the demo as a build artifact. A script drives the app, the
camera move is applied after capture, and the same inputs produce the same MP4.
When the UI changes you re-run it, you do not re-perform it.

This skill is the method and the reasoning. There is no runnable package here,
you implement it in your project, because every step touches your routes, your
selectors and your fixtures.

## When it is worth it

Worth it when the demo is a durable asset: a landing page hero, an ad creative,
an onboarding loop, anything that will be regenerated as the product changes.

Not worth it for a one-off. A manual capture with CleanShot or `screencapture`
is faster and good enough for a Slack message or a changelog entry. The payoff
here is reproducibility, so if you only ever record once, you are paying the
setup cost for nothing.

## The pipeline

Seven stages. Each one exists because the obvious approach fails in a specific
way, and the failure is named so you can recognize it.

| Stage | What it does | Why not the obvious way |
|---|---|---|
| 1. Deterministic app | Serve the app with fixed data and a frozen clock | Live data changes the recording between runs |
| 2. Headless drive | Puppeteer walks the real flow | Hand-clicking cannot be repeated |
| 3. Synthetic pointer | Draw your own cursor, move it by elapsed time | The real cursor is not captured, and step-based motion desyncs |
| 4. Capture wide | Record the whole viewport, never zoomed | Zooming the browser loses pixels you later want |
| 5. Virtual camera | Apply zoom and pan afterwards with ffmpeg | A real camera move cannot be re-timed or undone |
| 6. Pace and caption | Stretch time per window, overlay captions | A single speed is either too slow to watch or too fast to read |
| 7. Encode to budget | Raise CRF until the file fits | A fixed CRF either bloats the page or looks bad |

Read [references/pipeline.md](references/pipeline.md) for each stage in detail,
with the code that implements it and the exact bug that motivated it.

Then [references/device-frames.md](references/device-frames.md) for the frame
that goes around the video: real device measurements, the inline SVG shell, and
why the recording viewport is the screen minus the status bar and the browser
bar. Skip it and the demo reads as a cropped website rather than a product on a
phone.

## The scene model

Model a demo as scenes, and a scene as a declaration plus two functions. This
is the shape that made the rest composable:

```js
{
  name: 'quick-scan-phone',
  viewport: PHONE_VIEWPORT,
  output: PHONE_VIDEO,
  routes: { 'POST /api/scans': { response: json(FIXTURE, 201), delayMs: 700 } },
  css: ISOLATE_WORKSPACE_CSS,
  readySelector: '#form input[type="url"]',
  maxBytes: 900 * 1024,
  pace: { fast: 2.4, slow: 1, move: 1.4 },

  before: async ({ goto }) => { await goto('/scan'); },
  record: async ({ page, camera, cursor, poster, shot }) => { /* the beats */ },
}
```

`before` gets the app into the starting state, nothing is recorded yet.
`record` performs the beats. Splitting them matters: setup noise (a login, a
sidebar collapsing) must not land in the video, and `before` is also where the
poster frame comes from.

Keep `record` readable as a shot list. If it reads like a list of beats a
director would call, the video will have a rhythm. If it reads like a test, the
video will feel like a test.

## Rules that came from being burned

- **Move the cursor by elapsed time, not by step count.** Interpolating over
  `performance.now()` keeps the glide honest when a frame is slow. Stepping a
  fixed number of times makes the cursor arrive early or late depending on
  machine load, and it desyncs from the capture.
- **Never measure layout at tween time.** Compute target boxes once, before the
  move. `getBoundingClientRect()` during a move reads a mid-animation value.
- **The camera is data, not an action.** Record the move as
  `{ at, duration, rect }` and compile it to one ffmpeg `zoompan` expression at
  the end. That is what makes re-pacing and crossfades possible after the fact.
- **Align the move log to the real video length.** The capture always lags the
  driving code slightly. Measure the encoded frame count, subtract, shift every
  move earlier by the difference. Skip this and every zoom lands late.
- **Hold on captions, rush everything else.** Mark time windows as slow (a
  caption is up), move (the camera is traveling) or fast (everything else), then
  stretch each window separately in one `setpts` expression. One global speed
  cannot serve both reading and watching.
- **Stub the bot check.** A Turnstile or reCAPTCHA widget will block the flow in
  headless and you get a recording of a spinner.
- **Kill analytics during capture.** Abort requests to GA, GTM, Sentry, Meta.
  They slow the capture and they pollute real metrics with fake sessions.
- **Encode by budget, not by quality number.** Start at a CRF, check the file
  size, step up until it fits, fail loudly if it never does. A hero video that
  silently grows to 4 MB costs you the LCP you were trying to win.
- **Ship MP4 plus WebM plus a WebP poster.** The poster is the first frame the
  visitor sees and it is what keeps CLS at zero.

## Red flags

| Thought | Reality |
|---|---|
| "I'll just screen-record it myself" | Fine once. The second time the UI changed, you cannot match the original. |
| "Zoom the browser so the detail is readable" | Capture wide, zoom in post. Browser zoom throws away pixels you will want. |
| "The cursor is in the capture, right?" | It is not. Draw a synthetic one or the video has no visible intent. |
| "One playback speed is simpler" | It is, and it is unwatchable. Slow on captions, fast between them. |
| "CRF 28 is a good default" | Against a byte budget there is no good default. Step until it fits. |
| "Real data makes it authentic" | Real data makes it irreproducible, and it leaks whoever's record you recorded. |
| "The zoom feels slightly late" | It is. Align the move log to the measured video length. |

## Fixtures are not optional

Record against fixtures, never a real account. Two reasons, and the second is
the one that bites: a real account makes the demo impossible to reproduce once
that data changes, and it puts someone's real name, photo and history into an
asset you are about to publish on a landing page.

Build the fixture to look plausible and be obviously not a real person.
