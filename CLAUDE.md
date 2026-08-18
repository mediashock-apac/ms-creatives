# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Patterns shared across every Mediashock internal tool (notifications, theme toggle, icons, auth,
> Firestore rules gotchas) live in `CLAUDE.md` in the parent `Claude Projects/` folder — check there
> before building something this project's own architecture below doesn't cover.

## What this is

An AI content studio: type a brief, Claude writes the copy, Gemini generates the imagery, and a
canvas renderer composites them into LinkedIn-ready assets — a multi-page PDF for carousels, PNGs
for image posts, transparent PNGs for video title/end cards.

It is **not** an editor and not a replacement for Premiere/After Effects. It's the front half of
production: brief in, on-brand asset out.

## Where this sits next to the other two tools

- **MS Creatives** makes the asset.
- **MS LinkedIn Hub** schedules and tracks the post. Export from here, drop the file in Drive, paste
  the Drive link into the Hub's Images field (it auto-detects Drive links and renders a folder chip).
- **MS Project Manager** tracks the actual production work (shoots, edits, grades).

Don't fold any of those jobs into this one. In particular, don't add a production checklist or a
kanban here — the Hub already had an assignable checklist removed for being clutter, and Flowboard
is where production tasks belong.

## Deliberate deviations from the house pattern

Three, each with a reason:

1. **No Firebase, no auth, no Firestore.** There's no Firebase project for this yet, and the core
   loop (generate → render → export) needs no shared state or server. Drafts live in `localStorage`
   under `mscreatives_drafts`. If team-shared drafts are wanted later, that's when a Firebase project
   gets created — the payload written by `saveDraft()` is already a flat JSON document that would
   drop into a Firestore collection unchanged. Note the one real limit: `localStorage` caps around
   5MB per origin and generated images are stored as base64 data URLs inside the draft, so a handful
   of image-heavy drafts will fill it. `writeDrafts()` catches the quota error and says so rather
   than failing silently.
2. **Single `index.html`, hand-edited.** Unlike the Hub, there is no canonical-source/generated-file
   split and no `sync_from_scratchpad.py`. Edit `index.html` directly. That split exists in the Hub
   for historical reasons and is a wart, not a pattern worth copying.
3. **`version.txt` polling exists, but is hand-stamped, not script-generated.** Unlike the Hub
   (`sync_from_scratchpad.py` stamps both files automatically), this app has no build step at all —
   so `CURRENT_BUILD_VERSION` near the bottom of the inline script and `version.txt` are both
   **hand-edited** on any change worth forcing a reload for. Bump both to the same UTC timestamp
   (`date -u +%Y-%m-%dT%H:%M:%SZ`) before pushing; forgetting `version.txt` means open tabs never
   notice the deploy, forgetting `CURRENT_BUILD_VERSION` means every tab reloads on the *next*
   change instead of this one. Same `anyModalOpen()`-gated reload logic as both sibling apps —
   never reloads out from under an open modal or an in-progress generate/export.

## Deployment

Repo: https://github.com/deane-ms/ms-creatives — hosted on GitHub Pages. Run locally with:

```
python -m http.server 8791
```

`file://` mostly works too, but serve over `http://` to match how it really runs (and so
`version.txt` polling has something to fetch).

## API keys are per-person, in the browser

Each teammate pastes their own keys into the API keys dialog; they're stored in `localStorage` under
`mscreatives_keys` and never leave the machine. **Never commit a key, and never add a shared/default
key to this file** — the app is a static page, so anything in the source is public to anyone who
opens it. The dialog says this in plain language; keep that warning if you touch that modal.

- **Anthropic (copy)** — `claudeJson()` posts to `/v1/messages` with
  `anthropic-dangerous-direct-browser-access: true`. **Without that header the request is blocked by
  CORS**, which is the only reason a browser can call this API without a proxy at all. Model is
  `CLAUDE_MODEL`.
- **Google AI (imagery)** — `geminiImage()` posts to `/v1beta/interactions` with `x-goog-api-key`.
  Model is `GEMINI_MODEL` (`gemini-3.1-flash-image`). **Imagen was retired on 2026-08-17** — don't
  "fix" this back to an `imagen-*` model or a `:generateContent` call. `extractImageB64()`
  deliberately handles both the Interactions response shape (`output_image.data`) and the older
  `inlineData.data` shape, so a provider-side rollback doesn't break image generation outright.

Claude is asked for strict JSON and `parseLooseJson()` takes the outermost `{...}` — models
intermittently wrap JSON in prose or a code fence, and a bare `JSON.parse()` throws on that.

## The rule the whole design rests on

**An AI image model never renders text in these assets.** Image models still mangle letters,
kerning and logos, and a carousel is mostly text — a fully generated slide looks almost right and is
unusable. So the split is fixed:

- Claude writes the words.
- Gemini generates only background imagery (every image prompt ends with an explicit
  "no text, words, letters or logos" instruction — keep that if you touch the prompts).
- `drawSlide()` draws all type itself, at full resolution, in the brand fonts.

Don't add a feature that lets the image model produce the headline.

## Rendering

`drawSlide(ctx, slide, index, total, transparent)` is the single renderer — the preview canvas, PNG
export and every PDF page all go through it, so there is no second layout to keep in sync. Output is
1080×1350 (`CANVAS_W`/`CANVAS_H`), LinkedIn's 4:5 portrait.

