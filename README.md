# slicc-annotate

A screenshot annotation skill for [SLICC](https://github.com/ai-ecoverse/slicc) — the browser-native AI agent runtime.

## What it does

Captures the active browser tab, lets you draw annotations (pencil, highlighter, rectangle, ellipse, arrow, text), then attaches the composited image to your chat message.

## Structure

```
skills/screenshot-annotator/
├── SKILL.md                  # Skill definition (triggers, tool list, scoop brief)
└── sprinkle-reference.md     # HTML architecture reference

sprinkles/annotate/
└── annotate.shtml            # The full-document sprinkle UI
```

## Installation

Copy into your SLICC workspace:

```bash
# Skill
cp -r skills/screenshot-annotator /workspace/skills/

# Sprinkle
cp -r sprinkles/annotate /shared/sprinkles/
```

Or mount this repo directly:

```bash
mount git https://github.com/aem-growth-adoption/slicc-annotate.git /workspace/skills/screenshot-annotator
```

## Usage

1. Open the annotate sprinkle from the rail (pencil icon)
2. Click "Capture Screenshot" — select a tab/window to capture
3. Draw annotations using the toolbar tools
4. Click "Attach to Chat" — the annotated image is attached to your next message

## Tools

| Tool | Behavior |
|------|----------|
| Pencil | Freehand path, 3px stroke |
| Highlighter | Thick (20px) semi-transparent freehand path |
| Rectangle | Drag to draw outline rect |
| Ellipse | Drag to draw outline ellipse |
| Arrow | Drag to draw line with arrowhead |
| Text | Click to place, type, Enter commits |

## Keyboard Shortcuts

- `1`–`6` — switch tools
- `Ctrl+Z` / `Cmd+Z` — undo
- `Escape` — clear all strokes

## License

MIT
