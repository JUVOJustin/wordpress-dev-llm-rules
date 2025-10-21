# WordPress Development Rules for LLMs

A comprehensive collection of WordPress development guidelines, coding standards, and best practices designed specifically for Large Language Models (LLMs) and AI-powered development tools.

## 📋 Overview

This repository contains structured rules and patterns that help LLMs generate better WordPress code by following established conventions, security practices, and performance optimizations. The rules are organized into focused modules covering different aspects of WordPress development.

## 🚀 Quick Start

### Instruct AI to setup rules
My plugin boilerplate uses this command to instruct AI to clone and add the rules: https://raw.githubusercontent.com/JUVOJustin/wordpress-plugin-boilerplate/refs/heads/main/.opencode/command/rules-upsert.md

This command is setup for github copilot and opencode integration. You can adjust the folders to your setup

### For LLM Integration

* **Copying Rules**: You can copy rules from this repository to your LLM integration tool. Full control over the rules you want to use!
* **MCP Server**: Use the [WordPress Dev Community MCP Server](https://github.com/Citation-Media/wordpress-dev-community-mcp-server) for seamless integration

### MCP Server
My agency [Citation Media](https://citation.media) does offer a free MCP Server. It serves the rules of this repo as well as other nice to have functionality.
For **Claude Desktop, Windsurf or Cursor** add this to your MCP configuration:

```json
{
  "mcpServers": {
    "wordpress-dev-docs": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://wordpress-dev-mcp.citation.media/mcp"
      ]
    }
  }
}
```

For **Claude code** use:
```bash
claude mcp add wordpress-dev-docs --transport http https://wordpress-dev-mcp.citation.media/mcp -s project
```

For **Github Copilot** use:
```json
{
  "mcpServers": {
    "wordpress-dev-docs": {
      "type": "http",
      "url": "https://wordpress-dev-mcp.citation.media/mcp",
      "tools": [ "*" ]
    }
  }
}
```

## 🤝 Contributing

We encourage contributions from the WordPress development community! If you have rules, patterns, or best practices that would benefit other developers, please consider opening a Pull Request.

### How to Contribute

1. **Fork** this repository
2. **Create** a new branch for your changes
3. **Add** your rules following the existing format and structure
4. **Test** your rules with LLM tools to ensure they work effectively
5. **Submit** a Pull Request with a clear description of your additions

### Contribution Guidelines

- Follow the existing documentation format
- Include practical code examples
- Focus on WordPress-specific patterns and conventions
- Ensure rules are actionable and specific
- Test with multiple LLM tools when possible
