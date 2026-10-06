---
name: lumos-import-figma
description: Build a page or fill in Lumos for Astro variables from a Figma file or folder, especially a messy one missing global variables. Use when the user shares a Figma link or design, or says "read the figma folder" / "baca folder figma" / names a folder of Figma exports, and asks to implement it, translate it into Lumos, fill in the design tokens, or when the design's spacing, type and color are inconsistent and need reconciling against src/styles/base.css.
---

# Building Lumos from a Figma file

A design file is a picture of an intention, not a source of truth. The job is
to land the intention in the token system with as few new tokens as possible,
and to be explicit about every guess.

**The rule that outranks the rest: never invent a variable or a class to paper
over an inconsistency in the design. Surface it and ask.** Two paddings that
differ by 6px are usually one padding drawn twice. Ask which it is before
writing anything.

## One-command start

The user drops everything the design gave them into `figma/` — variable
exports, saved `get_metadata` XML, inventory JSON — and says "read the figma
folder", "baca folder figma", or names another folder. Then:

```bash
node .agents/skills/lumos-import-figma/convert.mjs --folder figma
```

(`npm run slice -- --folder figma` is the same.) The folder is the default when
`--folder` has no value. Every file is recognised by what is inside it, not by
its name: a JSON with `modes` and `variables` is a variable export, XML that
starts with a `section`, `frame`, `canvas` or `instance` tag is metadata, and a
JSON whose keys are all inventory keys is an inventory. Anything else, and any
invalid JSON, is listed under `SKIPPED (not recognised)` rather than failing the
run. An empty or missing folder exits non-zero.

The agent shows the consolidated report, puts the single `ASK BEFORE WRITING`
list to the user, and edits `base.css` only after the answers.

If the user gives a Figma node link instead of files, first save its
`get_metadata` XML into the folder through the Figma MCP, then run the same
command. The steps below explain what each part of the report means.

## What Figma cannot say

Three conversions are always needed, because the file physically cannot hold
the values this system uses.

| In Figma | In Lumos | Conversion |
| --- | --- | --- |
| `32px` | `2rem` | ÷ 16, in the token's name only — the values are stored as px |
| line height `36px` on a `32px` size | `--h4-line-height-desktop: 36` | px per breakpoint, snapped to the closest line-height token; never a ratio |
| letter spacing `-2.4px` on an `80px` size | `-0.03em` | letter spacing ÷ font size, or % ÷ 100 |
| `#FFFFFF` at 60% opacity | `color-mix(in lab, var(--color-neutral-0) 60%, transparent)` | alpha becomes the mix percentage |

Line height used to be a unitless ratio. Headings and text now carry a px line
height per breakpoint (`--h1-line-height-mobile` / `-tablet` / `-desktop`), so a
Figma line height is copied across, not divided. The `--line-height-*` ratios
remain only for `--display` and ad-hoc use.

One refinement on the last row. If the faded hex is whatever a theme uses for
`--text`, the answer is `currentcolor`, not that swatch — otherwise the muted
label stays dark when the section flips to the dark theme. The script spots
this and says so.

The opacity one matters most. A designer who wants a muted label has no
`color-mix`, so they restate the base hex at lower opacity. That is not a new
color — it is the existing swatch, mixed. Adding `--grey-400: #999` for it is
the mistake this skill exists to prevent.

## Three breakpoints, three measurements

Responsive tokens hold one px value per breakpoint, picked by the
`--bp-mobile` / `--bp-tablet` / `--bp-desktop` flags in `:root`. There is no
interpolation between them. Map Figma frames, or the modes of a variable
collection, onto them by width:

| Lumos | Viewport | Figma frame or mode |
| --- | --- | --- |
| mobile (default) | `<= 767px` | the mobile frame / `mobile` mode |
| tablet | `768px - 991px` | the tablet frame / `tablet` mode |
| desktop | `>= 992px` | the desktop frame / `desktop` mode |

`--viewport-max` (1440) is only the `max-width` of the content; it is not a
breakpoint and nothing is measured against it.

