---
sidebar_position: 2
title: Text Rendering
---

# Text Rendering: Pick the Right Label

Cocos2d-Mono has three ways to put text on screen. Two of them draw glyphs you shipped as
content; the third rasterizes glyphs at runtime from the operating system's fonts. Picking
the runtime one without meaning to is the single biggest "works today, bites you at porting
time" decision in the engine — because the difference only becomes visible when you target
a new platform.

## The three paths

**Bitmap fonts — `CCLabelBMFont`.** Glyphs are pre-rendered into a texture atlas at build
time (a `.fnt` + texture pair produced by tools like BMFont or Hiero, loaded through the
content pipeline). At runtime, drawing text is just drawing sprites.

```csharp
var title = new CCLabelBMFont("Score: 0", "fonts/arial-32.fnt");
```

**Compiled SpriteFonts — `CCLabelTTF`.** Despite the name, this does *not* rasterize a
`.ttf` at runtime. The font argument is a **content key** for a `.spritefont` asset the
content pipeline compiled into a glyph texture — resolved through `CCSpriteFontCache`. It
is content, just like a bitmap font.

```csharp
// "EditUndoBrk" is a content key: Content/fonts/EditUndoBrk-22.xnb
var score = new CCLabelTTF("Score: 0", "EditUndoBrk", 22);
```

**Dynamic system-font labels — `CCLabel`.** Text is rasterized *at runtime* from the
operating system's font stack, then cached into a glyph atlas. Each platform has its own
backend (GDI+/Skia on desktop, CoreGraphics on iOS, the canvas APIs on Android).

```csharp
var chat = new CCLabel(message, "Arial", 22);
```

:::tip Telling `CCLabelTTF` and `CCLabel` apart

They look almost identical at the call site, and the distinction is the whole point of this
page. The tell is what the font string means: `CCLabelTTF` takes a **content key** (no
directory, no extension — an asset you built), while `CCLabel` takes a **system font
family name** (`"Arial"`, `"Segoe UI"`) that must exist on the device.

If a `CCLabelTTF` font key doesn't resolve, the engine falls back to the default *content*
font and, failing that, logs `Failed to load default font` and draws nothing — it never
reaches for a system font.

:::

## The opinion: prefer content-backed text

Dynamic labels depend on the platform having a system font store to rasterize from.
Desktop and mobile do. **Consoles generally do not** — on the console targets explored in
the engine's own porting work, there was no OS font service to call, leaving `CCLabel`
nothing to render with. If your game ships text through `CCLabel` and you later target a
console, expect every one of those call sites to become porting work.

`CCLabelBMFont` and `CCLabelTTF` work identically everywhere the engine runs, because the
glyphs are just data you shipped. A commercial title built on this engine passed console
certification with **all** of its text going through `CCLabelTTF` over compiled
SpriteFonts — no porting work required for text.

**Use `CCLabel` when** the text is genuinely unpredictable — arbitrary user input, wide
Unicode coverage you can't pre-build, editor/tooling UI — and you know every shipping
platform has system fonts.

**Use `CCLabelBMFont` or `CCLabelTTF` for** everything you author yourself: HUDs, menus,
dialogue, scores. If the strings are known (or the alphabet is), content-backed glyphs
cover them. Between the two, bitmap fonts give you the most control over styling and
packing; SpriteFonts are the lower-friction option if you're already building fonts through
the content pipeline.

## Practical notes

- One `.fnt` per size looks crisper than scaling one atlas across many sizes; generate the sizes you actually use. The same applies to `.spritefont` assets — `CCSpriteFontCache` keys on name *and* size.
- `CCLabelBMFont` supports width-constrained layout and alignment via its constructor overloads (`width`, `CCTextAlignment`).
- Building for multiple languages? Content-backed fonts need the glyph ranges baked in, so a localized game typically ships one font set per script (Latin, Cyrillic, CJK) and swaps the content key when the language changes.
- The label scenes in the test app exercise every backend side by side — run it on your target platform when in doubt.
