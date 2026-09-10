# Skills installed in this repo

Claude Code loads every directory under `.claude/skills/` automatically when a
session starts in this repo. Invoke one with `/<name>` or just describe the task
and let Claude pick.

## Design skills — from [taste-skill](https://github.com/Leonxlnx/taste-skill) (MIT, © leonxlnx)

Installed with `npx skills add https://github.com/Leonxlnx/taste-skill --all`,
then flattened: the CLI's `.agents/` and `agent/` mirror trees were removed and
the symlinks under `.claude/skills/` replaced with real files, so the install
works on any OS and in any checkout. `skills-lock.json` in the repo root records
the upstream source and content hash of each skill.

### Skills that write code

| Skill | What it does |
| --- | --- |
| `design-taste-frontend` | The main taste skill (v2). Reads the brief, infers a design direction, tunes variance / motion / density. Landing pages, portfolios, redesigns. |
| `design-taste-frontend-v1` | The original v1, kept for exact backward compatibility. |
| `gpt-taste` | Stricter GPT/Codex-oriented variant: higher layout variance, heavier GSAP direction. |
| `image-to-code` | Image-first pipeline — generate design references, analyze them, then build the frontend to match. |
| `redesign-existing-projects` | Audits an existing UI first, then fixes layout, spacing, hierarchy, styling without breaking it. |
| `high-end-visual-design` | Agency-tier fonts, spacing, shadows, cards, motion. Blocks the defaults that make AI output look cheap. |
| `minimalist-ui` | Clean editorial style: warm monochrome, typographic contrast, flat bento grids. |
| `industrial-brutalist-ui` | Swiss print meets military terminal — rigid grids, extreme type scale, analog degradation. |
| `stitch-design-taste` | Generates `DESIGN.md` files for Google Stitch. |
| `full-output-enforcement` | Bans truncation and placeholder comments; forces complete files. |

### Skills that only produce images (no code)

| Skill | What it does |
| --- | --- |
| `imagegen-frontend-web` | Website comps — one horizontal image per section. |
| `imagegen-frontend-mobile` | Mobile screens and flows in phone mockups. |
| `brandkit` | Brand boards: logo directions, palettes, type, identity applications. |

### Updating

```bash
npx skills add https://github.com/Leonxlnx/taste-skill --all
```

That recreates the CLI's `.agents/` + symlink layout. Re-flatten it afterwards
(remove `.agents/` and `agent/`, copy the real files into `.claude/skills/`) to
keep this repo's structure.

## Other skills

| Skill | What it does |
| --- | --- |
| `prompt-master` | Prompt engineering patterns and templates. |

## Not installed here (already built in)

Browser testing and screenshots do not need a skill — Chromium and Playwright
1.56 are preinstalled in the Claude Code web environment. Ask for a screenshot
or a browser test and Claude drives it directly.
