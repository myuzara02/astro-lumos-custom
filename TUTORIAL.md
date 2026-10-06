# Tutorial: Lumos custom tool (Figma → tokens → site)

This template turns Lumos for Astro from a fluid scale (`clamp()`) into fixed values per breakpoint, taken from Figma, so the site can match the design 1:1.

## 1. Concepts

| Breakpoint | Width | How it works |
| --- | --- | --- |
| mobile | ≤ 767px | default |
| tablet | 768–991px | `@media (width >= 48rem)` |
| desktop | ≥ 992px | `@media (width >= 62rem)` |

- Every responsive token has three values (unitless px) in `src/styles/base.css`:
  ```css
  --h1: calc((var(--bp-mobile) * var(--h1-mobile) + var(--bp-tablet) * var(--h1-tablet) + var(--bp-desktop) * var(--h1-desktop)) / 16 * 1rem);
  --h1-mobile: 32;
  --h1-tablet: 45;
  --h1-desktop: 54;
  ```
  Changing the design means changing the numbers, never the formula.
- Line height also has three values per breakpoint (`--h1-line-height-mobile/-tablet/-desktop`), not a ratio.
- Spacing, radius and icon tokens use Figma's names: `--space-1-5rem`, `--radius-0-5rem`, `--icon-m`, and so on. Figma's spacing and padding scales are merged into one.
- Colors have two layers: primitives `--color-<group>-<step>` (the Figma palette), then semantic tokens in the theme blocks (`--background`, `--text`, `--brand`).
- Leading text trim is off by default. Turn it on with the `.text-trim` class on an element or its ancestor.
- The measuring tool is the `lumos-import-figma` skill (`.agents/skills/lumos-import-figma/`). Its script `convert.mjs` reads tokens straight from `base.css`, so it keeps no copy that can go stale.

## 2. Starting a project from the template

```bash
gh repo create my-project --template myuzara02/astro-lumos-custom --clone
cd my-project
npm install
npx astro dev --background      # dev server in the background
npx astro dev status            # see the port
npx astro dev stop              # stop it
```

Or click **Use this template** on the GitHub repo page.

The template's defaults (the safron-mango palette, 112px margin, and so on) belong to the design it was built from. The steps below replace them with your own design.

## 3. Connecting Figma to omp

1. Open the Figma desktop app, open your design file, and switch to **Dev Mode**.
2. Enable the **MCP server** in the right-hand panel of Dev Mode (address `http://127.0.0.1:3845/mcp`).
3. Register it in omp. Add this to `~/.omp/agent/mcp.json` (applies to every project) or `.omp/mcp.json` (this project only):
   ```json
   {
     "mcpServers": {
       "figma-desktop": { "type": "http", "url": "http://127.0.0.1:3845/mcp" }
     }
   }
   ```
4. In omp run `/mcp reload`, then `/mcp test figma-desktop`. If the tools do not show up, start a new session.

Note: Figma's remote server (`mcp.figma.com`) uses OAuth and rejected omp's client registration, so use the desktop server. The Figma desktop app must stay open while you slice.

## 3b. One-command start: the `figma/` folder

Instead of running each mode yourself, drop everything into the `figma/` folder at the project root and let the AI read it:

- variable exports (`Responsive.json`, `Static.json`, any name),
- saved `get_metadata` XML for each page,
- an optional inventory JSON (type styles, letter spacing, layout overrides).

Files are recognised by their **content**, not their name. Then tell the AI, for example:

> Read the figma folder with the `lumos-import-figma` skill. Show me the report, ask me whatever is ambiguous, and only then update `base.css`.

Other phrases that work: "read the figma folder", "baca folder figma", or the name of another folder. If you give a Figma node link instead of files, the AI first saves the `get_metadata` XML into the folder (Figma desktop must be open), then runs the same command.

The command behind it:

```bash
npm run slice -- --folder figma      # or --folder some-other-dir
```

It prints what it found (for example `2 variable exports, 1 metadata, 1 inventory, 2 skipped`), runs every applicable analysis, and ends with one consolidated list of questions and CSS lines to place. Unrecognised files are listed under `SKIPPED`. An empty or missing folder exits with an error. Everything in `figma/` except its README is gitignored, so design data never reaches the template.

## 4. Slicing, step by step

Run everything from the project root. Commands use `npm run slice --`, which runs `node .agents/skills/lumos-import-figma/convert.mjs`. (`convert` is not a shell command; `slice` is the npm script defined in `package.json`.)

### 4.1 Figma variables → tokens

1. Export your Figma variable collections as JSON (for example `Responsive.json` with desktop/tablet/mobile modes, and `Static.json` for colors and fonts). Put them in the project root. They are gitignored on purpose.
2. Run:
   ```bash
   npm run slice -- --variables Responsive.json Static.json
   ```
3. The output is a table per variable: `match`, `DIFFERS` (both values shown), or `MISSING`. The `TO UPDATE BY HAND` and `TO PLACE BY HAND` sections list the CSS lines to put in `base.css`.
4. Modes are identified by name (case-insensitive; the typo `dekstop` is tolerated). Unknown variable groups are reported, not ignored.
5. The `FONTS` section compares Figma's fonts with `fonts:` in `astro.config.mjs` (see section 6).

The skill never writes `base.css`. You (or the agent) place the tokens following the placement table in `SKILL.md`.

