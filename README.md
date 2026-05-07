# AI Coding Guides

Reusable project guidance files for AI coding tools.

These files are meant to be copied into real projects so tools like Claude Code, Codex, and similar agents have durable instructions before they write code. The goal is not to make AI follow a rigid template. The goal is to make architectural preferences explicit enough that generated code stays aligned with the project.

## Available Guides

- [iOS Architecture Guide](ios/ARCHITECTURE.md): an opinionated `ARCHITECTURE.md` for Swift and SwiftUI projects.

## How To Use

Copy the guide that matches your project into the root of the app repository:

```bash
curl -o ARCHITECTURE.md https://raw.githubusercontent.com/frugoman/ai-coding-guides/main/ios/ARCHITECTURE.md
```

Then tell your AI coding tool to follow it when making changes.

## Why This Exists

AI can generate code quickly. That makes architecture more important, not less important.

When code is cheap to produce, the hard part becomes keeping decisions consistent:

- Where should dependencies be created?
- Who owns navigation?
- What belongs in a ViewModel?
- When is a use case useful?
- When is a protocol just ceremony?
- How should tests protect behavior without freezing implementation details?

A project guide gives those answers a stable home.

## Updating

These guides are living documents. They should change when real project work reveals better wording, missing examples, or repeated mistakes.
