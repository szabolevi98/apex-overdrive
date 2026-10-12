# Levente Racing social cover

The English Open Graph / Twitter cover is `web/assets/og-cover-en.jpg`.
Final output: **1200 × 630 pixels**, RGB JPEG, quality 90, optimized and progressive.
Its public URL is `https://racing.levente.net/assets/og-cover-en.jpg`.

## Reference images

- `racing.png`: the original gameplay screenshot in the repository root. The
  portfolio uses the same screenshot. Use it as the background reference when
  regenerating the cover from scratch.
- `web/assets/og-cover.jpg`: the original Hungarian cover. Use it as the edit
  target for the English translation prompt below.
- `web/assets/og-cover-en.jpg`: the final English cover. Use it as the reference
  for future variations that should retain the current design.

## Prompt: regenerate from the gameplay screenshot

Attach `racing.png` and use this English prompt:

> Create a polished Open Graph social share cover for Levente Racing, a
> browser-based racing game. Landscape format, exactly 1200 × 630 pixels,
> aspect ratio 1.90476:1. Use the attached real gameplay screenshot as the
> background reference. Preserve the recognizable red Formula racing car,
> Hungaroring pit building, sky, and original game-rendered appearance.
> Remove the minimap HUD from the bottom left.
>
> Add a dark navy gradient overlay, strongest on the left and fading toward
> the right, with subtle diagonal racing stripes in accent blue #4d8dff.
> Keep the red car clearly visible in the lower center-right. At the top left,
> place a small white italic L inside a rounded blue square, matching the
> site's favicon. Below it, set the large bold italic sans-serif title on
> two lines: "LEVENTE" in white and "RACING" in blue.
>
> Directly below the title, place the exact subtitle "Browser-based racing"
> in smaller white sans-serif type. Along the bottom left, place the exact
> labels "Single player", "Time trial", and "Multiplayer", separated by
> small blue dots. In the top right, place the muted URL "racing.levente.net".
> Keep text at least 45 pixels inside the edges and legible at thumbnail size.
>
> Use only the specified text. Do not invent cars or scenery, add buttons or
> fake interfaces, change the game-rendered look, or add borders or watermarks.

## Prompt: translate the original cover

This is the exact prompt used with the built-in image generation tool for
the English version. Attach `web/assets/og-cover.jpg`:

> Edit the provided Levente Racing Open Graph cover to English. Preserve the entire design, composition, typography style, title, blue L logo, colors, gradients, blue diagonal accents, URL, red Formula racing car, Hungaroring pit building, sky and game-rendered appearance. Change ONLY the Hungarian text: replace 'Böngészős autóverseny' with 'Browser-based racing'; replace 'Egyjátékos' with 'Single player'; replace 'Időmérés' with 'Time trial'. Keep 'LEVENTE', 'RACING', 'Multiplayer', and 'racing.levente.net' exactly as they are. Keep subtitle beneath title and the three mode labels along the bottom left with the same small blue dot separators. Preserve all other details. Text must be spelled exactly and legible. Landscape 1200 x 630 pixels, aspect ratio 1.90476:1. No new text or objects, no cropping, no watermark.

## Export and verify

1. Inspect the generated image: exact spelling, complete car, correct logo,
   no minimap, readable text, and no clipped content.
2. If the generator returns a larger image, resize to 1200 × 630 using Lanczos.
   Export as RGB JPEG with quality 90, optimized and progressive.
3. When replacing a deployed cover, use a new filename (for example
   `og-cover-en-v2.jpg`) and update `og:image`, `og:image:secure_url`, and
   `twitter:image` in `web/index.html` so cached images can refresh.
4. Keep `og:image:width`, `og:image:height`, `og:image:type`, and both image
   alt descriptions consistent with the actual file.
5. Verify public HTTPS access, `image/jpeg` content type, and a matching
   SHA-256 between the deployed file and the local export.

The title, description and cover URL are static HTML metadata so social
sharing crawlers can read them without running JavaScript.
