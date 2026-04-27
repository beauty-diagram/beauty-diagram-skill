# Beauty Diagram skill

Turn Mermaid / PlantUML source into presentation-ready SVG or PNG — straight
from your agent. This skill teaches Claude (or any compatible agent) to call
the [`bd` CLI](https://www.npmjs.com/package/@beauty-diagram/cli) instead of
hand-authoring SVG, so you get consistent, slide-quality diagrams without
leaving the conversation.

## What it does

- Beautifies existing Mermaid / PlantUML files into polished SVG/PNG
- Renders model-generated diagram source on demand
- Produces shareable `https://www.beauty-diagram.com/s/...` links
- Surfaces actionable error codes (`quota_exhausted`, `parse_failed`, …)
  instead of silently retrying

## Requirements

- **Node.js** (for `npx @beauty-diagram/cli`); no global install needed
- **Optional**: a Beauty Diagram API key for unwatermarked output, share
  links, and higher quotas — anonymous demo mode works out of the box
  (1 export per IP per 24h)

Get a key at <https://www.beauty-diagram.com/account/api-keys>.

## Triggering

The skill activates when a user asks for things like:

- "beautify this flowchart"
- "make this Mermaid diagram look like a deck slide"
- "give me an SVG of this architecture"
- "share this diagram as a link"

## Example

```
You: Here's our service flow in Mermaid — make it slide-ready and give me a share link.

Agent (uses skill):
  $ bd beautify flow.mmd --theme modern --out flow.svg
  $ bd share flow.mmd --title "Service flow"
  → https://www.beauty-diagram.com/s/abc123
```

See `examples/` for runnable diagram sources and `scripts/` for shell
wrappers you can drop into a repo.

## Files

```
beauty-diagram-skill/
├── SKILL.md          # agent-facing instructions
├── examples/         # sample Mermaid sources
│   ├── flowchart.mmd
│   └── sequence.mmd
└── scripts/          # copy-pasteable shell wrappers
    ├── beautify.sh
    └── export.sh
```

## Links

- Site: <https://www.beauty-diagram.com>
- CLI on npm: <https://www.npmjs.com/package/@beauty-diagram/cli>
- API keys: <https://www.beauty-diagram.com/account/api-keys>

## License

MIT-0 (per ClawHub publishing terms).
