<div align="center">
<p><img alt="Salvo" width="132" style="max-width:40%;min-width:60px;" src="https://salvo.rs/images/logo-text.svg" /></p>
<h1>Salvo AI Agent Skills</h1>
<p>
    <a href="https://github.com/salvo-rs/salvo">Salvo Framework</a>&nbsp;&nbsp;
    <a href="https://salvo.rs">Documentation</a>&nbsp;&nbsp;
    <a href="https://discord.gg/G8KfmS6ByH">Discord</a>
</p>
<p>
<a href="https://salvo.rs">
    <img alt="Website" src="https://img.shields.io/badge/https-salvo.rs-%23f00" />
</a>
<a href="https://crates.io/crates/salvo"><img alt="crates.io" src="https://img.shields.io/crates/v/salvo" /></a>
<a href="https://docs.rs/salvo"><img alt="Documentation" src="https://docs.rs/salvo/badge.svg" /></a>
</p>
</div>

AI agent skills for the [Salvo](https://salvo.rs) web framework. These skills help AI assistants understand and generate Salvo code more effectively.

## 📚 What are Agent Skills?

Agent Skills are specialized knowledge modules that AI assistants can load to perform specific tasks. They follow the [Agent Skills](https://agentskills.io) open standard and work with tools like GitHub Copilot, Claude Code, and other AI coding assistants.

## 🎯 Available Skills

### Core Framework

- **[salvo-basic-app](./salvo-basic-app)** - Create basic Salvo applications with handlers, routers, and server setup
- **[salvo-routing](./salvo-routing)** - Configure routers with path parameters, nested routes, and filters
- **[salvo-middleware](./salvo-middleware)** - Implement middleware for authentication, logging, CORS, and request processing

### Data Handling

- **[salvo-data-extraction](./salvo-data-extraction)** - Extract and validate data from requests (JSON, forms, query params, path params)
- **[salvo-database](./salvo-database)** - Integrate databases using SQLx, Diesel, SeaORM, or other ORMs
- **[salvo-file-handling](./salvo-file-handling)** - Handle file uploads, downloads, and serve static files

### Advanced Features

- **[salvo-openapi](./salvo-openapi)** - Generate OpenAPI documentation automatically from handlers
- **[salvo-auth](./salvo-auth)** - Implement authentication and authorization (JWT, Basic Auth, sessions)
- **[salvo-realtime](./salvo-realtime)** - Implement WebSocket and Server-Sent Events for real-time features
- **[salvo-testing](./salvo-testing)** - Write unit and integration tests using TestClient

## 🚀 Quick Start

### For GitHub Copilot / VS Code

1. Copy the `salvo-skills` folder to your project:
   ```bash
   cp -r salvo-skills .github/skills/
   ```

2. Enable agent skills in VS Code settings:
   ```json
   {
     "chat.useAgentSkills": true
   }
   ```

3. Skills will automatically activate when you ask Copilot about Salvo!

### For Claude Code

1. Copy the skills to your project:
   ```bash
   cp -r salvo-skills .claude/skills/
   ```

2. Skills will be automatically loaded when working with Salvo code.

### For Personal Use

Copy to your home directory for use across all projects:

```bash
# GitHub Copilot
cp -r salvo-skills ~/.github/skills/

# Claude Code
cp -r salvo-skills ~/.claude/skills/
```

## 💡 Usage Examples

Once installed, you can ask your AI assistant questions like:

- "Create a basic Salvo web server with a hello world endpoint"
- "Add JWT authentication to my Salvo API"
- "How do I extract JSON data from a POST request in Salvo?"
- "Set up a WebSocket chat handler in Salvo"
- "Generate OpenAPI documentation for my Salvo endpoints"

The AI will use these skills to provide accurate, framework-specific guidance.

## 📖 Skill Structure

Each skill contains:

- **SKILL.md** - Main skill file with YAML frontmatter and instructions
- **Examples** - Code snippets demonstrating common patterns
- **Best Practices** - Framework-specific recommendations
- **Setup Instructions** - Dependencies and configuration

## 🤝 Contributing

Contributions are welcome! To add or improve skills:

1. Fork the repository
2. Create a new skill directory or modify existing ones
3. Follow the [Agent Skills specification](https://agentskills.io)
4. Submit a pull request

### Skill Format

Each skill must have a `SKILL.md` file with:

```markdown
---
name: skill-name
description: Brief description of what the skill does and when to use it
---

# Skill Title

Detailed instructions, examples, and best practices...
```

## 📝 License

These skills are part of the Salvo project and follow the same license.

## 🔗 Links

- [Salvo Framework](https://github.com/salvo-rs/salvo)
- [Salvo Documentation](https://salvo.rs)
- [Agent Skills Specification](https://agentskills.io)
- [Discord Community](https://discord.gg/G8KfmS6ByH)

## ⭐ Support

If you find these skills helpful, please consider:

- Starring the [Salvo repository](https://github.com/salvo-rs/salvo)
- Joining our [Discord community](https://discord.gg/G8KfmS6ByH)
- Contributing improvements and new skills

---

<div align="center">
Made with ❤️ for the Salvo community
</div>