**Check which frames or modes exist before measuring anything** — files differ,
and the two paths produce different work:

- **All three.** Measure each. Nothing is guessed. Say which frame widths you
  measured, since a 390px frame stands for "mobile" but is not the 767px edge.
- **Only some** (desktop only, say). The missing breakpoints are **guesses**.
  `convert.mjs` derives each one by ratio from the closest existing token: the
  token's value at the missing breakpoint, scaled by how that token's value at
  the nearest measured breakpoint compares to yours. Every derived value goes
  in the report; they are the values most likely to be wrong.

Never mix the two silently. If half the tokens are measured and half derived,
the report has to say which is which.

## Layout slicing

Site margin, gutter, section padding and display size are properties of the
page, not of a component, and they can be read from geometry alone — without
trusting node names.

1. Run `get_metadata` on the page section that holds the three frames (desktop,
   tablet, mobile) and save the XML, one file per page:

   ```bash
   node .agents/skills/lumos-import-figma/convert.mjs --metadata page.xml
   # try it on the synthetic fixture: .agents/skills/lumos-import-figma/fixtures/sample-page.xml
   ```

2. The script tells the frames apart by **width** (`>= 992` desktop, `768-991`
   tablet, `<= 767` mobile). The `[Desktop]` names are only a tie-break; a file
   without exactly one frame per breakpoint is reported, not guessed. Each
   full-width child of a page frame is a section. The content container is the
   first descendant that is narrower than the section and inset
   symmetrically, found by descending through single full-width children, so
   the wrapper's name does not matter. `--wrapper <nodeId,...>` overrides where
   it starts looking. Instances (navbar, footer) and sections with no children
   are listed as not measurable from metadata.

3. Per breakpoint it measures **site-margin** (container x), **section
   padding** (container y above, and section height minus container bottom
   below), and **site-gutter** (the modal gap between adjacent equal-sized
   siblings in a row that spans at least half the container, falling back to
   stacked siblings, marked low confidence). It prints the modal value, every
   distinct value with its count, and each outlier with its node id, so one
   stray 40 among thirty-twos is visible rather than averaged away. It compares
   the result with `base.css` and prints a ready `layout` snippet for `--json`.

4. It never decides which `section-space-*` token a padding group belongs to.
   Each group is listed with its sections and an `ASK`. **If the padding groups
   differ, ask whether they are variants (small/large, say) before mapping any
   of them.** `display` is reported only when a text layer is named `display`;
   otherwise the script says it is not used in these frames. It also lists
   which type styles the page uses (layers named `h1`–`h6`, `p__lg|md|sm|xs`,
   `overline__md|sm`) with their widths — metadata carries no sizes.

5. Use `get_design_context` only for variable bindings and text styles. Its
   fallback numbers, such as `var(--padding/4_5rem,72px)`, are **desktop-mode**
   values: the tablet frame really renders 64. The designers here bind a
   different variable per frame (`padding/5rem` on desktop, `padding/4_5rem` on
   tablet and mobile) instead of relying on modes. Measured geometry from
   metadata is the source of truth; the fallback number is not.

Differences between Figma's text styles and the template's defaults — Figma H1
Bold 700 against the project's Medium, letter spacing -2.5% against
`-0.03em` — are **per-project slicing results**. The template keeps its
defaults. The skill reports each difference and asks; it never applies one on
its own.

The Figma MCP guidance skill `figma-design-to-code` is not available in omp and
is not needed here: slicing only uses the data the tools return.

## Steps

1. **Read the file.** Prefer a variable export when there is one: the
   Responsive and Static collections written out as JSON (see step 3). Otherwise
   use the Figma MCP tools — `get_variable_defs` for whatever variables do
   exist, `get_design_context` for each frame's text styles, `get_screenshot` to
   see what it should look like. Start with the variables: they tell you how
   much of the system the designer actually used.