- **Text is bottom-anchored**, measured before drawing: `wrapText()` wraps against real
  `measureText()` widths, then the block's total height is subtracted from the footer line so a
  three-line headline and a one-line headline both sit correctly above the wordmark.
- **A photo background always gets a scrim** (a bottom-weighted dark gradient) before type is drawn.
  Without it, white text on a light photo is unreadable. Don't skip it for "cleaner" slides.
- **The wordmark is always `BRAND.orange`**, never the bucket accent colour. The bucket colour tints
  the top rule and the slide counter only — the brand mark itself must not drift per topic. (It did
  originally; caught in a render screenshot.)
- **`transparent: true`** (video cards) skips the background fill entirely so the PNG exports with a
  real alpha channel for compositing. It does *not* skip the scrim when there's a photo, since a
  card over footage still needs legible type.
- Slide 1 of a multi-slide deck is treated as the hook: larger headline plus a swipe arrow.

## Brand kit

`BRAND` at the top of the script is the single place to change colours, the wordmark and fonts.
Currently it uses the Mediashock orange (`#ff4e20`) and Zinc neutrals shared with the sibling apps,
and the system font stack.

**The real kit hasn't landed yet.** When the font files arrive, drop the `.woff2` into this folder,
add an `@font-face` block, and point `BRAND.headFamily`/`BRAND.bodyFamily` at it. Canvas can only
draw a font the browser has actually loaded — `await document.fonts.ready` before the first
`drawSlide()` or the first render silently falls back to the system font. A font name alone is not
enough; the file has to be there.

## PDF export is hand-written on purpose

`buildPdf()` emits the PDF bytes directly — no library. One JPEG per page, drawn full-bleed via a
`/DCTDecode` image XObject, at 612×765pt (4:5). This is the only output that would need a
dependency, the format is simple enough to emit directly, and it keeps the app dependency-free like
its siblings.

**If you touch it, the fragile part is the xref table**: every entry is a byte offset that must land
exactly on its object's `N 0 obj` header. The whole string is assembled as latin1 (one char = one
byte) so `out.length` *is* the byte offset — if you ever introduce a multi-byte character into that
string, every offset after it silently shifts and the file becomes unopenable. `dataUrlToLatin1()`
uses `atob` for the same reason.

Verified against a real Chrome PDF viewer, plus byte-level checks that every embedded JPEG is intact
(SOI/EOI) and that each `/Length` exactly matches its stream. Re-run those if you change the writer.

## Gotchas already hit here

- **`[hidden] { display: none !important }` is load-bearing.** `.btn` is `display: inline-flex`, and
  an author `display` rule outranks the UA stylesheet's `[hidden]` rule — so `el(...).hidden = true`
  on a button does nothing without it. This silently left the carousel PDF button visible in
  non-carousel formats.
- **Icon buttons are filled from JS constants at first paint.** `prevSlideBtn`/`nextSlideBtn`/the two
  modal close buttons get their SVG assigned at the bottom of the script. Miss one and it renders as
  an empty square with no error — add any new icon button to that block.
- **Rapid-fire downloads get dropped by browsers.** `exportAllPng()` puts a 350ms gap between saves;
  without it only the first few of a 6-slide deck actually land.
- **Images are read as data URLs, never fetched from a URL.** A cross-origin image taints the canvas
  and makes `toDataURL()` throw, which would break export entirely. This is the opposite of the
  Hub's paste-a-URL approach, and deliberately so — same reasoning (no paid storage), different
  conclusion, because this app has to read the actual pixels.
- **This is a classic `<script>`, not `type="module"`.** Top-level `var`s are therefore on `window`,
  which is what makes the app testable from Playwright (`window.state`, `window.buildPdf`). Don't
  convert it to a module without rewriting the tests.

## Icons

Inline SVG only — never emoji, never an icon font, including in CSS `content:`. Reuse an `ICON_*`
constant before drawing a new one. Convention: `stroke="currentColor"`, `fill="none"`,
`stroke-width="1.8"`–`2.4`, `viewBox="0 0 24 24"`. Same rule as both sibling apps, where a stray
emoji placeholder has slipped through three separate times — treat it as non-negotiable at write
time, not a cleanup pass.

## Syntax check (guards against a one-typo blackout)

The app is one inline `<script>` with no build step, so a single syntax error anywhere kills the
whole page. Flowboard shipped exactly that once and its live board was dead for days.

```
node scripts/check-syntax.mjs
```

Wired into `.githooks/pre-push` (needs `git config core.hooksPath .githooks` once per clone) and
`.github/workflows/syntax-check.yml` as a backstop. `.gitattributes` pins `.githooks/*` to LF so a
CRLF checkout can't silently disable the hook.

## Testing

No unit tests. Verification is Playwright against a real browser, reusing the Hub's cached install
(`MS LinkedIn Hub/.qa-tools/node_modules`) — set `NODE_PATH` to it rather than installing a second
copy. The suite covers canvas rendering (sampled pixels, not screenshots alone), slide navigation,
PNG export, PDF structure, and the transparency of video cards. No API key is needed: state is
injected via `window.state` + `window.renderDraft()`, so nothing paid is ever called in a test run.

**Chrome's PDF viewer is a component extension**, so Playwright's default `--disable-extensions`
turns it off and a PDF navigation becomes a download that closes the tab. Launch with
`ignoreDefaultArgs: ['--disable-extensions', ...]` and serve the file over `http://` — `file://`
downloads it regardless. The viewer also lives in a closed shadow root and leaves `document.title`
empty, so assert on painted pixels (screenshot size), not on DOM contents.
