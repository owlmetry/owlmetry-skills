# Owlmetry Skills

[Claude Code](https://claude.com/claude-code) plugin marketplace for [Owlmetry](https://owlmetry.com) — the self-hosted analytics platform for mobile and backend apps.

## Install

In Claude Code:

```text
/plugin marketplace add owlmetry/owlmetry-skills
/plugin install owlmetry@owlmetry-skills
```

That's it. Claude Code clones this repo, registers the skills, and auto-updates them when `/plugin marketplace update` runs.

## What you get

The `owlmetry` plugin installs three skills:

| Skill | Use when |
|-------|----------|
| `owlmetry-cli` | Signing up, creating projects and apps, defining metrics and funnels, querying events, triaging issues and feedback from the command line. |
| `owlmetry-swift` | Instrumenting an iOS, iPadOS, or macOS app with the [Owlmetry Swift SDK](https://github.com/owlmetry/owlmetry-swift). |
| `owlmetry-node` | Instrumenting a Node.js backend (Express, Fastify, serverless) with the [`@owlmetry/node`](https://www.npmjs.com/package/@owlmetry/node) SDK. |

The CLI skill is the entry point — it handles auth, project/app creation, and knows how to hand off to the SDK skills once an app exists.

## Pinning versions

Plugin versions are resolved from the commit SHA of `main` by default, so every push is a new version. To pin to a specific release, pass `@<ref>` when adding the marketplace:

```text
/plugin marketplace add owlmetry/owlmetry-skills@v1.0.0
```

## Contributing

Source of truth for each skill lives in `plugins/owlmetry/skills/<skill>/SKILL.md`. Edits land directly on `main`; CI lints the marketplace manifest and each skill's frontmatter on every push.

## License

MIT — see [LICENSE](LICENSE).
