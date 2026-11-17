# Authoring skills

## Layout and format

- One skill per folder: `skills/<name>/SKILL.md`. Each skill stands alone: the CLI
  installs skills one at a time. Never link to another skill’s files; naming
  another skill as “see also” is fine.
- Follow the [skills CLI format and discovery rules](https://github.com/vercel-labs/skills/blob/main/docs/creating-skills.md).
  Use YAML frontmatter containing only `name` and `description` string fields.
  Use plain, single-quoted, or JSON-compatible double-quoted scalars, or a block
  scalar (`|`, `>`, optionally with `-` or `+`) indented by two spaces.
- `name` equals the folder name, uses lowercase letters and digits separated by
  single hyphens, and is at most 64 characters. `description` is nonempty and at
  most 1024 characters. Descriptions start with what the skill does and say when
  to use it, with words a user would type, such as “migrate from Supabase” or
  “move off Firebase”.
- SKILL.md is what an agent reads first: when to use it, prerequisites, then
  numbered steps (inventory, plan, migrate, verify, cut over, roll back) with
  checklists. Keep it at most 500 lines with a nonempty body.
- Put long material, such as type maps and code samples per language, in
  `references/*.md` inside the same folder. Optional scripts live in `scripts/`
  and must be runnable as written. Do not add symlinks.
- Every relative link in the skill’s Markdown must resolve to a file inside that
  same skill folder, including links from reference documents. Use inline or
  reference-style Markdown links; external documentation uses full HTTPS URLs.

## Accuracy and secrets

- Every TerraScale fact comes from the public docs at https://docs.terrascale.tech
  and cites the docs page. On our servers inspect the source with
  `git -C ~/Projects/TerraScale/terrascale show origin/main:docs-site/docs/...`.
  Never invent a TerraScale API, CLI flag, SDK method, or limit. When TerraScale
  lacks something the source platform has, say so plainly and give the workaround.
- Source-platform commands (`supabase`, `pg_dump`, `firebase`, `gcloud`, `turso`,
  `pscale`, `wrangler`) must be real and cite their docs.
- Never print or embed secret values: environment variable names only.
- Describe TerraScale’s SQL dialect only in the words the public docs use. Never
  state or imply what engine TerraBase is built on.

## Change scope and checks

- Each skill PR owns its folder and updates only its own README skills-table row
  to available. Do not edit another skill’s files or status.
- Run `node scripts/validate-skills.mjs` and
  `node scripts/validate-skills.mjs --cli-list` before shipping. The latter runs
  `npx -y skills add . --list` and verifies every skill is discovered. The explicit
  no-skills result is accepted only while the catalog is empty.
- Keep validation dependency-free. Add no tests beyond the CI validation.
- New code, scripts, and workflows contain no comments. Do not commit secrets,
  dependency directories, generated logs, or screenshots.
