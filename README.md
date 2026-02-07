# X Developer Platform Skills

Agent skills for building with the [X API](https://developer.x.com/) using the official TypeScript and Python XDKs.

Skills are reusable capabilities for AI coding agents. Install them to give your agent expert knowledge of the X API and the official SDKs. Built on the open [Agent Skills](https://agentskills.io/) specification.

[![View on skills.sh](https://img.shields.io/badge/skills.sh-xdk-blue)](https://skills.sh/l4r-s/xdevplatform-skills/xdk)

## Available Skills

| Skill | Description |
|---|---|
| [xdk](skills/xdk/SKILL.md) | Build applications with the X API using the official TypeScript (`@xdevplatform/xdk`) and Python (`xdk`) SDKs. Covers setup, authentication, pagination, streaming, media upload, and best practices. |

## Installation

### Any Agent (via skills.sh)

```
npx skills add l4r-s/xdevplatform-skills
```

### Cursor

1. Open Cursor Settings (`Cmd+Shift+J` / `Ctrl+Shift+J`)
2. Navigate to **Rules & Command** > **Project Rules** > **Add Rule** > **Remote Rule (GitHub)**
3. Enter: `https://github.com/l4r-s/xdevplatform-skills.git`

The agent will automatically discover and use the skill based on context when you ask about the X API.

### Claude Code

```
/install-skill https://github.com/l4r-s/xdevplatform-skills
```

## What the XDK Skill Covers

- **SDK setup** for TypeScript (`@xdevplatform/xdk`) and Python (`xdk`)
- **Authentication** -- Bearer Token, OAuth 2.0 PKCE, and OAuth 1.0a
- **Pagination** -- async iterators (TypeScript) and automatic iterator-based paging (Python)
- **Streaming** -- filtered stream, sampled stream, firehose, and connection management
- **Media upload** -- one-shot and chunked uploads
- **Fields and expansions** -- requesting exactly the data you need
- **Error handling** -- typed errors and rate limit best practices
- **All API clients** -- posts, users, lists, DMs, spaces, communities, trends, webhooks, and more

## Skill Files

```
skills/xdk/
├── SKILL.md                  # Main skill instructions and quick start
├── api-concepts.md           # X API v2 concepts (response structure, rate limits, search operators)
├── typescript-patterns.md    # TypeScript SDK patterns and reference
└── python-patterns.md        # Python SDK patterns and reference
```

## License

Apache 2.0
