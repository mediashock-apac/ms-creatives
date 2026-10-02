# MediaShock APAC — Creatives

An AI content studio: type a brief, and it produces LinkedIn-ready assets — a multi-page PDF for
carousels, PNGs for image posts, and transparent PNGs for video title and end cards. Claude writes
the copy, Gemini generates the background imagery, and a canvas renderer composites them in the
brand system.

It is **not** an editor, and not a replacement for Premiere or After Effects. It's the front half
of production: brief in, on-brand asset out.

**Live:** https://mediashock-apac.github.io/ms-creatives/

## Access and API keys

Unlike the other two tools, this one has **no sign-in**. It's a static page with no backend, so
anyone with the link can open it — but it does nothing until you supply your own API keys.

Each person pastes their own Anthropic and Google AI keys into the API keys dialog. They're stored
in your browser's `localStorage` and never leave your machine. There is deliberately **no shared
team key**: the app is a static page, so anything in the source would be public to anyone who
opens it.

## What it does

1. **Brief** — describe the post. Optionally attach up to three reference screenshots under
   *Inspiration* to steer the art direction.
2. **Generate** — Claude writes the copy and picks a layout per slide; Gemini generates background
   imagery to match.
3. **Edit** — adjust any headline, body line or layout in the slide editor before exporting.
4. **Export** — a carousel PDF, individual PNGs, or transparent PNGs for video cards.

Drafts are saved locally under `mscreatives_drafts`. Note that generated images are stored inline,
and browsers cap `localStorage` at around 5MB per site — a handful of image-heavy drafts will fill
it, and the app will tell you plainly when that happens rather than failing silently.

## The rule the whole design rests on

**The image model never renders text.** Image models still mangle letters, kerning and logos, and
a carousel is mostly text — a fully generated slide looks almost right and is unusable. So the
split is fixed: Claude writes the words, Gemini generates only background imagery, and the renderer
draws all type itself at full resolution in the brand fonts.

Don't add a feature that lets the image model produce the headline.

## Layouts

Output is 1080×1350 (LinkedIn's 4:5 portrait). Claude picks one of four compositions per slide, so
a day's output doesn't share the same structural rhythm:

- **Bottom** — full-bleed photo, headline and body anchored at the base. The default, and the best
  fit for a hook slide.
- **Split** — image fills roughly half the frame, text on a solid panel beside it. Alternates sides
  so a deck doesn't lean the same way twice.
- **Quote** — centred pull-quote, no image required.
- **Stat** — a big number or short claim anchored near the top.

The wordmark is always the Mediashock orange in a fixed position, and a photo background always
gets a scrim before type is drawn. Those don't vary per layout — the brand mark must not drift per
topic or per composition.

## Where this sits next to the other two tools

- **MS Creatives** makes the asset.
- **[MS LinkedIn Hub](https://mediashock-apac.github.io/ms-linkedin-hub/)** schedules and measures
  the post. Export from here, drop the file in Drive, and paste the Drive link into the Hub's
  Images field.
- **[MS Project Manager](https://mediashock-apac.github.io/ms-project-manager/)** tracks the
  production work behind it.

Keep those jobs in their own tools — don't add a production checklist or a kanban board here.

## Local development

There's no build step. Serve the folder over `http://` rather than opening the file directly, so
`version.txt` polling has something to fetch:

```
python -m http.server 8791
```

The app is one inline `<script>`, so a single syntax error anywhere kills the whole page. Run the
check before pushing:

```
node scripts/check-syntax.mjs
git config core.hooksPath .githooks   # one-time per clone: runs the check on every push
```

## Deploying

Push to `main` — GitHub Pages serves the repo root directly, so the live site updates on its own.

Before pushing a change worth forcing a reload for, **bump both** `version.txt` and
`CURRENT_BUILD_VERSION` (near the bottom of the inline script) to the same UTC timestamp:

```
date -u +%Y-%m-%dT%H:%M:%SZ
```

Unlike the LinkedIn Hub, there's no script that stamps these for you — both are hand-edited.
Bumping only `version.txt` means open tabs reload in a loop; bumping only `CURRENT_BUILD_VERSION`
means they never notice this deploy at all. Open tabs are never reloaded out from under an open
modal or an in-progress generate/export.

## Tech stack

- Plain HTML, Tailwind CSS (via CDN) and vanilla JavaScript — no framework, no bundler, no build
  step, and no dependencies. The PDF writer is hand-written rather than pulling in a library.
- The Anthropic API, called directly from the browser for copy.
- The Google AI API for background imagery.
- No Firebase, no database, no server.

## Contributing

`CLAUDE.md` in this repo is the deep reference — the renderer, the prompt structure, the PDF
writer's fragile parts, and the gotchas worth knowing before changing anything. Read it first. The
patterns shared across all Mediashock internal tools live in the `CLAUDE.md` one folder up.
