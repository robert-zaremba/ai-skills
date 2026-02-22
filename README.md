# AI Agent Skills

A comprehensive set of skills for AI Agents. Focus on developing secure, efficient, and idiomatic code.

Skills are tested with [OpenCode](https://opencode.ai), but you can use them with Kilo, Crush, Claude Code and many other tools.

Skills:

- [Sui Move](./sui-move/)

## Installation

Clone this repository and link the provided skill directories.

### Option 1: Project-Level

Place the skill in your project's `.opencode/skills/` directory:

```bash
# From your project root
mkdir -p .opencode/skills
ln -s sui-move .opencode/skills/

# Claude Code
mkdir -p .claude/skills
ln -s sui-move .claude/skills/
```

### Option 2: Global Installation

Make the skill available across all projects:

```bash
# OpenCode global location
mkdir -p ~/.config/opencode/skills
ln -s sui-move ~/.config/opencode/skills/

# Claude Code
mkdir -p ~/.claude/skills
ln -s sui-move ~/.claude/skills/
```

## Usage

### With OpenCode

Once installed, the skill will be automatically discovered. The agent can load it when working on Sui Move code:

```
/<skill-name>
# or
Ctrl+p skills
```

Or explicitly:

```
> Use the sui-move skill to review my smart contract for best practices
```

### With Claude Code CLI

The skill works with Claude Code CLI as well. Just say:

```
> Load the sui-move skill
```

Then ask questions or request help:

```
> Help me implement the capability pattern for my admin functions
> Review this code for anti-patterns
> How should I test this flash loan function?
```

## Example Prompts

```
# Create new module
Load sui-move and help me create an NFT module with the capability pattern

# Code review
Review my Move code against the anti-patterns section

# Testing
How should I test a function that uses shared objects?

# Best practices
Check my code against the code quality checklist

# Patterns
Explain the hot potato pattern with a flash loan example

# Debugging
Why does this fail with internal constraint error?
```

## Contributing

Found an issue or want to improve the skill? Contributions are welcome!

1. Fork or clone the repository
2. Submit a pull request or share your improvements

## License

See [LICENSE](./LICENSE).
