# astro-lumos-custom

A template for turning Figma designs into Astro pages that match the design
1:1. It is [Lumos for Astro](https://lumosframework.com) with two changes:

- **Fixed values per breakpoint.** Tokens in `src/styles/base.css` are explicit
  px values for mobile (up to 767px), tablet (768–991px) and desktop (992px and
  up), taken from Figma, instead of fluid `clamp()` scaling.
- **A Figma slicing skill, `lumos-import-figma`.** It reads a Figma file,
  compares it with the project's tokens, asks about everything that is
  ambiguous, and measures the built page against the design.

The tool is meant for developers building a project from a design. An AI coding
agent does the measuring; you provide the design data and answer its questions.

> The palette and layout values in `base.css` belong to the design this template
> was first tested on. The skill replaces them for a new project.

## What is in it

| Path | What it is |
| --- | --- |
| `.agents/skills/lumos-import-figma/` | The skill: `SKILL.md` (the workflow), `convert.mjs` (reads the Figma data), `verify.mjs` (measures the page), `fixtures/` |
| `figma/` | Drop folder for variable exports, `get_metadata` XML and `get_design_context` captures. Git-ignored except its README |
| `scripts/push-both.sh` | Maintainer script for pushing the template to two remotes |
| `TUTORIAL.md` | Step-by-step guide: setup, a new design, updating a project |
| `LUMOS.md` | How to build pages and components in this project |
| `AGENTS.md` | The instruction file agents read first |

The other `lumos-*` skills (`audit-props`, `import-webflow`, `scaling-graphic`,
`upgrade-version`) come from Lumos for Astro.

## Quick start

```sh
git clone https://github.com/<account>/astro-lumos-custom.git my-project
cd my-project
npm install
npm run dev
```

Then start a fresh git history for the project and create its own repository
(see **Create your project** in [TUTORIAL.md](TUTORIAL.md)), so it does not push
back to the template.

Node 22.12 or newer is required.

## Using the tool on a design

1. Export the Figma variables to `figma/` (`Responsive.json`, `Static.json`).
2. Open the file in Figma desktop with Dev Mode and its MCP server on, and give
   your agent a node link.
3. The agent saves the node data into `figma/` and runs
   `npm run slice -- --folder figma`. Answer its list of questions; nothing in
   `src/` changes before that.
4. After building, run `npm run verify -- --url http://localhost:4321/ --page <name>`.
   It measures every section and text box against the Figma metadata and, with
   Figma desktop open, writes a side-by-side diff image per section.

Full steps are in [TUTORIAL.md](TUTORIAL.md); the rules the agent follows are in
[`SKILL.md`](.agents/skills/lumos-import-figma/SKILL.md).

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | Starts the dev server |
| `npm run build` | Builds the site to `dist/` |
| `npm run preview` | Serves the built site |
| `npm run check` | Type-checks every `.astro` file |
| `npm run format` | Formats the project with Prettier |
| `npm run slice` | Reads Figma data and reports tokens, text styles, buttons, assets and questions |
| `npm run verify` | Measures a built page against Figma and writes section diffs |

## Improving the tool

Fix the tool here, never inside a project. When a project shows something the
tool missed or got wrong, the agent logs it in the project's `.feedback/` folder
(git-ignored), using `.agents/skills/lumos-import-figma/feedback-template.md`.
Those entries are read here, sorted into tool problems and design decisions, and
the tool problems are fixed and tested before the project pulls the new version.

## Updating a project

Replace the skill folder as a whole so deleted or renamed files do not linger.
The commands are in [TUTORIAL.md](TUTORIAL.md#updating-an-older-project).

## License and credit

[MIT](LICENSE). Lumos for Astro is by Timothy Ricks; this template builds on it
and keeps its license.
