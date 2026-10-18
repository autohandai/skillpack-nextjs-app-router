# Next.js App Router skills

Four skills for Next.js 16 App Router work: routes and layouts, Server Actions that validate input, caching that fits the model your app uses, and tests that run.
Each one is a plain SKILL.md file. Your coding agent reads it when the task matches, so you don't have to explain the conventions again every time.

## Install

From the root of your Next.js project:

```bash
git clone https://github.com/autohandai/skillpack-nextjs-app-router
mkdir -p .claude/skills
cp -R skillpack-nextjs-app-router/skills/* .claude/skills/
```

Copy the skill folders themselves. Some agents only look one level deep in `.claude/skills/`, so `.claude/skills/nextjs-testing/SKILL.md` gets loaded and a nested pack folder doesn't. To use them in every project, copy them into `~/.claude/skills/` instead.

Most agents pick a skill up when your request matches its description. To call one by name, mention it: `$nextjs-server-actions` in Autohand Code, or `/nextjs-server-actions` in Claude Code.

## Skills

| Skill | What it does |
|---|---|
| `nextjs-routes-layouts` | Pages, layouts, dynamic segments, route groups, loading and error files, typed `params` |
| `nextjs-server-actions` | Actions that check auth, validate with Zod, revalidate and redirect in the right order |
| `nextjs-data-caching` | Works out whether you're on Cache Components or the earlier model, then applies the right APIs |
| `nextjs-testing` | Vitest for components and action logic, Playwright for async Server Components and full flows |

## Adapting it

They're Markdown, so edit them. The usual changes:
- Swap Zod for Valibot, or `@/lib/auth` for your auth helper, in `nextjs-server-actions`.
- Delete the caching section you don't use from `nextjs-data-caching`.
- Add your own folder rules (where components live, naming) to `nextjs-routes-layouts`.

Keep the `name` in the frontmatter the same as the folder name, in lowercase letters, numbers and hyphens.

Written against Next.js 16.4, React 19, Zod 4, Vitest and Playwright as of October 2026. If something's out of date, please open an issue.

Runs in Autohand Code (free CLI, any model): https://autohand.ai/cli/?utm_source=skillpack-nextjs-app-router&utm_medium=github

Until 31 October 2026: first month of Pro for $10 with code FOUNDING10 (first-time customers).

MIT licensed. See LICENSE.
