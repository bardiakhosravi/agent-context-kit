# Agent Context Kit

Coding standards context for AI agents building backend services with Domain Driven Design and Hexagonal Architecture. 

## 🤖 Why This Exists

AI coding agents like Claude Code, Cursor, and Windsurf can generate incredible amounts of code quickly. But **I still care about the code quality**. We're not at the point where we can be completely hands-off—I constantly review, iterate, and guide these tools toward better implementations.

For effective code review and collaboration with AI agents, we need **predictable design patterns**. This is even more crucial with AI-generated code since these tools can produce entire features at once, making architectural consistency vital for maintainability.

## 🏗️ What's Inside

These rules are based on years of building backend services using:

- **Domain Driven Design (DDD)** - Rich domain models, ubiquitous language, clear boundaries
- **Hexagonal Architecture (Ports & Adapters)** - Separation of concerns, testability, technology independence
- **Python Best Practices** - Leveraging Python's strengths while maintaining clean architecture

## 🚀 How to Use With AI Agents

### Option 1: Direct Context
Copy the rules into your AI agent's context when starting a new project or feature:

```
Please follow the architectural rules in this repository when implementing backend services.
Focus on DDD and Hexagonal Architecture patterns as outlined in the guidelines.
```

### Option 2: Project-Specific Rules
Add these rules to your project's documentation and reference them:

```
Implement this feature following our backend standards: 
[link to relevant rule sections]
```

### Option 3: Tool Integration
Many AI coding tools support project-specific context files:
- Copy rules to `.cursorrules` for Cursor
- Add as project context in Claude Code
- Reference in Windsurf workspace settings


## 🎯 Language Support

This repository currently focuses on Python implementations, providing concrete examples and patterns optimized for Python's language features and ecosystem.

However, I'd love to collaborate with experts in other languages to expand this into a comprehensive reference for all major backend languages. If you're experienced with Java, C#, Go, TypeScript, or other languages, I welcome contributions that translate these architectural principles into language-specific implementations.

The goal is to make this the go-to reference for backend development standards across the entire ecosystem.

## 🤝 Contributing

This is a living document based on real-world experience, but there's always room for improvement! 

### How to Contribute

1. **Open an Issue** - Propose new rules, improvements, or discuss existing ones
2. **Submit a PR** - Add new patterns, fix examples, or improve clarity
3. **Share Examples** - Real-world implementations that follow these patterns
4. **Language Adaptations** - Help translate patterns to other languages

### What We're Looking For

- 🔍 **Uncovered Scenarios** - Backend development situations not yet addressed by the current rules
- 🎨 **Missing Design Patterns** - Additional patterns that complement DDD and Hexagonal Architecture
- 🛠️ **Implementation Strategies** - Better approaches for complex scenarios like distributed systems, event sourcing, or CQRS
- 🧪 **Advanced Testing Patterns** - Testing strategies for complex domain logic and integration scenarios
- 📊 **Real-World Examples** - Case studies showing these patterns in production systems

## 🧠 Philosophy

> "I think of AI coding agents as another developer on my team—I expect them to follow the same best practices."

These standards ensure that:

- **Code is reviewable** - Predictable patterns make AI-generated code easier to understand
- **Architecture is consistent** - Clear rules prevent architectural drift
- **Teams can collaborate** - Shared standards improve team efficiency
- **Systems remain maintainable** - Good architecture scales with your codebase

## 📚 Additional Resources

- [Medium Article: Why I Still Care About Code Quality with AI Agents](link-to-your-article)
- [Domain Driven Design by Eric Evans](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)
- [Hexagonal Architecture by Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Clean Architecture by Robert Martin](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

## ⭐ Support

If these standards help you build better backend services with AI agents, please:

- ⭐ Star this repository
- 🔗 Share with your team
- 💬 Open discussions for improvements
- 🤝 Contribute your own learnings

---

*Let's build better backend services together, one AI-generated feature at a time.*
