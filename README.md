# 6×8 Bitmap Font Generator

A single-file browser tool for drawing and generating the `font6x8[95][6]` glyph table used by
the [Gotek Touchscreen interface](https://github.com/dimitrihilverda/Gotek-Touchscreen-interface)
firmware — and by any other Arduino/ESP32 project that renders text as column-encoded bitmaps.

**▶ Run it here: <https://dimitrihilverda.github.io/6x8-bitmap-font-generator/>**

No install, no build step, no dependencies. One HTML file, everything runs in the browser.
Nothing is uploaded anywhere; the only network request is the optional Google Fonts stylesheet.

---

## What it does

* **Pixel editor** — click or drag across a 6×8 grid to toggle pixels, with 1×/3×/6×/10× live zooms.
* **All 95 characters** (ASCII 32–126) in a sidebar, each rendered live as you edit.
* **Text preview** — type a sample string and see the whole font in context, at 2×–8×, with
  configurable pixel and background colours.
* **Per-character tools** — clear, fill, invert, shift in four directions, mirror horizontally
  or vertically.
* **Google Fonts → 6×8** — load any Google Font by name and rasterise it into the grid, either
  for the current character or for all 95 at once, with an adjustable coverage threshold.
* **Import C** — paste an existing font table back in to keep editing it. Any `{0x.., 0x.., …}`
  groups are picked up, so a partial paste works too (missing characters are padded blank).
* **Export C** — emits a ready-to-paste `static const uint8_t font6x8[95][6] PROGMEM = { … }`
  array with an `// <ascii> <char>` comment on every row.

---

## The encoding

Each character is **6 bytes = 6 columns**, left to right. Each byte is a vertical column bitmask:

```
bit 0 (LSB) → top row
bit 1       → row 1
…
bit 7 (MSB) → bottom row
```

So `{0x7E, 0x11, 0x11, 0x11, 0x7E, 0x00}` is a capital `A`: five columns of glyph plus one
empty spacing column. Column 5 is the inter-character gap — the renderer advances a fixed
6 pixels per character, so nothing is proportionally spaced.

The matching renderer in the firmware is `gfx_print()` in `Gotek_Touchscreen.ino`, which walks
`col 0..5` and `row 0..7` and tests `bits & (1 << row)` — exactly the layout above.

### About row 7

The editor dims row 7 and its on-screen help calls it unused, but that is a little conservative:
the firmware does render all 8 rows, and the built-in font already uses row 7 for the descenders
of `g`, `j`, `p`, `q` and `y` (e.g. `g` is `{0x18, 0xA4, 0xA4, 0x9C, 0x78, 0x00}` — `0xA4` has
bit 7 set). You can paint row 7 by hand and it will export and render correctly.

What does ignore row 7:

* **Invert** masks to `0x7F`, so inverting clears any row-7 pixels.
* **Mirror vertical** flips rows 0–6 only and drops row-7 pixels.
* **Google Fonts conversion** always fits a glyph into 5 columns × 7 rows, so converted fonts
  have no descenders below row 6. Redraw those by hand afterwards if you want them.

---

## Typical workflow for the Gotek touchscreen

1. Open the [editor](https://dimitrihilverda.github.io/6x8-bitmap-font-generator/).
2. **Import C** and paste the current `font6x8` block from `Gotek_Touchscreen/Gotek_Touchscreen.ino`
   (or start from the built-in font, which is the same table).
3. Edit, or convert a Google Font and touch up the characters that came out badly.
4. **Export C** → *Copy to Clipboard*.
5. Replace the whole `static const uint8_t font6x8[95][6] PROGMEM = { … };` block in the sketch
   and reflash.

The exported declaration is byte-for-byte the same shape the firmware already uses, so it is a
straight block replacement — no other code changes needed.

## Converting a Google Font

Pick one from the dropdown or type any family name exactly as it appears on
[fonts.google.com](https://fonts.google.com). The font is drawn to an offscreen canvas at 80px,
its ink bounding box is measured, and that box is resampled into a 5×7 cell grid; a cell becomes
a pixel when its average darkness reaches the threshold.

* **Lower threshold → more pixels.** 35% is a reasonable default; thin fonts often need 15–25%.
* Because every glyph is stretched to fill the same 5×7 box, characters with very different
  natural proportions (`.` `,` `'` `|`) come out oversized. Fix those by hand.
* Fonts already designed for a pixel grid — *Press Start 2P*, *VT323* — convert best.
* Conversion needs a working internet connection, and it needs the page served over http(s):
  opening the file straight from disk falls back to a `<link>` injection that some browsers
  block.

## Keyboard

| Key | Action |
| --- | --- |
| `←` / `→` | previous / next character |
| `↑` / `↓` | jump 10 characters (one row in the sidebar) |
| any printable key | jump straight to that character |

Shortcuts are ignored while a text field, textarea or dropdown has focus.

## Running it locally

Just open `index.html` in a browser. For the Google Fonts converter to work reliably, serve it
over http instead:

```bash
python -m http.server 8000
```

Then browse to <http://localhost:8000>.

## Good to know

* There is **no undo and no autosave**. Your work lives in the page — export before you close
  the tab or hit *Reset*.
* *Reset to built-in font* restores the original 95-character table and cannot be undone.
* Import expects 6 values per group; longer groups are truncated, shorter ones zero-padded.

## Licence

No licence file yet — add one if you want other people to reuse the code.
