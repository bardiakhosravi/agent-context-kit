# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a documentation repository containing architectural standards and best practices for AI agents building Python backend services. It focuses on Domain Driven Design (DDD) and Hexagonal Architecture patterns.

## Key Files

- `domain_driven_design_hexagonal_arhictecture_python_rules.md` - The main standards document containing 24 comprehensive rules for building backend services
- `README.md` - Project philosophy, usage instructions, and contribution guidelines

## Working with This Repository

Since this is a documentation repository, there are no build, test, or lint commands. When making contributions:

1. **Editing Standards**: Focus on clarity, practicality, and consistency with existing rules
2. **File Naming**: Note that "arhictecture" in the main file name is a typo but should be preserved for compatibility
3. **Rule Format**: Each rule follows the pattern "Rule N: [Title]" with clear explanations and examples

## Architecture Principles

This repository documents:
- **Domain Driven Design**: Entities, Value Objects, Aggregates, Domain Services, Repository pattern
- **Hexagonal Architecture**: Ports & Adapters with clear separation between domain, application, and infrastructure layers
- **Testing Strategy**: Unit tests for domain logic, integration tests for adapters, contract tests for boundaries

## Contributing

When adding or modifying rules:
- Ensure rules are practical and battle-tested
- Provide clear examples where helpful
- Maintain consistency with existing rule numbering and format
- Consider how AI agents will interpret and apply the rules