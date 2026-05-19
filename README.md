# Skills

Model-loadable skills for code-assistant harnesses working on Preact and the `@preact/signals` ecosystem. Each skill is a directory with a `SKILL.md` (Anthropic-format frontmatter + markdown body) and an `agents/openai.yaml` manifest for OpenAI Apps SDK consumers.

## Index

### Preact — setup and configuration

| Skill | When it loads |
|---|---|
| [preact-no-build-vite-setup](preact-no-build-vite-setup/SKILL.md) | Starting a project: Vite, no-build, or integrating into an existing pipeline |
| [preact-typescript-jsx](preact-typescript-jsx/SKILL.md) | TypeScript + JSX transform, `jsxImportSource`, namespace augmentation |
| [preact-compat-aliasing](preact-compat-aliasing/SKILL.md) | Using React libraries via `preact/compat` — bundler, SSR, Jest aliases |

### Preact — runtime debugging

| Skill | When it loads |
|---|---|
| [preact-hooks-debugging](preact-hooks-debugging/SKILL.md) | Invalid hook calls, `__H` errors, duplicate Preact copies |
| [preact-debug-warnings](preact-debug-warnings/SKILL.md) | Interpreting `preact/debug` output |

### Preact — forms and UI

| Skill | When it loads |
|---|---|
| [preact-forms-events](preact-forms-events/SKILL.md) | Controlled vs. uncontrolled inputs, `onInput` vs. `onChange`, form hydration |

### Preact — SSR and hydration

| Skill | When it loads |
|---|---|
| [preact-ssr-prerendering](preact-ssr-prerendering/SKILL.md) | `preact-render-to-string`, streaming, `preact-iso` prerender |
| [preact-hydration-mismatches](preact-hydration-mismatches/SKILL.md) | Diagnosing SSR/client DOM divergence |

### Signals

| Skill | When it loads |
|---|---|
| [preact-signals-core](preact-signals-core/SKILL.md) | `@preact/signals-core`: signal, computed, effect, batch, untracked |
| [preact-signals-preact-integration](preact-signals-preact-integration/SKILL.md) | `@preact/signals` in Preact: `useSignal`, `useComputed`, JSX rendering, `Show`/`For` |
| [preact-signals-react-integration](preact-signals-react-integration/SKILL.md) | `@preact/signals-react` with React 18/19 |
| [preact-signals-models-utils](preact-signals-models-utils/SKILL.md) | Signal-based model patterns and utilities |
| [preact-signals-debugging](preact-signals-debugging/SKILL.md) | Diagnosing stale values, missing updates, loops |
| [preact-signals-testing](preact-signals-testing/SKILL.md) | Testing signal-driven code |
| [preact-signals-no-build](preact-signals-no-build/SKILL.md) | Using signals in no-build / CDN setups |
| [preact-signals-eslint-plugin](preact-signals-eslint-plugin/SKILL.md) | `@preact/signals` ESLint rules |

### Extending Preact

| Skill | When it loads |
|---|---|
| [preact-options-hooks](preact-options-hooks/SKILL.md) | Building plugins/devtools via `options` hooks, internal VNode access |

### Release engineering

| Skill | When it loads |
|---|---|
| [npm-trusted-publishing](npm-trusted-publishing/SKILL.md) | npm trusted publishing, OIDC, GitHub environments, pinned actions, and publish-path cache hardening |

### General

| Skill | When it loads |
|---|---|
| [review](review/SKILL.md) | Reviewing code changes, features, and pull requests |

## Installing into your harness

The `SKILL.md` files use Anthropic's standard skill frontmatter and work in any harness that reads that format. For harness-specific wiring:

- **Claude Code** — copy or symlink directories into `~/.claude/skills/` (user-level) or `.claude/skills/` (project-level). Claude Code auto-triggers on the `description` field. See [Claude Code skills docs](https://docs.claude.com/en/docs/claude-code/skills).
- **OpenAI Apps SDK / ChatGPT** — the `agents/openai.yaml` manifest in each skill declares the display name and default prompt. Point your Apps SDK project at this directory.
- **Cursor** — copy each `SKILL.md` body into `.cursor/rules/<name>.mdc` and prepend the Cursor frontmatter (`description`, `globs`, `alwaysApply: false`).
- **Windsurf / Continue / Aider** — consume the `SKILL.md` body as a rule or system-prompt fragment. The YAML `description` is the trigger text; the markdown body is the guidance.
- **Raw system prompts** — paste the relevant `SKILL.md` body into your system prompt when the user's task matches the `description`.

## Design principles

- **Opinionated.** Each skill leads with the recommended path and reserves nuance for the pitfalls section.
- **Consumer-first.** For the app-developer-facing skills (setup, forms, hooks, hydration, etc.), core-maintainer topics are out of scope. The `preact-options-hooks` skill is the exception — it's scoped to plugin and devtools authors.
- **Doc-anchored.** Skills link to [preactjs.com](https://preactjs.com) and package READMEs rather than repo-internal source files.
- **Version-tolerant.** Examples use unpinned majors (`preact@10`) so they age gracefully.

## Contributing a new skill

1. Pick a narrow, high-signal topic (one failure mode, one setup decision, one API cluster).
2. Write `SKILL.md` with frontmatter: `name` (kebab-case, matches directory), `description` (keyword-rich, triggers the skill). Keep the body under ~4KB.
3. Include at least one concrete code snippet when prose would otherwise re-explain syntax.
4. Link to doc URLs and package READMEs, not repo paths.
5. Add the OpenAI manifest at `agents/openai.yaml` mirroring the existing format.
6. Add a row to the index above.
