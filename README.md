# agent-inventory

A registry of modular, stateless AI agent skills. Fetch, equip, and integrate capabilities directly into your workflow via CLI.

## Installation

This repository follows the [skills](https://github.com/vercel-labs/skills) standard. You can add skills to your environment using the following command:

```bash
npx skills add nkchinh/agent-inventory --skill <skill-name>
```

## Skill Catalog

| Skill Name | Description |
| :--- | :--- |
| **dotnet-memleak-collab** | Collaborative human-agent workflow for diagnosing .NET memory leaks and deadlocks using `dotnet-dump`. |

### Detailed Skill Info

#### `dotnet-memleak-collab`
A structured investigation session where the agent provides expert direction and interpretation, and the developer executes commands and reports back. Use this whenever you encounter:
- Memory leaks or high memory usage in .NET.
- App hangs or deadlocks.
- Analyzing `.dmp` or core dump files.

**How to add:**
```bash
npx skills add nkchinh/agent-inventory --skill dotnet-memleak-collab
```

---

## How it works

These skills are designed to be stateless and modular, providing instructions and references that guide an AI agent through specific technical tasks or workflows. By adding a skill, you provide your agent with the specialized knowledge and procedures required to handle complex engineering challenges.

## Contributing

Contributions are welcome! If you have a skill you'd like to share:
1. Fork this repository.
2. Create a new directory for your skill.
3. Include a `SKILL.md` file following the standard format.
4. Submit a pull request.

## License

MIT
