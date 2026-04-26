# Changelog

All notable changes to this skill will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-04-26

### Added
- Version number in SKILL.md frontmatter (`version: 1.1.0`)
- `README.md` - GitHub entry documentation
- `CHANGELOG.md` - Version history
- `LICENSE` - MIT license
- `SETUP.md` - Complete setup guide for users and agents
- `validation-config.example.yaml` - Configuration template
- `references/troubleshooting.md` - Troubleshooting guide
- Preflight checks section in SKILL.md
- Installation section with detailed steps
- Blockers section in output-contract.md
- MCP server configuration instructions

### Changed
- Improved `agents/openai.yaml` with version info and capabilities
- Added Mermaid decision flowchart to route-matrix.md
- Fixed namespace typo in custom-tools-input.md
- Quick Start now references SETUP.md

## [1.0.0] - 2025-12-01

### Added
- Initial release
- Core validation routing logic (same as unity-validation-router)
- Unity-MCP tool reference (77 tools documented)
- Custom input simulation tool templates
- Runtime probes for PlayMode debugging
- Output contract for validation reports