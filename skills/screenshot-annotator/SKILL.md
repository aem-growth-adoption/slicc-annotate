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
  → Sprinkle requests screenshot via slicc.lick({action:'request-screenshot'})
  → Owning scoop uses Chrome tabs API (extension mode) to capture visible tab
  → Scoop pushes base64 data URL back via sprinkle send annotate '{"baseImage":"data:..."}'
  → User annotates on canvas overlay
  → User clicks "Capture" → sprinkle composites base + annotations into JPEG
  → Sprinkle sends slicc.lick({action:'capture', dataUrl:'data:image/jpeg;base64,...'})
  → Cone receives lick and attaches the image to the current chat input
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
| `request-screenshot`  | Sprinkle  | Scoop handles via Chrome tabs API; pushes base64 back to sprinkle           |
| `capture`             | Sprinkle  | Contains `dataUrl` — cone attaches as pending image to chat input           |
| `close`               | Sprinkle  | Dismiss sprinkle                                                            |

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
- Capture (composite + send lick)
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

1. Run: read_file /workspace/skills/sprinkles/style-guide.md
2. Run: read_file /workspace/skills/screenshot-annotator/sprinkle-reference.md
3. Write the sprinkle to /shared/sprinkles/annotate/annotate.shtml
   — Use full-document mode (<!DOCTYPE html>)
   — Follow the structure in sprinkle-reference.md exactly
   — Icon: <link rel="icon" href="pencil" />
4. Run: sprinkle open annotate
5. Stay ready for lick events forwarded by the cone.
   - action='request-screenshot': Use Chrome tabs API to capture, push back via sprinkle send
   - action='capture': Contains dataUrl; acknowledge to cone
   - action='close': Run sprinkle close annotate
Do not send a completion message.
```

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
- Don't run screencapture from the cone — the scoop handles capture via Chrome API.
- Don't use `onclick` attributes — use `addEventListener` inside an IIFE.
