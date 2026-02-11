# Pimcore Skills for Claude Code

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for developing on the Pimcore platform and its ecosystem.

These skills give Claude deep context about Pimcore's architecture, patterns, and APIs so it can assist you more effectively when building bundles, data objects, Studio v2 plugins, CoreShop extensions, and more.

## Skills

| Skill | Description |
|-------|-------------|
| **pimcore** | Pimcore platform development — bundles, data objects, class definitions, CoreExtensions, events, workflows, documents, assets |
| **pimcore-studio** | Build Pimcore Studio v2 features — React/TypeScript, Ant Design, plugins, modules, dynamic types, DI container, widgets |
| **pimcore-studio-migrate** | Migrate any Pimcore ExtJS admin UI to Studio v2 React/TypeScript — panels, grids, forms, trees, stores, plugins |
| **coreshop** | Extend and customize CoreShop eCommerce — custom rules, payment gateways, entity extensions, workflows, Studio v2 plugins, notifications |

## Installation

### Via Plugin Marketplace (recommended)

Add this repository as a Claude Code plugin marketplace, then install the plugins you need:

```bash
# Add the marketplace
/plugin marketplace add cors-gmbh/pimcore-skills

# Install Pimcore skills (pimcore, pimcore-studio, pimcore-studio-migrate)
/plugin install pimcore@pimcore-skills

# Install CoreShop skills
/plugin install coreshop@pimcore-skills
```

### Manual Installation

Copy the `skills/` directory into your project's `.claude/skills/` folder:

```bash
# From your Pimcore project root
mkdir -p .claude/skills
cp -r /path/to/pimcore-skills/skills/* .claude/skills/
```

## Usage

Once installed, you can invoke skills directly in Claude Code:

```
/pimcore
/pimcore-studio
/pimcore-studio-migrate
/coreshop
```

Claude will also automatically use these skills when it detects relevant Pimcore development tasks.

## What's Included

Each skill contains:

- **SKILL.md** — Main skill definition with step-by-step guidance, code examples, and architectural context
- **reference.md / patterns.md / migration-checklist.md** — Supplementary reference material with templates, cheat sheets, and checklists

## License

MIT — see [LICENSE](LICENSE) for details.