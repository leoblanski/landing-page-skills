# The seven stages

Code here is illustrative, lifted from a working implementation. Adapt the
selectors, routes and viewports; keep the shape.

Requirements: Node, Puppeteer, `ffmpeg` and `ffprobe` on PATH. Your app has to
be reachable at some URL you pass in. Nothing here assumes a bundler.

---

## 1. A deterministic app

Three things make a run reproducible.

**Stub the API.** Intercept requests and answer from fixtures. The delay is
part of the demo, a form that resolves instantly reads as fake.

```js
export function stubNetwork(page, routes, { assets = {} } = {}) {
  page.on('request', async (request) => {
    const url = new URL(request.url());

    if (url.hostname === 'challenges.cloudflare.com') {
      request.respond({ status: 200, contentType: 'text/javascript', body: TURNSTILE_STUB });
      return;
    }
    if (TRACKING_HOSTS.test(url.hostname)) { request.abort(); return; }
    if (assets[url.href]) { request.respond({ status: 200, ...assets[url.href] }); return; }
    if (!url.pathname.startsWith('/api/')) { request.continue(); return; }

    const route = routes[`${request.method()} ${url.pathname}`];
    if (!route) {
      console.warn(`demo: no stub for ${request.method()} ${url.pathname}`);
      request.respond({ status: 404, contentType: 'application/json', body: '{}' });
      return;
    }
    await wait(route.delayMs || 0);
    request.respond(route.reply ? route.reply(request, url) : route.response).catch(() => {});
  });
}
```

Warning on an unstubbed route, rather than a silent pass-through, is what tells
you the flow changed under you.

The bot-check stub resolves the challenge immediately:

```js
const TURNSTILE_STUB = `window.turnstile = {
  render: (el, opts) => { setTimeout(() => opts.callback('demo-token'), 50); return 'demo'; },
  reset: () => {}, remove: () => {},
};`;
```

**Freeze the clock** if anything renders a relative date, or "2 days ago"
drifts every run.

```js
export function shiftClock(page, iso) {
  return page.evaluateOnNewDocument((targetMs) => {
    const RealDate = Date;
    const offset = targetMs - RealDate.now();
    class DemoDate extends RealDate {
      constructor(...args) { super(...(args.length ? args : [RealDate.now() + offset])); }
      static now() { return RealDate.now() + offset; }
    }
    window.Date = DemoDate;
  }, Date.parse(iso));
}
```

**Hide the chrome** that is true in the app but noise in a demo: a support
bubble, a cookie banner, the text caret. Per scene, inject CSS that isolates
the one surface being sold.

```js
const HIDE_CHROME_CSS = `
  div.fixed.bottom-5.right-5 { display: none !important; }
  input { caret-color: transparent; }
`;
```

---

## 2. Drive it headless

One page per scene, with the viewport and a device scale factor above 1 so the
capture has pixels to spare when the camera zooms in.

```js
async function openPage(browser, baseUrl, scene) {
  const page = await browser.newPage();
  await page.setViewport({ ...scene.viewport, deviceScaleFactor: captureScale });
  await page.setRequestInterception(true);
  stubNetwork(page, scene.routes, { assets: scene.assets });
  await page.evaluateOnNewDocument(() => {
    localStorage.setItem('token', 'demo');
    localStorage.setItem('cookieConsent', JSON.stringify({ type: 'essential', timestamp: Date.now() }));
  });

  const settle = async () => {
    await page.waitForSelector(scene.readySelector, { timeout: 60000 });
    await page.waitForNetworkIdle();
    await page.addStyleTag({ content: `${HIDE_CHROME_CSS}${scene.css}` });
    await page.evaluate(() => document.fonts.ready);
    await wait(600);
  };

  return {
    page,
    goto: async (p) => { await page.goto(`${baseUrl}${p}`, { waitUntil: 'networkidle0' }); await settle(); },
  };
}
```

`document.fonts.ready` before recording is load-bearing. Start without it and
the first frames show fallback metrics, then text reflows on camera.

---

## 3. A synthetic pointer

The OS cursor is not in the capture. Draw one into the page, mirror the real
mouse onto it, and animate **by elapsed time**:

```js
export function createCursor(page, start = { x: 0, y: 0 }, glideMs = 260) {
  const position = { current: start };

  const moveTo = async (target, durationMs = glideMs) => {
    const from = position.current;
    const startedAt = performance.now();
    const step = async () => {
      const progress = durationMs === 0 ? 1 : Math.min((performance.now() - startedAt) / durationMs, 1);
      const t = easeInOut(progress);
      await page.mouse.move(from.x + (target.x - from.x) * t, from.y + (target.y - from.y) * t);
      if (progress === 1) return;
      await wait(16);
      await step();
    };
    await step();
    position.current = target;
  };
  ...
}
```

The progress comes from the clock, not from a loop counter. A slow frame then
costs smoothness, not synchronisation. Driving it as `for (let i = 0; i < 20; i++)`
makes the glide take whatever wall time 20 awaits happen to take, which is not
the duration you asked for.

Before clicking, scroll the target into view and wait for the scroll. Clicking
an element that is technically present but off-screen produces a cursor that
jumps to an edge.

Two pointer kinds, because mobile is not a small desktop: a mouse glides and
shows an arrow, a touch pointer teleports and shows a ripple. Hide the pointer
entirely for the poster frame via a `demo-still` class on the root.

---

## 4. Capture wide

Record the full viewport, always. `page.screencast()` writes WebM while the
script drives.

One recorder, many segments. Stopping and restarting is how you get a cut:

```js
const shot = async (transition, between = async () => {}) => {
  await recorder.stop();
  await between();
  await recorder.start(transition);
};
```

Anything inside `between` is invisible: navigate, swap fixtures, collapse a
sidebar. That is also where the poster screenshot is taken.

