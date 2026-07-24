# Schedulala Agent Skill

Teach any AI agent to schedule and publish social media posts to **12
platforms** — Twitter/X, Instagram, TikTok, LinkedIn, YouTube, Facebook,
Threads, Bluesky, Pinterest, Mastodon, Telegram, and Google Business Profile —
through [Schedulala](https://schedulala.com).

<!-- DEMO VIDEO PLACEHOLDER: connect-demo.gif goes here -->
> 🎬 Demo video coming soon — watch the connector go from zero to a scheduled
> post in under a minute.

## The fastest path: the hosted connector (claude.ai / Claude Desktop)

No install at all. In Claude, go to **Settings → Connectors → Add custom
connector** and paste:

```
https://schedulala.com/api/mcp
```

Claude discovers the OAuth flow automatically — you sign in on schedulala.com
and approve; no API keys to copy. You get every tool **plus interactive
widgets** rendered inline in the conversation:

- **Post previews** — pixel-faithful per-platform mockups (threads, carousels,
  reels, PDF documents, polls, live character counts) before anything publishes
- **In-chat media upload** — drag-and-drop photos (20MB), videos (500MB), and
  PDFs (100MB) straight into the conversation at full quality
- **Analytics cards, leaderboards, comparisons** — follower growth and top
  posts as interactive panels
- **Calendar & inbox** — scheduled posts with cancel actions; comments with
  reply/hide actions

> Note: custom connectors require a paid Claude plan. The connector also works
> in ChatGPT (Settings → Connectors) with the same URL.

Setup guide: https://schedulala.com/developers/docs/claude-connector

## Install the skill (Claude Code, Codex, and other agent CLIs)

Pick one:

```bash
# skills CLI
npx skills add schedulala/schedulala-agent

# Claude Code plugin marketplace
/plugin marketplace add schedulala/schedulala-agent

# plain git
git clone https://github.com/schedulala/schedulala-agent ~/.claude/skills/schedulala-agent
```

The skill teaches the agent the safe publishing workflow (draft → validate /
preview → confirm → publish → verify), the media decision tree, and the
per-platform rules that reject posts (TikTok privacy + postMode, Instagram
media requirements, YouTube titles, Pinterest boards, character limits, …).

Pair it with either:

- the **local MCP server** — `npx -y @schedulala/mcp-server` with
  `SCHEDULALA_API_KEY` (Claude Desktop / Claude Code / Cursor config in the
  [package README](https://www.npmjs.com/package/@schedulala/mcp-server)), or
- the **CLI** — `npx schedulala init --email you@example.com`, then
  `npx schedulala post "Hello" --platforms twitter --now`. Every command
  supports `--json`.

## What's in this repo

```
skills/schedulala/SKILL.md      the skill: workflow, rules, decision trees
skills/schedulala/references/   per-platform schemas, media protocol, CLI, widgets
examples/                       ready-to-adapt JSON payloads (tiktok, threads, bulk, …)
.claude-plugin/                 Claude Code plugin + marketplace manifest
```

## Links

- Developer docs (REST API, CLI, connector): https://schedulala.com/developers/docs
- Connector setup: https://schedulala.com/developers/docs/claude-connector
- Claude integration page: https://schedulala.com/integrations/claude
- CLI on npm: [`schedulala`](https://www.npmjs.com/package/schedulala) ·
  MCP server on npm: [`@schedulala/mcp-server`](https://www.npmjs.com/package/@schedulala/mcp-server)
- Pricing: https://schedulala.com/pricing

## License

MIT — see [LICENSE](LICENSE). The skill documents the Schedulala service,
which is a separate commercial product.