2. **Inventory before converting.** List every distinct spacing value, type
   size with its line height, and color with its opacity, at each breakpoint you
   have. Distinct *values*, not distinct layers — the same 24px appearing eleven
   times is one value.

   **Text styles come from `get_design_context`.** For each frame (desktop,
   tablet, mobile) of the page, call `get_design_context` with
   `excludeScreenshot: true` and save the raw output into `figma/`, for example
   `about-desktop.txt`, then run `--folder`. Its "These styles are contained in
   the design" line is the only place that says which text style uses which
   weight and tracking: variables hold sizes and line heights, not the weight or
   letter spacing a style applies. The script reads the weight, the letter
   spacing (a percent: -2.5 means -0.025em) and any `uppercase` from there, and
   compares them per style with `base.css`. Line height comes from the variable
   export per breakpoint; the capture only says which variable a style uses. The
   px fallbacks in the code, such as `var(--line-height\/heading\/h1,60px)`, are
   desktop-mode values even in the tablet and mobile frames, so they are never
   used as measurements. **Do not apply or skip these three attributes
   silently:** report each difference, ask, then edit. A style used with two
   weights is a conflict to ask about, not to guess.

   Leading trim is off by default in this project (opt-in with a `.text-trim`
   class; the `--*-trim-top/bottom` tokens still exist). A Figma text box
   includes the full line height, so measure spacing to the line box, not to the
   glyphs.

