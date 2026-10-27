# The device frame around the video

A recording of a phone-sized viewport still reads as a cropped website until it
sits in a phone. The frame is what tells the visitor "this is the product, on a
phone, right now".

Draw it as inline SVG from one table of measurements. Not a PNG mockup, not a
stock image: an SVG scales to any width, themes with your tokens, costs about a
kilobyte, and the geometry stays honest because every part is derived rather
than eyeballed.

---

## One source of measurements

Everything comes from the real device, in points, and every other number is
computed from it. Change one value and the whole frame stays consistent.

```js
export const IPHONE_15 = Object.freeze({
  screen: Object.freeze({ width: 393, height: 852, radius: 55 }),
  bezel: 19.6,
  railEdge: 3,
  buttonDepth: 5,
  island: Object.freeze({ width: 126, height: 37, top: 11 }),
  statusBar: 59,
  safariBar: 83,
  buttons: Object.freeze({
    action:     Object.freeze({ side: 'left',  top: 188, height: 44 }),
    volumeUp:   Object.freeze({ side: 'left',  top: 271, height: 70 }),
    volumeDown: Object.freeze({ side: 'left',  top: 356, height: 70 }),
    power:      Object.freeze({ side: 'right', top: 263, height: 109 }),
  }),
});

export const PHONE_BODY = Object.freeze({
  width:  IPHONE_15.screen.width  + IPHONE_15.bezel * 2,
  height: IPHONE_15.screen.height + IPHONE_15.bezel * 2,
  radius: IPHONE_15.screen.radius + IPHONE_15.bezel,
});

export const PHONE_BOX = Object.freeze({
  width:  PHONE_BODY.width + IPHONE_15.buttonDepth * 2,
  height: PHONE_BODY.height,
});
```

Three boxes, and the distinction matters: `screen` is the display, `BODY` adds
the bezel, `BOX` adds the side buttons that stick out. Lay out against `BOX`,
or the buttons get clipped.

The 393x852 and the 126x37 island are the real iPhone 15 numbers. Use the real
ones for whatever device you are framing. A frame with invented proportions is
the thing people notice without being able to say why.

---

## What the video is actually sized to

This is the subtle part. The recording does not fill the screen, it fills the
screen **minus the status bar and minus the browser bar**:

```js
export const PHONE_CONTENT = Object.freeze({
  width:  IPHONE_15.screen.width,
  height: IPHONE_15.screen.height - IPHONE_15.statusBar - IPHONE_15.safariBar,
});

export const CAPTURE_SCALE = 2;
export const PHONE_VIDEO = scaled(PHONE_CONTENT, CAPTURE_SCALE);
```

So the recording viewport is `PHONE_CONTENT`, and the encoder output is that
times the capture scale. Record at the content size and the video drops into
the frame with no letterboxing and no scaling blur.

Get this wrong and you either squash the video or pad it, and both look
immediately cheap. The recorder and the frame must import the same constants:
that shared import is the whole point.

---

## Everything in percentages

The frame has to work at any width, so positions are percentages of the box,
never fixed pixels:

```js
export const pct = (part, whole) => `${Number(((part / whole) * 100).toFixed(3))}%`;

const SCREEN_STYLE = {
  left:   pct(bezel + buttonDepth, PHONE_BOX.width),
  right:  pct(bezel + buttonDepth, PHONE_BOX.width),
  top:    pct(bezel, PHONE_BOX.height),
  bottom: pct(bezel, PHONE_BOX.height),
  borderRadius: `${pct(screen.radius, screen.width)} / ${pct(screen.radius, screen.height)}`,
};
```

The two-value `border-radius` is deliberate. A single percentage on a
non-square box gives you an egg; the `x / y` form keeps the corner round at
every width.

The outer element just holds the aspect ratio, so the frame can be dropped into
any column and stay correct:

```jsx
const BOX_STYLE = { aspectRatio: `${PHONE_BOX.width} / ${PHONE_BOX.height}` };

export default function IPhoneFrame({ children }) {
  return (
    <div className="relative w-full" style={BOX_STYLE}>
      <PhoneShell />
      <div className="absolute flex flex-col overflow-hidden bg-white" style={SCREEN_STYLE}>
        <IosStatusBar />
        {children}
        <SafariBar />
      </div>
    </div>
  );
}
```