The recorder keeps a log rather than doing anything to the picture:

```js
move: (rect, duration) => {
  const at = now();
  updateCurrent((current) => ({
    moves: [...current.moves, { at, duration, rect }],
    windows: [...current.windows, { from: at, to: at + duration, kind: PACE_KIND.move }],
  }));
  take.rect = rect;
},
hold: () => { take.holdingSince = now(); },
```

---

## 5. The virtual camera

A camera move is compiled into one ffmpeg expression over the wide capture.
Because it is data, it can be re-timed, crossfaded or dropped later.

Framing is computed from the union of the elements you want on screen, plus a
margin, plus room for a caption band, clamped to the viewport:

```js
export function frameAround(box, viewport, { width, captionRatio }) {
  const frameWidth = fittingWidth(box, viewport, width, captionRatio);
  const height = frameWidth / aspectOf(viewport);
  const caption = captionFor(frameWidth, captionRatio);
  const rect = {
    x: clamp(box.x + box.width / 2 - frameWidth / 2, 0, viewport.width - frameWidth),
    y: clamp(box.y + box.height / 2 - (height - caption.band) / 2, 0, viewport.height - height),
    width: frameWidth,
    height,
  };
  return { rect, caption: { ...caption, side: sideFor(box, rect, caption.band) } };
}
```

The caption picks its own side: below the subject when there is room, above it
otherwise. A caption that covers the thing it describes is the most common
self-inflicted wound here.

Each rect component becomes a nested `if` over time, eased with a smootherstep,
fed to `zoompan`:

```js
export function cameraFilter(path, { viewport, scale, fps }) {
  if (isStill(path, viewport)) return '';
  const zoom = `${viewport.width}/(${scalarExpression(path, 'width')})`;
  const x = `${scale}*(${scalarExpression(path, 'x')})`;
  const y = `${scale}*(${scalarExpression(path, 'y')})`;
  const size = `${viewport.width * scale}x${viewport.height * scale}`;
  return `setpts=PTS-STARTPTS,zoompan=z='${zoom}':x='${x}':y='${y}':d=1:fps=${fps}:s=${size},`;
}
```

Return empty when nothing moved and the frame is full, so a static scene skips
the filter instead of paying for a no-op resample.

**Alignment.** Capture lags the driving code. Measure the encoded length,
subtract, shift every logged move earlier:

```js
export function alignToVideo(segment, videoSeconds) {
  const lag = Math.max(segment.length - videoSeconds, 0);
  const shift = (time) => Math.max(time - lag, 0);
  return {
    ...segment,
    moves: segment.moves.map((move) => ({ ...move, at: shift(move.at) })),
    windows: segment.windows.map((w) => ({ ...w, from: shift(w.from), to: shift(w.to) }))
      .filter((w) => w.to > w.from),
  };
}
```

Without this every zoom lands slightly after the thing it was meant to reveal.

---

## 6. Pace and captions

Tag each window by kind, then stretch each kind separately in one `setpts`:

```js
export function pacedRetime(windows, pace) {
  const extra = windows.map((w) => `+${round(slowdown(pace, w.kind))}*clip(T-${round(w.from)},0,${round(w.to - w.from)})`);
  return `setpts='(T/${pace.fast}${extra.join('')})/TB',`;
}
```

With `{ fast: 2.4, slow: 1, move: 1.4 }`: dead time runs at 2.4x, a camera move
at 1.4x, and a caption window at real speed so it can be read. Typing, waiting
and scrolling compress; the moments that carry meaning do not.

Captions are static. They fade in, hold, fade out, and their window matches the
beat exactly. Animated captions read as made by a brand. Size them from the
frame width, not in absolute pixels, or they break when the camera zooms.

Roughly 3 seconds per caption. Long enough to read once, short enough that the
video does not stall.

---

## 7. Encode to a byte budget

Step the CRF until the file fits, and fail loudly rather than shipping
something oversized:

```js
function encodeWithin({ file, startCrf, maxBytes, encodeAt }) {
  const fits = (crf) => {
    encodeAt(crf);
    return !maxBytes || statSync(file).size < maxBytes;
  };
  const crf = Array.from({ length: 9 }, (_, step) => startCrf + step).find(fits);
  if (crf === undefined) throw new Error(`${file} is still over ${maxBytes} bytes at crf ${startCrf + 8}`);
  console.log(`${path.basename(file)} crf ${crf}`);
}
```

Three outputs per scene: H.264 MP4, VP9 WebM, and a WebP poster from the
screenshot taken during `before`.

Useful starting points: CRF 32 for MP4, 55 for WebM (VP9 numbers do not mean
what H.264 numbers mean), 24 fps, `-preset veryslow`, `-movflags +faststart`,
`-an` since a landing-page loop is silent.

Two optional timeline shapes worth having:

- **Open on the poster.** Hold the poster image, crossfade into the clip. The
  video then starts on the same frame the visitor already saw, so there is no
  flash at play.
- **Seamless loop.** Crossfade the last half second back into frame one. A hero
  loop that hard-cuts reads as broken.

In the page: `<video>` with the WebP as `poster`, `muted`, `playsinline`,
`preload="metadata"`, WebM source before MP4. Show the poster and skip playback
under `prefers-reduced-motion`.

---

## Running it

A scene list, an output directory, a name filter so you can iterate on one
scene without re-encoding the rest:

```bash
node scripts/recordDemo.js                  # everything
node scripts/recordDemo.js quick-scan-phone # one scene
```

Expect minutes, not seconds, at `veryslow` with a CRF search. It is a build
step, not something to run in a watch loop.

Keep the output in version control. The point is that a UI change gives you a
reviewable diff in the generated video, not a mystery about whether anyone
remembered to re-record.
