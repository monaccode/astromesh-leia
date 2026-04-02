# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-04-02

### Added

- Initial release of astromesh-leia Claude Code plugin
- 10 slash commands: `/leia`, `create`, `deploy`, `status`, `logs`, `test`, `templates`, `config`, `bootstrap`, `teardown`
- 5 specialized subagents: `leia-interpreter` (sonnet), `leia-architect` (opus), `leia-operator` (sonnet), `leia-tester` (sonnet), `leia-doctor` (sonnet)
- 6 business-vertical templates: customer support, restaurant booking, ecommerce, appointment scheduling, lead qualification, employee onboarding
- 4 schema references: astromesh/v1 agent spec, nexus REST API, WhatsApp channel config, orchestration patterns
- Multi-context configuration with `~/.astromesh-leia/config.yaml`
- Full cluster lifecycle management (bootstrap Kind, connect remote, teardown)
- Smart model detection (Ollama auto-detect with cloud fallback)
- Interactive and automated agent testing with per-template test scenarios
- Diagnostic engine (`leia-doctor`) for troubleshooting agent and cluster issues
- Comprehensive documentation with Mermaid diagrams:
  - Architecture guide with decision log
  - Complete commands reference (10 commands)
  - Subagents guide (5 agents)
  - Templates guide (6 templates with customization examples)
  - Configuration reference
  - WhatsApp end-to-end setup guide
  - Troubleshooting guide with diagnostic flowcharts
  - 4 tutorials: first agent, business templates, multi-tenant, advanced patterns

[0.1.0]: https://github.com/monaccode/astromesh-leia/releases/tag/v0.1.0
