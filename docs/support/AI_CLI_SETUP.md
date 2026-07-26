# AI CLI Setup

**Purpose:** Optional setup for Codex CLI and Claude CLI on the Pi 500

AI command-line tools are optional helper tools. They can help explain code, suggest edits, and answer questions while teams work in the Pathfinder2026 repo.

These tools belong on the **Pi 500 only**. Do not install or configure AI tools on the robot.

---

## Event Policy

- AI CLI tools are optional.
- AI CLI tools may be preinstalled on the Pi 500 image.
- AI CLI tools should not be configured with tokens in the shared image.
- Do not install AI CLI tools on the robot.
- Do not store API keys, tokens, or login files in this repository.
- Do not paste secrets into markdown files, Python files, screenshots, or terminal output that will be shared.

## What May Be Preinstalled

| Tool | Command | Purpose |
|------|---------|---------|
| Codex CLI | `codex` | OpenAI coding assistant in the terminal |
| Claude CLI | `claude` | Anthropic Claude Code in the terminal |

Official install references:

- Codex CLI: <https://github.com/openai/codex>
- Claude Code: <https://code.claude.com/docs/en/getting-started>

## Verify The Tools

Run these commands on the Pi 500:

```bash
codex --version
claude --version
claude doctor
```

If the commands are found, the tools are installed.

If a command is not found, ask a facilitator. These tools are optional and should not block robot work.

## Configure Only If Needed

Only configure these tools if:

- a facilitator tells you to use them, or
- you already have your own account/token and permission to use it.

Start the tool from the Pi 500 terminal:

```bash
codex
claude
```

Follow the login or token prompts shown by the tool.

## Safe Use During The Workshop

Good uses:

- Ask what a Python file does.
- Ask how to change a demo safely.
- Ask for help understanding an error message.
- Ask for a checklist before editing robot code.

Avoid:

- Asking the tool to change many files at once during the event.
- Running commands you do not understand.
- Sharing API keys or tokens.
- Installing AI tools on the robot.
- Letting an AI tool overwrite team work without checking the diff first.

## Where To Run AI Commands

Run AI tools from a Pi 500 terminal, usually inside the local repo clone:

```bash
cd ~/Pathfinder2026
codex
```

or:

```bash
cd ~/Pathfinder2026
claude
```

Robot code still runs on the robot. Use AI tools on the Pi 500 to understand and edit code, then use the normal robot connection workflow to test changes.