---

## The shell: three rects and a gradient

The body is not a flat rounded rectangle. What makes it read as metal is three
stacked shapes:

```jsx
{/* 1. the rail, a diagonal gradient */}
<rect x={buttonDepth} y="0" width={PHONE_BODY.width} height={PHONE_BODY.height}
      rx={PHONE_BODY.radius} fill={`url(#${gradientId})`} />

{/* 2. the black face, inset by the rail edge */}
<rect x={buttonDepth + railEdge} y={railEdge}
      width={PHONE_BODY.width - railEdge * 2} height={PHONE_BODY.height - railEdge * 2}
      rx={PHONE_BODY.radius - railEdge} fill="#09090b" />

{/* 3. a hairline highlight between them */}
<rect x={buttonDepth + railEdge / 2} y={railEdge / 2}
      width={PHONE_BODY.width - railEdge} height={PHONE_BODY.height - railEdge}
      rx={PHONE_BODY.radius - railEdge / 2}
      fill="none" stroke="#a1a1a6" strokeOpacity="0.28" strokeWidth="0.8" />
```

The gradient runs corner to corner with five stops that go dark, light, dark,
because a metal rail catches light unevenly. A two-stop gradient reads as
plastic.

Note `rx` shrinking with each inset. Keep the same radius on a smaller rect and
the corners stop being concentric, which is visible even if nobody can name it.

Two shadows, not one, because a phone sits in space: a wide soft one for the
drop, a tight one for contact.

```
drop-shadow(0 28px 36px rgba(15,23,42,0.28))
drop-shadow(0 3px 6px rgba(15,23,42,0.18))
```

Side buttons are rects that overlap the body by `buttonDepth + 2` so there is
no seam where they meet the rail.

---

## The chrome sells it more than the body

The status bar and the browser bar are what make it read as a real screenshot,
and they are cheap to draw: time, Dynamic Island, signal, wifi, battery, then a
Safari bar with a lock, your domain and a reload glyph, plus the home
indicator.

Details that matter:

- **9:41.** Apple's own placeholder time in every keynote and mockup. A real
  clock, or a random time, reads as a screenshot someone forgot to clean up.
- **Your real domain in the address bar.** It is free advertising and it proves
  the capture is yours.
- **Measured icons, not an icon font.** Signal is four rects of increasing
  height, battery is a rounded rect plus an inner fill plus a nub. Twenty
  lines, no dependency, scales perfectly.
- **Each bar is its own SVG with its own viewBox**, sized in screen points,
  stacked in a flex column with the video between them. Each scales
  independently and the video never overlaps the chrome.

```jsx
<rect data-testid="dynamic-island"
      x={(screen.width - island.width) / 2} y={island.top}
      width={island.width} height={island.height}
      rx={island.height / 2} fill="#000" />
```

Centred by computation, pill radius = half the height. Never a hand-tuned
offset: it breaks the moment a measurement changes.

---

## Desktop

Same idea, less of it: a toolbar with traffic lights, an address pill with your
domain, a 52px bar with a 10px radius, and the video below it. A wider viewport
(960x600 or 1280x800) and an output scale that gives the encoder more pixels
than the frame displays, so text stays crisp.

Resist drawing a whole laptop. The bezel, keyboard and hinge eat the space the
product should occupy. A browser frame says "this is a web app" with a fraction
of the pixels.

---

## Rules

- **One constants file, imported by both the recorder and the frame.** If they
  can disagree about the content size, one day they will.
- **Derive, never eyeball.** Every position computed from the measurements. The
  moment you hand-tune one offset, the next change breaks it silently.
- **Real device numbers.** Invented proportions are the tell.
- **SVG, not PNG.** Scales, themes, weighs nothing, stays editable.
- **9:41.** Just use it.
- **Test the geometry.** Assert the island is centred, the screen inset equals
  the bezel, the content height equals screen minus both bars. These are the
  bugs you cannot see in a thumbnail but can see on a 4K monitor.
- **`aria-hidden` on all of it.** It is decoration around the real content; a
  screen reader should hear the product, not a drawing of a phone.
