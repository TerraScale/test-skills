# TerraScale agent skills

Agent skills for migrating to TerraScale and using it in your applications. Each
skill is an independent set of instructions installed with Vercel’s
[skills CLI](https://github.com/vercel-labs/skills).

The catalog below is planned; this scaffold does not publish any skills yet.
Install a skill once its status changes to available.

## Install

Requires Node.js and npm. Run from the project where your agent works:

```sh
npx skills add TerraScale/test-skills
npx skills add TerraScale/test-skills --skill supabase-to-terrascale
npx skills add TerraScale/test-skills --list
```

The first command opens skill selection. `--skill` selects one skill; `--list`
shows available skills without installing. An empty catalog reports “No skills
found” until the first skill lands. `npx skills list` lists skills already installed.

Add `--agent codex` or `--agent claude-code` to choose an agent, and `--global`
to install for your user instead of the current project. The CLI supports Codex,
Claude Code, Cursor, GitHub Copilot, Gemini CLI, OpenCode, Windsurf, and
[other agents](https://github.com/vercel-labs/skills/blob/main/docs/agents.md).
See the [CLI reference](https://github.com/vercel-labs/skills/blob/main/docs/commands.md)
for all installation options.

## Skills

Each skill PR updates only its own row from planned to available.

### Supabase

| Skill | Purpose | Status |
| --- | --- | --- |
| supabase-to-terrascale | Entry point: inventory, plan, cutover, and rollback. | planned |
| supabase-database-to-terrascale | Migrate a Supabase database. | planned |
| supabase-auth-to-terrascale | Migrate Supabase authentication. | planned |
| supabase-storage-to-terrascale | Migrate Supabase storage. | planned |
| supabase-edge-functions-to-terrascale | Move Supabase Edge Functions. | planned |
| supabase-client-to-terrascale | Replace Supabase client integrations. | planned |

### Firebase

| Skill | Purpose | Status |
| --- | --- | --- |
| firebase-to-terrascale | Entry point for moving off Firebase. | planned |
| firestore-to-terrascale | Migrate Firestore data and access patterns. | planned |
| firebase-auth-to-terrascale | Migrate Firebase authentication. | planned |

### Other databases

| Skill | Purpose | Status |
| --- | --- | --- |
| postgres-to-terrascale | Migrate Neon, RDS, Cloud SQL, or self-hosted Postgres. | planned |
| turso-to-terrascale | Migrate from Turso. | planned |
| sqlite-d1-to-terrascale | Migrate SQLite files or Cloudflare D1. | planned |
| planetscale-to-terrascale | Migrate PlanetScale or other MySQL databases. | planned |

### Using TerraScale

| Skill | Purpose | Status |
| --- | --- | --- |
| terrascale-getting-started | Start a TerraScale project. | planned |
| terrascale-migrations-and-branches | Work with migrations and branches. | planned |
| terrascale-orm-setup | Configure an ORM for TerraScale. | planned |
| terrascale-sdks | Use TerraScale SDKs. | planned |

## Contributing

Follow [AGENTS.md](AGENTS.md). Skills live at `skills/<name>/SKILL.md`; references
and optional scripts stay inside that skill’s folder. The CLI discovers this
layout without a manifest; see its
[authoring and discovery rules](https://github.com/vercel-labs/skills/blob/main/docs/creating-skills.md).

Run the same checks as CI:

```sh
node scripts/validate-skills.mjs
node scripts/validate-skills.mjs --cli-list
```

The second command runs `npx -y skills add . --list` and checks the discovered
names. Only an empty catalog may return the CLI’s explicit no-skills result.

[MIT licensed](LICENSE).
