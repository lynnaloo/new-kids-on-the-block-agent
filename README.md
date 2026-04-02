# New Kids on the Block Agent

A Salesforce expert agent system for Claude Code, themed around New Kids on the Block. A manager agent routes questions to specialized subagents, each named after a band member.

## Agents

| Agent | Invoke | Specialty | Vibe |
|-------|--------|-----------|------|
| Manager | `@manager` | Routes questions to the right specialist | The band manager |
| Donnie | `@Donnie` | Salesforce Data Cloud / Data 360 | Passionate, outspoken hype man |
| Joey | `@Joey` | Salesforce AI & Agentforce | Energetic showman |
| Jordan | `@Jordan` | CRM, Marketing Cloud, platform & integrations | Cool, confident, quietly in command |
| Jon | `@Jon` | Slack | Shy, grounded, calm and nurturing |
| Danny | `@Danny` | Service Cloud | Disciplined, loyal backbone |
| nkotb-fans | `@nkotb-fans` | NKOTB band history & trivia | Proud Blockhead, Gen X energy |

## Using the Agents

### Setup

Requires [Claude Code](https://claude.ai/code). Clone this repo and the agents are available immediately — no additional configuration needed.

If you don't have Git installed, get it first:
- **Mac:** `brew install git` (requires [Homebrew](https://brew.sh)) or install [Xcode Command Line Tools](https://developer.apple.com/xcode/resources/) via `xcode-select --install`
- **Windows:** Download from [git-scm.com](https://git-scm.com/download/win)
- **Linux:** `sudo apt install git` (Debian/Ubuntu) or `sudo dnf install git` (Fedora)

Then clone the repo:

```bash
git clone https://github.com/lynnaloo/new-kids-on-the-block-agent
cd new-kids-on-the-block-agent
```

Start Claude Code in your terminal from the repo directory:

```bash
claude
```

Then invoke any agent with an `@`-mention:

```
@manager What Data Cloud capabilities should I highlight for a retail customer?
@Donnie How does identity resolution work in Data Cloud?
@Joey Walk me through building an Agentforce agent with a custom action.
@Jordan What's the best way to set up lead routing in Sales Cloud?
@Jon How do I set up Slack Connect for an external partner?
@Danny How should I configure omni-channel routing for a high-volume support team?
@nkotb-fans Who was the youngest member of NKOTB?
```

Use `@manager` when you're not sure which specialist to ask — it will route automatically.

## Contributing

The agents get smarter when their knowledge does. Contributions are welcome — especially additions to the context documents in `.claude/context/`.

Good contributions include:
- New architectural patterns, platform limits, or implementation gotchas
- Corrections to outdated feature information
- New context documents for agents that don't have one yet (Jordan, Jon, Danny)
- Improvements to agent personas or routing logic
- Additional trivia context about NKOTB

To contribute, fork the repo, make your changes, and open a pull request. Keep context documents factual, cite the Salesforce release if relevant, and avoid speculating about unannounced features.

If you have a feature request, please add an issue.
