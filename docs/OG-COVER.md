# Apex Overdrive social cover

The current English Open Graph / Twitter cover is `web/assets/apex-overdrive-cover-en.jpg`.
Final output: **1200 × 630 pixels**, RGB JPEG, quality 90, optimized and progressive.
Public URL: `https://racing.levente.net/assets/apex-overdrive-cover-en.jpg`.

The domain remains `racing.levente.net`. The logo is a white italic **A** in a
rounded blue square. The title reads white **APEX** above blue **OVERDRIVE**.
The former `og-cover-en.jpg` and `og-cover.jpg` URLs are compatibility copies
of the current English cover, so old shared links also receive current branding.
`racing.png` in the repository root remains the original gameplay reference.

## Image generation

Rebranded with the built-in image generation tool using the previous English
cover as the edit target and this exact prompt:

> Use case: text-edit. Edit target: the attached English racing game cover. Rebrand the game to APEX OVERDRIVE. Preserve the Hungaroring pit building, sky, original game-rendered red Formula racing car, road, navy gradient, blue diagonal accents and overall visual style. Replace the blue square's white italic L with a white italic A. Replace the title LEVENTE RACING with two lines: APEX in bold italic white, OVERDRIVE in bold italic blue. OVERDRIVE must fit completely, resizing/rebalancing the typography if necessary, no overlap with the car. Keep exact subtitle 'Browser-based racing', exact labels 'Single player', 'Time trial', 'Multiplayer', and exact URL 'racing.levente.net'. Landscape social cover aspect ratio 1200:630, all text at least 45px inside edges at final 1200x630 size. No other text, no watermark, no cropping of car, preserve recognizable scenery. Return the edited image and save the output as a local file.

## Export and verify

1. Inspect exact spelling, A logo, complete car, recognizable scenery and unclipped text.
2. Resize to 1200 × 630 with Lanczos; export RGB JPEG at quality 90, optimized and progressive.
3. Update the compatibility copies when replacing the current cover.
4. Use a new public filename for future cover changes to refresh social caches.
5. Keep the static Open Graph / Twitter URLs, dimensions and alt descriptions
   in `web/index.html` consistent with the actual file.

## Portfolio

The portfolio full source is `web/assets/img/projects/apex-overdrive.png`
(1200 × 630). Its dedicated thumbnail source is
`web/assets/img/projects/thumbs/apex-overdrive.png` (**500 × 300**, 5:3).
The thumbnail is composed separately; do not center-crop the full cover to make it.
Its generation prompt and safe-area requirements are in the portfolio's
`tools/apex-overdrive-thumbnail-prompt.md`.
