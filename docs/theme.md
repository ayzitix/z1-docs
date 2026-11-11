# Theme

The theme lives in its own repository and is vendored into the editor, where it
is the default. This page records the rules it follows, so a future change does
not undo them.

## The three themes

| Theme | Type | Editor background | Accent |
| --- | --- | --- | --- |
| `z1 Ocean` | light (`uiTheme: vs`) | `#FFFFFF` | black |
| `z1 Default` | dark | `#1A1A1A` | white |
| `z1 Soft` | dark | `#242424` | white |

`z1 Default` is the editor's default dark theme, set as a constant in
`workbenchThemeService.ts`. It is not a setting a user has to find.

## Rules

**White surfaces, black accents.**
Every surface is `#FFFFFF`. Black is the only accent, and it appears as a rule, a
border or an inverse block — never as a tint.

**No hue that is not a state.**
Interface greys are channel-neutral: `#5A5A5A`, not `#5A6773`. The only colours
allowed to carry a hue are the ones that mean something:

| Meaning | Colour |
| --- | --- |
| error | `#C0392B` |
| warning | `#8A6A00` |
| added (git, diff, gutter) | `#2E7D4F` |

**Three steps of text.**
`#000000` for primary text, `#6E6E6E` for secondary, `#C4C4C4` for comments. A
fourth step is a mistake.

**Selection is a tint, not a block.**
`#ECECEC` with black text. Hover is lighter still (`#F4F4F4`), so the three
states of a row — normal, hover, selected — stay distinguishable.

**Cover the inheritance chains, not the visible surfaces.**
A key the theme does not set falls back to the registry default for that theme
type, and the light defaults are blue and dark. The chain that matters most:

```
editor.background ← panel.background, tab.activeBackground, breadcrumb.background
focusBorder ← list.focusOutline, tab.selectedBorderTop, sash.hoverBorder
list.activeSelectionBackground ← menu.selectionBackground, quickInputList.focusBackground
editorWidget.background ← quickInput.background, editorHoverWidget.background
```

`z1 Ocean` sets 458 keys. That number is not over-engineering; it is what it
takes to stop the defaults from leaking.

**Each theme is self-contained.**
No `include` chains. A chain that points at a file outside the package renders
half a theme with no error at all.

## How to change a colour

1. Edit `themes/<theme>.json`.
2. `bun script/check-colors.ts` — must print
   `no blue in any interface color`.
3. `python3 script/make-images.py` — regenerates the palette card.
4. `npm run package` — builds the VSIX.
5. Restart the editor window and look at the actual element.

Step 5 is not optional. `vsce` does not validate a theme, and a key can be
perfectly correct while the element it should colour is drawn by the editor
instead. The list of those elements is in [gotchas.md](gotchas.md).

## What the check enforces

`script/check-colors.ts` is the project's own gate, because `vsce` has none. It
fails when:

- a theme has an `include` chain;
- a value is not a 6- or 8-digit hex colour;
- a colour reads blue after being composited onto its own surface — this catches
  translucent values, which a plain hex scan misses;
- a light theme grows a dark surface.

Colours that carry meaning are exempt: error and warning states, git decorations,
gutters, the terminal palette, and badges.

## Vendoring into the editor

The editor keeps its own copies in
`extensions/theme-defaults/themes/z1-*.json` and registers them in
`extensions/theme-defaults/package.json`. When the theme repository changes, the
copies have to be updated deliberately — they do not sync — and the extension
cache may need clearing before the change appears.

The editor also sets the initial colours used while the real theme loads:
`COLOR_THEME_DARK_INITIAL_COLORS` in `workbenchThemeService.ts`. Without it the
window flashes the upstream blue for a moment at startup.