3. **Convert and match.** Two inputs, in order of preference.

   **A Figma variable export** is the best input, because the designer already
   named and measured every breakpoint. Pass the exported files:

   ```bash
   node .agents/skills/lumos-import-figma/convert.mjs --variables Responsive.json Static.json
   ```

   Each file has the shape `{ name, modes: { id: name }, variables: [{ name, type, valuesByMode }] }`.
   Mode ids differ between files, so modes are identified by name,
   case-insensitively (`desktop`, `tablet`, `mobile`; the typo `dekstop` is
   accepted). A collection with no breakpoint modes is treated as static. The
   script names each variable's Lumos token, then prints a table saying whether
   the three values already match `base.css`, differ (both shown), or are
   missing from it:

   | Figma variable | Lumos token |
   | --- | --- |
   | `font-size/heading/h1` | `--h1` (line height: `line-height/heading/H1` → `--h1-line-height`) |
   | `font-size/body/Lg` `Md` `Sm` `Xs` | `--text-large` `--text-main` `--text-small` `--text-xsmall` |
   | `font-size/overline/Sm` `Md` | `--overline-small` `--overline-main` |
   | `padding/…` and `spacing/…`, merged | `--space-<N-M>rem`, e.g. `padding/3_5rem` → `--space-3-5rem`, `None` → `--space-none` |
   | `corner-radius/…` | `--radius-*`, e.g. `1_25rem` → `--radius-1-25rem`, `Full` → `--radius-full` |
   | `icon-size/…` | `--icon-*`, e.g. `3XL` → `--icon-3xl`, `M` → `--icon-m` |
   | `color/<group>/<name>/<step>` | `--color-<group>-<name>-<step>`; a trailing `[base]` is dropped |
   | `font/weight/…` | `--primary-regular` / `-medium` / `-bold`; others reported missing |
   | `font/family/…` | compared with `fonts:` in `astro.config.mjs` and `--primary-family` |

   A variable export does not hold letter spacing, font weight per style or
   text-transform. The script says so on every `--variables` run; they come
   from `get_design_context` (see step 2).

   **Fonts.** The script also lists which Figma families are configured under
   `fonts:` in `astro.config.mjs` (`--astro-config <file>` to point elsewhere),
   which one `--primary-family` resolves to, and which `font/weight/*` values
   the entry does not cover. Coverage comes from the entry's `weights` array
   (numbers, `"400"`, names like `"bold"`, or a range such as `"400 700"` that
   covers every weight inside it), or from `variants[].weight` for a local
   provider. A weight outside that shows as `MISSING`. The provider (`google`,
   `local`) is printed. It never touches the network.

   "Inter Display" is the Inter family at optical size 32 (the `opsz` axis), not
   a separate Google family. The known-good setup is a single Inter entry on
   `fontProviders.google()` with the weights the design uses and the axis pinned:

   ```js
   {
     name: "Inter",
     cssVariable: "--font-inter",
     provider: fontProviders.google(),
     weights: ["400 700"],
     styles: ["normal"],
     options: { experimental: { variableAxis: { opsz: ["32"] } } },
   }
   ```

   The script reads "Inter Display" as configured only when the entry is named
   Inter and `variableAxis.opsz` is exactly `["32"]` (it prints "Inter Display =
   Inter pinned at opsz 32"); plain Inter without that pin stays `NOT
   CONFIGURED`. Nothing else gets this treatment. For any other missing family
   or weight the fix is either widening `weights` on a Google entry where the
   family exists on Google Fonts, or a local file under `src/assets/fonts` as a
   `variants` entry. This is an **ask the user** item: the skill reports it and
   does not decide.

   Padding and spacing are one scale in this system. If both exist for a step
   and disagree, the script says `CONFLICT` and asks which is right. Any
   variable in a group it does not know is printed under `UNKNOWN VARIABLE
   GROUPS` rather than dropped, so ask what it is.

   **A hand-made inventory** is for when there is no export, only frames. Write
   it to JSON and run the script:

   ```bash
   node .agents/skills/lumos-import-figma/convert.mjs --json design.json
   ```

   The script does the arithmetic because there is a lot of it and it is easy
   to get quietly wrong: forty values, each measured against a scale of
   three-value tokens, plus RGB distance for every colour. It reads the tokens
   out of `base.css` rather than carrying a copy, so it cannot drift from the
   system.

   ```json
   {
     "space":  [{ "name": "stack gap", "px": 30 },
                { "name": "card pad", "px": { "desktop": 64, "tablet": 56, "mobile": 48 } }],
     "type":   [{ "name": "Section title",
                  "sizePx": { "desktop": 54, "tablet": 45, "mobile": 32 },
                  "lineHeightPx": { "desktop": 60, "tablet": 48, "mobile": 36 },
                  "letterPx": { "desktop": -1.62, "tablet": -1.35, "mobile": -0.96 } }],
     "letter": [{ "name": "Hero tracking", "px": -2.4, "sizePx": 80 }],
     "radius": [{ "name": "Card corner", "px": { "desktop": 16, "tablet": 12, "mobile": 8 } }],
     "icon":   [{ "name": "Nav icon", "px": 24 }],
     "layout": [{ "token": "section-space-medium", "px": { "desktop": 80, "tablet": 64, "mobile": 56 } }],
     "weight": [{ "name": "Heading", "value": "Medium" }],
     "color":  [{ "name": "Muted label", "hex": "#FFFFFF", "alpha": 0.6 }]
   }
   ```

   `space`, `radius`, `icon` and `layout` take `px`, and `type` takes `sizePx` and
   `lineHeightPx`, as either a single number (a desktop measurement only) or an
   object with any of `desktop`, `tablet`, `mobile`. A breakpoint left out is a
   guess, derived as described above. A `layout` entry names its token
   (`site-margin`, `site-gutter`, `display` or any `section-space-*`) and is
   compared with `base.css`: `match`, or `DIFFERS` with the three value lines
   under `TO UPDATE BY HAND`, or `UNMAPPED` with a four-line block to place.

   Add `"on": "#1F1D1E"` and `"sizePx"` to a colour and the script also reports
   its WCAG contrast, using the large-text bar of 3:1 at 24px and above. These
   are flagged, never blocking — a decorative label may fail deliberately — but
   an unreadable body colour is usually the design being messy rather than a
   decision, so raise it with the other questions.

   Without a design-context capture, `type` entries also take `letterPx` or
   `letterPct`, as a single number or a
   per-breakpoint object like `sizePx`. Pixels are divided by `sizePx` at the
   same breakpoint, percentages by 100, giving em. Letter spacing is **one value
   per style** in `base.css` (`--h1-letter-spacing: var(--letter-spacing-tight)`),
   so if the breakpoints disagree by more than 0.002em the script asks instead
   of choosing. Otherwise it compares against the style's current value and
   prints either a match, or the exact line to change — pointing at an existing
   `--letter-spacing-*` token when one is within 0.002em, and a new
   `--letter-spacing-<name>: <em>;` under `TO PLACE BY HAND` when none is.
   The top-level `letter` list still works for a value that belongs to no style.
   Add `"token": "name"` to any entry to choose what a new variable would be
   called. Unknown keys are rejected rather than silently ignored, so a typo
   does not read as "nothing to convert".

   The script snaps anything within 2px at every measured breakpoint to the
   token it is drifting from, derives the breakpoints that were not measured,
   and prints an `ASK BEFORE WRITING` list. Values off by more than 2px are
   decisions, not drift, and belong in that list. A type size that matches a
   token keeps that token's own line height; if yours differs, that is a
   question about the token, not a new one.

   For one-off lookups: `--px 30` snaps against the desktop column of the
   scale, and `--bp tablet` or `--bp mobile` switches column. `--lh 36` does the
   same against the line-height tokens, and `--lh 36/32` also shows the ratio
   against the `--line-height-*` values. `--color "#FFFFFF@60"` restates an
   opacity as a mix.

4. **Ask the questions.** Put the whole `ASK BEFORE WRITING` list to the user
   at once, each with the option to consolidate:

   > The design uses 30px, 32px and 34px gaps in three places. `--space-2rem` is
   > 24, 28 and 32px across the breakpoints. Consolidate all three, or is one of
   > them deliberate?

   Wait for answers. Do not write tokens for anything still in question.

5. **Place the tokens yourself.** The script prints what to add under
   `TO PLACE BY HAND`, and the tokens whose values differ under `TO UPDATE BY
   HAND`; it does not touch `base.css`. Where a token goes says what it means,
   and `:root` is ordered by kind — put each one with its own:

   | Kind | Goes beside |
   | --- | --- |
   | spacing | the `--space-*` scale (`--space-none` … `--space-7-5rem`), in order by rem size — spacing and padding are one scale |
   | section spacing | `--section-space-large` |
   | type size and line height | the `h1`–`h6` / `text-*` / `overline-*` block, in size order — add **both** the font-size triple and the line-height triple |
   | letter spacing | beside `--letter-spacing-tight` / `-normal` |
   | radius | the `--radius-*` group, in order by size |
   | icon | the `--icon-*` group, in order by size |
   | font weight | the `--primary-*` weights |
   | swatch | the `--color-*` list in the Swatches section, grouped by palette family and ordered by step |
   | themed color | **every** theme block — `:root`/`.theme-light`, `.theme-dark`, `.theme-brand` — or it breaks on one theme |

   A responsive token is four lines: the `calc()` that picks a value by
   breakpoint, then `-mobile`, `-tablet` and `-desktop` as unitless px numbers.
   The script prints exactly that shape, so it matches the existing tokens:

   ```css
   --space-4rem: calc((var(--bp-mobile) * var(--space-4rem-mobile) + var(--bp-tablet) * var(--space-4rem-tablet) + var(--bp-desktop) * var(--space-4rem-desktop)) / 16 * 1rem);
   --space-4rem-mobile: 48;
   --space-4rem-tablet: 56;
   --space-4rem-desktop: 64;
   ```

   A line height is the same shape, named `--<type>-line-height`. Keep the scale
   in order: a `--space-8rem` of 128px belongs after `--space-7-5rem`, not
   wherever it was measured.

   Colors are two layers. The `--color-<group>-<step>` primitives are the Figma
   palette; the theme blocks hold the semantic names (`--background`, `--text`,
   `--brand`, the button and link variables), which point at primitives. A new
   swatch goes in the primitive list. A themed color goes in every theme block
   as a `var(--color-…)`, never a raw hex.

6. **Fill the gaps the design forgot.** A messy file will be missing states
   nobody drew: hover and focus colors, the dark-theme counterpart of a button,
   disabled text. Derive them from what the file does show, following the
   existing pattern in `base.css` — each theme block defines the same set of
   `--button-*` variables, so a missing dark-theme hover has an obvious shape
   to fill. **Every one of these is a guess and goes in the report.**

7. **Build with what exists, then build what doesn't.** Compose from the
   library first — `Wrapper/Section` and `Wrapper/ContentWrapper` for layout,
   `Wrapper/Grid` for columns, `Item/Card` for repeated blocks,
   `Typography/*` for text. A design that "needs" a new class usually needs an
   existing variant, and a one-off class is how a system stops being one.

   When something genuinely does not compose — a testimonial slider, a stats
   row — build it, following the new component checklist in `LUMOS.md`. List
   every component you added in the report, with a sentence on why nothing
   existing covered it. That list is the one most worth arguing with: it is
   where the system grows, and growth is harder to undo than a token.

8. **Look at it.** Tokens matching the table does not mean the page matches the
   design. Start the dev server, open the page, and compare it against the
   screenshot from step 1:

   ```bash
   astro dev --background
   ```

   Screenshot the built page at the same width as each frame you measured, and
   check the two side by side. Then check the breakpoints you did *not*
   measure — a design given only at desktop still has to survive 390px and
   800px, and that is where derived values show up as wrong. Test the
   boundaries too, because the switch is instant: 767px must show mobile values
   and 768px tablet, 991px tablet and 992px desktop. Report what does not match
   rather than quietly adjusting tokens until it does: a mismatch is often the
   design being inconsistent, which is a question, not a bug.

## The report

Close with these lists. Anything empty, say so.

- **New variables** — name, value at each breakpoint, and what in the design
  asked for it.
- **New components** — what was built, and why nothing existing covered it.
- **Guesses** — derived breakpoint values, invented states (dark-theme button
  hover, focus rings), anything the file did not actually specify.
- **Snapped** — values moved to an existing token, with the delta. These were
  applied without asking; the user may still want to reverse one.
- **Contrast** — any pair below its WCAG floor, with the ratio. Flagged, not
  fixed.
- **Weights** — for every text style, the Figma weight next to the project
  weight (the ones that match too), and any style used with two weights.
- **Letter spacing** — each text style whose letter spacing was changed or
  newly added, with the em value and the Figma value it came from.
- **Fonts** — whether the Figma family and every weight it uses are configured
  under `fonts:` in `astro.config.mjs`, and what is missing.
- **Still open** — inconsistencies the user has not ruled on yet.

## Checklist before editing base.css

- [ ] Variables compared: every row `match`, `DIFFERS` or `MISSING`, nothing left in an unknown group.
- [ ] Layout measured: site-margin, gutter and section padding from metadata, outliers looked at.
- [ ] Text styles compared: weight, letter spacing and line height for every style from `get_design_context`, conflicts asked.
- [ ] Fonts checked: the family and every weight are configured.
- [ ] Colors compared: swatches matched, new and themed ones listed.
- [ ] The consolidated `ASK BEFORE WRITING` list was put to the user and answered.

## Versions

This skill versions separately from the framework. A fix here does not need a
Lumos release, and a Lumos release does not invalidate the skill.

- **Skill version** — `SKILL_VERSION` in `convert.mjs`. Bump it when the
  conversion rules or the workflow change.
- **Lumos version** — `package.json` is the source of truth. Nothing here
  duplicates it; the script reads it and prints both on every run:

  ```
  lumos-import-figma 2.0.0  ·  Lumos <whatever package.json says>
  ```

  If the running project is a different version than `TESTED_AGAINST`, the
  script says so. That is a prompt to check `base.css` still looks the way this
  skill assumes — token names, the four-line responsive token shape, the theme
  blocks — not a reason to stop.

## Using this without Claude Code

Nothing here is Claude-specific except the loading. The workflow is this file
and the script is plain Node, so another assistant can be pointed at
`.agents/skills/lumos-import-figma/SKILL.md` and follow it, and anyone can run
`node .agents/skills/lumos-import-figma/convert.mjs` by hand. Only the automatic
triggering and `/lumos-import-figma` are Claude Code features.
