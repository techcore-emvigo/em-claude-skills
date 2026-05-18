## Quick Start

```bash
/plugin marketplace add jeffallan/claude-skills
```

**Then, install the skills:**

```bash
/plugin install fullstack-dev-skills@jeffallan
```

For all installation methods and first steps, see the **[Quick Start Guide](QUICKSTART.md)**.

**Full documentation:** [jeffallan.github.io/claude-skills](https://jeffallan.github.io/claude-skills)

## Skills

66 specialized skills across 12 categories covering languages, backend/frontend frameworks, infrastructure, APIs, testing, DevOps, security, data/ML, and platform specialists.

See **[Skills Guide](SKILLS_GUIDE.md)** for the full list, decision trees, and workflow combinations.

## Usage Patterns

### Context-Aware Activation

Skills activate automatically based on your request:

```bash
# Backend Development
"Implement JWT authentication in my NestJS API"
→ Activates: NestJS Expert → Loads: references/authentication.md

# Frontend Development
"Build a React component with Server Components"
→ Activates: React Expert → Loads: references/server-components.md
```

### Multi-Skill Workflows

Complex tasks combine multiple skills:

```
Feature Development: Feature Forge → Architecture Designer → Fullstack Guardian → Test Master → DevOps Engineer
Bug Investigation:   Debugging Wizard → Framework Expert → Test Master → Code Reviewer
Security Hardening:  Secure Code Guardian → Security Reviewer → Test Master
```

## Context Engineering

Surface and validate Claude's hidden assumptions about your project with `/common-ground`. See the **[Common Ground Guide](docs/COMMON_GROUND.md)** for full documentation.

## Project Workflow

9 workflow commands manage epics from discovery through retrospectives, integrating with Jira and Confluence. See [**Workflow Commands Reference](docs/WORKFLOW_COMMANDS.md)** for the full command reference and lifecycle diagrams.

> [!TIP]
> **Setup:** Workflow commands require an Atlassian MCP server. See the **[Atlassian MCP Setup Guide](docs/ATLASSIAN_MCP_SETUP.md)**.

## Documentation

- **[Quick Start Guide](QUICKSTART.md)** - Installation and first steps
- **[Skills Guide](SKILLS_GUIDE.md)** - Skill reference and decision trees
- **[Common Ground](docs/COMMON_GROUND.md)** - Context engineering with `/common-ground`
- **[Workflow Commands](docs/WORKFLOW_COMMANDS.md)** - Project workflow commands guide
- **[Atlassian MCP Setup](docs/ATLASSIAN_MCP_SETUP.md)** - Atlassian MCP server setup
- **[Local Development](docs/local_skill_development.md)** - Local skill development
- **[Contributing](CONTRIBUTING.md)** - Contribution guidelines
- **skills//SKILL.md** - Individual skill documentation
- **skills//references/** - Deep-dive reference materials

## Contributing

See **[Contributing](CONTRIBUTING.md)** for guidelines on adding skills, writing references, and submitting pull requests.

## Changelog

See [Changelog](CHANGELOG.md) for full version history and release notes.

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Support

- **Issues:** [GitHub Issues](https://github.com/jeffallan/claude-skills/issues)
- **Discussions:** [GitHub Discussions](https://github.com/jeffallan/claude-skills/discussions)
- **Repository:** [github.com/jeffallan/claude-skills](https://github.com/jeffallan/claude-skills)

## Author

Built by **[jeffallan](https://jeffallan.github.io)** 

**Principal Consultant** at **[Synergetic Solutions](https://synergetic.solutions)** 

Fullstack engineering, security engineering, compliance, and technical due diligence.

## Community

[Stargazers repo roster for @Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills/stargazers)

## Star History

[Star History Chart](https://www.star-history.com/#Jeffallan/claude-skills&type=date&legend=top-left)

---

**Built for Claude Code** | **9**** Workflows** | **366**** Reference Files** | **66**** Skills**