### 4.2 Layout: margin, gutter, section padding

Figma variables do not hold these, so measure them from node geometry.

1. Ask the agent to call `get_metadata` on the page node (the Figma section that contains the Desktop, Tablet and Mobile frames) and save the XML, for example `page.xml`.
2. Run:
   ```bash
   npm run slice -- --metadata page.xml
   npm run slice -- --metadata page-a.xml page-b.xml        # several pages at once
   npm run slice -- --metadata page.xml --wrapper 123:456   # when the wrapper is not detected
   ```
3. Result: `site-margin`, section padding top and bottom, and `site-gutter` per breakpoint, with confidence levels and outliers, then a comparison with `base.css`: `MATCH`, `DIFFERS` or `UNMAPPED`.
4. Breakpoints are identified by frame width, not by name. The content container is found by structure (the narrower, inset child), so the name `global-wrapper` is not required.
5. The skill does not decide whether a padding is `small`, `medium` or `large`. You map it. `Section` uses `medium` by default.
6. Hero and CTA sections often have no children in the metadata. Read them through `get_design_context`. Be careful: the fallback numbers in its output (`var(--padding/4_5rem,72px)`) are desktop-mode values. The metadata geometry is the source of truth.

A sample input lives at `.agents/skills/lumos-import-figma/fixtures/sample-page.xml`:
```bash
npm run slice -- --metadata .agents/skills/lumos-import-figma/fixtures/sample-page.xml
```

### 4.3 Typography from text nodes

Letter spacing and text-transform are not in variables; they live on text nodes. Collect them per style and put them in a JSON file:

```json
{
  "type": [{
    "name": "H1",
    "sizePx": { "desktop": 54, "tablet": 45, "mobile": 32 },
    "lineHeightPx": { "desktop": 60, "tablet": 48, "mobile": 36 },
    "letterPx": { "desktop": -1.35, "tablet": -1.125, "mobile": -0.8 }
  }],
  "layout": [
    { "token": "section-space-medium", "px": { "desktop": 80, "tablet": 64, "mobile": 56 } }
  ]
}
```
```bash
npm run slice -- --json design.json
```

Letter spacing is stored as one `em` value per style. If it differs between breakpoints, the skill asks instead of applying it.

### 4.4 Quick lookups

```bash
npm run slice -- --px 30                 # snap to the nearest spacing token (desktop)
npm run slice -- --px 30 --bp mobile
npm run slice -- --lh 36/32              # line height 36 on a 32 font size
npm run slice -- --color "#FFFFFF@60"    # color with alpha → color-mix
```

## 5. Ground rules while slicing

- The skill reports differences and asks. It never silently creates a new token to cover an inconsistency in the design.
- Differences such as Figma's Bold H1 versus the template's Medium are per-project decisions. The template keeps its defaults, and slicing overrides them.
- Values that were not measured (for example tablet when only desktop exists) are marked as guesses in the report.

## 6. Fonts

Figma usually specifies "Inter Display". That is Inter at optical size 32, not a separate family on Google Fonts. The template already loads it:

```js
// astro.config.mjs
fonts: [{
  name: "Inter",
  cssVariable: "--font-inter",
  provider: fontProviders.google(),
  weights: ["400 700"],
  styles: ["normal"],
  options: { experimental: { variableAxis: { opsz: ["32"] } } },
}],
```

- The font is downloaded from Google at dev or build time, so it needs an internet connection.
- For another font, change `name`, `weights` and `variableAxis`. For licensed fonts, use `fontProviders.local()` with a `variants` entry per weight.
- `--variables` tells you when a Figma family or weight is not configured.

## 7. Verifying the result

1. Run `npx astro dev --background` and open the page.
2. Resize the window across 767/768px and 991/992px. Values must jump at those points.
3. Type check: `npx astro check`.
4. Compare against the Figma screenshots at three widths (1440, 834, 393). A mismatch usually means the design is inconsistent (a question to ask), not a token bug.

## 8. Building pages

Follow `LUMOS.md`: compose from components (`Section`, `ContentWrapper`, `Heading`, `Paragraph`) and avoid new classes unless no existing variant fits. Available text variants: `display`, `h1`–`h6`, `large`, `main`, `small`, `xsmall`, `overline-small`, `overline-main`. Icon variants: `small`, `medium`, `large`, `2xs`–`4xl`.

## 9. Updating the skill in older projects

A project made from the template is a copy. Later template changes do not flow in automatically. To bring in the latest skill, copy the `.agents/skills/lumos-import-figma` folder from the template. Its version is `SKILL_VERSION` in `convert.mjs`. For the Lumos framework itself, use the `lumos-upgrade-version` skill.

## 10. Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `/mcp reauth` fails with "OAuth authorization failed" | Figma's remote server rejects omp's client. Use the desktop server (section 3). |
| Figma tools do not appear | Run `/mcp reload` or start a new session. Make sure Figma desktop is open with the MCP server enabled. |
| Headings look Regular instead of Bold | The weight is not loaded. Check `weights` in `astro.config.mjs`. |
| Tablet section padding does not match the variable | The designer bound different variables per frame. Measure from the metadata. |
| `--metadata` reports "not measurable" | The section is an instance or has no children. Read it through `get_design_context`. |
| Button text is taller than in the design | Trim is off, by design, to follow Figma's line box. Turn it on with `.text-trim` if needed. |
