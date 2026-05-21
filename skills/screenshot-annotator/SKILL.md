---
name: screenshot-annotator
description: |
  Use this when the user wants to take a screenshot of the browser tab,
  annotate it with drawing tools (pencil, highlighter, rectangle, ellipse,
  arrow, text), and attach the result as an image to the chat. Covers the
  full flow: trigger → sprinkle opens → canvas annotation → capture JPEG →
  attach to chat input. Also use when the user says things like "mark up a
  screenshot", "draw on a screenshot", "annotate and send", or "screenshot tool".
allowed-tools: bash, read_file, write_file, edit_file
---

# Screenshot Annotator

A sprinkle-based tool that captures the active browser tab, lets the user draw
annotations on a full-viewport canvas, then sends the composited image back as
a JPEG data URL for attachment to the chat input.

## Architecture

```
User triggers annotation (lick button / command / keyboard shortcut)
  → Sprinkle "annotate" opens as full-document .shtml
  → Empty state shown: "Capture screenshot" or "Paste image (Cmd+V)"
  → User clicks Screenshot → slicc.lick({action:'request-screenshot'})
  → Cone runs screencapture, resizes, pushes base64 via sprinkle send annotate '{"baseImage":"data:..."}'
  → OR user pastes image from clipboard (Cmd+V) — loaded directly onto canvas
  → User annotates on canvas overlay
  → User clicks "Attach to Chat" → sprinkle composites base + annotations into JPEG
  → Sprinkle calls slicc.attachImage(dataUrl) to attach directly to chat input
  → Sprinkle stays open (user switches to chat tab manually)
```

The sprinkle lives at `/shared/sprinkles/annotate/annotate.shtml`.

For full HTML structure details, read the companion file:
`read_file /workspace/skills/screenshot-annotator/sprinkle-reference.md`

## Starting the tool

The cone creates (or re-uses) the `annotate` scoop and sprinkle.

```
scoop_scoop("annotate")
feed_scoop("annotate", "<scoop brief — see template below>")
```

If the scoop already exists (`list_scoops`), just `feed_scoop` the existing one.

## Lick event routing

| Lick `action`         | Origin    | Cone action                                                                 |
| --------------------- | --------- | --------------------------------------------------------------------------- |
| `request-screenshot`  | Sprinkle  | Cone runs screencapture, resizes to ≤1568px, pushes base64 back via sprinkle send |
| `close`               | Sprinkle  | Dismiss sprinkle via `sprinkle close annotate`                              |

Note: The "Attach to Chat" button calls `slicc.attachImage(dataUrl)` directly — no lick event
is sent to the cone for attachment. If `slicc.attachImage` is unavailable, falls back to
`slicc.lick({action:'capture', dataUrl:'...'})` but the dataUrl may be truncated by the bridge.

## Drawing tools

| Tool        | Behavior                                      |
| ----------- | --------------------------------------------- |
| Pencil      | Freehand path, 3px stroke                     |
| Highlighter | Thick (20px) semi-transparent freehand path   |
| Rectangle   | Drag to draw outline rect                     |
| Ellipse     | Drag to draw outline ellipse                  |
| Arrow       | Drag to draw line with arrowhead              |
| Text        | Click to place, type, Enter commits fillText  |

## Toolbar

- Tool selector buttons (inline SVG icons)
- Color picker (6 preset swatches: red, blue, green, yellow, white, black — default red)
- Undo (pop last stroke)
- Clear (remove all strokes)
- Screenshot (request new base image from cone)
- Attach to Chat (composite + call slicc.attachImage)
- Close (dismiss without capturing)

Shadow DOM element, position fixed top center, z-index 2147483647, dark translucent
backdrop-blur background (rgba(30, 30, 50, 0.92)), pill-shaped buttons.

## Keyboard shortcuts

- Escape — close/cancel
- 1-6 — switch tools (pencil, highlighter, rectangle, ellipse, arrow, text)
- Ctrl+Z — undo

## Output format

JPEG, 0.85 quality, capped at 1568px long edge, <5MB. If result >5MB, downscale proportionally.

## Scoop brief template

```
You own the sprinkle 'annotate'. Your job:

1. Run: read_file /workspace/skills/screenshot-annotator/sprinkle-reference.md
2. Check if /shared/sprinkles/annotate/annotate.shtml already exists. If it does, skip to step 4.
3. Write the sprinkle to /shared/sprinkles/annotate/annotate.shtml
   — Use full-document mode (<!DOCTYPE html>)
   — Follow the structure in sprinkle-reference.md exactly
   — Icon: <link rel="icon" href="pencil" />
4. Run: sprinkle open annotate
5. Stay ready for lick events forwarded by the cone.
   - action='request-screenshot': Cone handles this directly (runs screencapture, pushes back)
   - action='close': Run sprinkle close annotate
Do not send a completion message.
```

## Cone handling of request-screenshot

When the cone receives `{action: 'request-screenshot'}` from the annotate sprinkle:

```bash
screencapture /shared/annotate-capture.png
magick /shared/annotate-capture.png -resize 1568x -quality 85 /shared/annotate-capture.jpg
base64 /shared/annotate-capture.jpg | tr -d '\n' > /tmp/b64.txt
DATAURL="data:image/jpeg;base64,$(cat /tmp/b64.txt)"
sprinkle send annotate "{\"baseImage\":\"$DATAURL\"}"
```

The cone handles this directly (not via scoop) because it has full filesystem access
and `screencapture` requires the cone's execution context.

## Key implementation details

- **Stroke model**: Each gesture produces a Stroke object stored in an array for undo/redraw.
- **Live preview**: During drag, redraw all committed strokes then render in-progress shape on top.
- **Compositing**: Offscreen canvas — draw base image first, then annotation canvas scaled to fit.
- **HiDPI**: Canvas sized by viewport × devicePixelRatio for crisp rendering.
- **Coordinate handling**: Pointer events adjusted via getBoundingClientRect().
- **Dedup guard**: slicc.lick() fires duplicates — use timestamp cooldown pattern.
- **Shadow DOM toolbar**: Isolated from page styles, always on top.

## Don't

- Don't create a second scoop if `annotate` already exists — feed the existing one.
- Don't run screencapture from a scoop — scoops don't have write access to `/shared/`. The cone handles capture directly.
- Don't use `onclick` attributes — use `addEventListener` inside an IIFE.
- Don't auto-request a screenshot on sprinkle load — show the empty state instead.
- Don't auto-close the sprinkle after attach — let the user switch tabs manually.
