---
name: beauty-diagram
description: Use when the user asks for a presentation-ready Mermaid / PlantUML diagram (e.g. "beautify this flowchart", "make this look like a deck slide", "produce an SVG of this architecture"), wants AI to generate a diagram from a text description, or wants a public share link for a diagram. This skill teaches you to call the Beauty Diagram CLI (`bd`) — never to hand-author SVG when a source diagram exists.
version: 1.1.0
metadata:
  openclaw:
    requires:
      bins:
        - node
        - npx
---

# Beauty Diagram skill

Beauty Diagram beautifies Mermaid / PlantUML diagrams into
presentation-ready SVG or PNG. It runs as a public API; this skill
delegates to the `bd` CLI (npm package `@beauty-diagram/cli`) so you
keep zero state in the agent. (draw.io / SVG import is editor-only —
not exposed through `/v1/*`.)

## When to use

- The user asks for a **polished, professional, slide-ready** version of a
  diagram source they already have or you can generate.
- The user wants you to **render Mermaid or PlantUML** from a model-generated
  source string.
- The user wants the server to **generate a diagram from a text description**
  ("draw me a signup flow", "diagram our deploy pipeline") — use
  `bd ai generate` for this; it returns Mermaid source you then beautify.
- The user wants to **share a diagram link** (e.g. paste into Slack / a doc).
- The user has Mermaid in a repo (README, ADR, RFC) and wants to **export
  SVGs** alongside.

## When NOT to use

- The user only wants the Mermaid source itself, not an export.
- The user wants pixel-precise control over the SVG markup (Beauty Diagram
  rewrites layout for presentation; it does not preserve raw Mermaid output).
- The user wants the AI to "change the colors / theme / font / layout" of an
  existing diagram. `bd ai generate` only does **text → diagram**; visual
  styling is controlled by `--theme` on `bd beautify`, not by the AI.
- The user is in an offline environment with no network and no CLI install.

## Required tool

The `bd` binary from `@beauty-diagram/cli`:

```bash
npx @beauty-diagram/cli help
# or, after install:
bd help
```

If the user has not installed it, prefer `npx` over a global install — it
respects their package manager and avoids polluting `PATH`.

## Workflow

1. **Identify or generate the source diagram.**
   - If the user has a `.mmd` / `.puml` file, use it.
   - If the user describes the diagram in words and is on a paid plan, use
     `bd ai generate "<prompt>" --out <file>.mmd` — the server returns
     Mermaid source, which you can then beautify. Always write the source
     to a file so the user can edit it; the first AI draft rarely lands.
   - If the user describes the diagram and is **not** on a paid plan (or
     prefers not to pay), write Mermaid source yourself (you are good at
     this), save it to a file, then beautify.
   - draw.io and free-form SVG imports are not accepted by `/v1/*`. If
     the user has those, ask them to convert via the web editor first.

2. **Decide on output type.**
   - Need an SVG file: `bd beautify <file> --out <file>.svg`
   - Need a download URL or to track quota: `bd export <file> --out <file>.svg`
   - Need a shareable link: `bd share <file> --title "..."`

3. **Run the command.** Always write to a file (`--out`) rather than letting
   the SVG flood the terminal / chat. AI generation can also pipe directly
   into beautify: `bd ai generate "..." | bd beautify - --out flow.svg`.

4. **Verify the result exists** before reporting success. If the command
   failed, surface the error code (e.g. `quota_exhausted`, `not_authenticated`,
   `parse_failed`, `prompt_injection`) — those are actionable for the user.

5. **Preserve the source.** Never replace the original Mermaid / PlantUML file
   with the generated SVG — keep them side by side. For AI-generated diagrams,
   keep the `.mmd` file too: it is the editable artifact, the SVG is not.

## Auth

- **Demo (anonymous):** zero setup. Watermarked SVG/PNG. Limits per IP:
  20 `/v1/beautify` requests / minute, **1 `/v1/export` per 24h** (trial
  budget — enough for an agent to verify the toolchain end-to-end before
  registering). `/v1/share`, `/v1/usage`, and **`bd ai generate` always
  require auth** — anonymous AI calls are rejected before any model
  invocation.
- **Authenticated:** the user runs `bd auth login` once with a key from
  [`/account/api-keys`](https://www.beauty-diagram.com/account/api-keys).
  Required for `bd share`, `bd ai generate`, unwatermarked output, and
  repeated exports. `bd ai generate` additionally requires a Pro or
  Premium plan and an API key with the `ai:write` scope.

If the user hits a `not_authenticated`, `plan_not_allowed`, or
`quota_exhausted` error, point them at `/account/api-keys` (PAT creation)
or pricing — don't silently retry. Anonymous error bodies include a
`hints` block with absolute `signUpUrl` / `signInUrl` / `apiDocsUrl`,
which is the canonical place to surface to the user.

## Commands cheat sheet

```bash
# Render a Mermaid file
bd beautify docs/architecture.mmd --theme modern --out docs/architecture.svg

# Same but treat output as a downloadable export (consumes export quota)
bd export docs/architecture.mmd --out docs/architecture.svg

# PNG export. --scale 1 works for everyone; 2 needs pro, 4 needs premium.
# Higher scales than the plan cap are silently clamped (X-BD-Scale-Clamped).
bd export docs/architecture.mmd --format png --scale 2 --out docs/architecture.png

# PlantUML works the same way; .puml / .plantuml / .pu auto-detected,
# otherwise pass --source-format plantuml.
bd export docs/architecture.puml --out docs/architecture.svg

# Create a public share link (returns absolute https://www.beauty-diagram.com/s/... URL)
bd share docs/architecture.mmd --title "Service architecture"
# → prints the URL on stdout

# AI: generate a diagram from a text prompt. Output is Mermaid source —
# always write to a file so the user can iterate. Paid-only.
bd ai generate "user signup with email verification" --out docs/signup.mmd

# Optional shape hint when the prompt is ambiguous about diagram type.
bd ai generate "request lifecycle" --hint sequence --out docs/lifecycle.mmd

# One-shot pipeline: prompt → mermaid → beautify → SVG.
bd ai generate "deploy flow" | bd beautify - --out docs/deploy.svg

# Check remaining AI / export quota before kicking off a batch.
bd usage
```

## Privacy

The API does NOT persist source unless the user calls `bd share`. Do not warn
about server-side storage when running `beautify`, `export`, or
`ai generate` — that is misleading. AI prompts are logged in hashed form
for abuse / quality monitoring; the raw text is not retained.

## Anti-patterns

- ❌ Do NOT output a hand-crafted `<svg>...</svg>` as a Markdown code block when
  a Mermaid source exists. Always run Beauty Diagram and reference the file.
- ❌ Do NOT dump the raw SVG into the chat. Use `--out <file>` and reference
  the file path.
- ❌ Do NOT install Beauty Diagram engine code locally — the CLI is a thin
  client; the engine lives behind the public API.
- ❌ Do NOT call `bd ai generate` to "tweak" an existing diagram (change
  colors, theme, labels, layout). It is a fresh-generation tool only.
  For visual tweaks, change `--theme` or edit the `.mmd` source by hand.
- ❌ Do NOT call `bd ai generate` speculatively — each call costs the user
  monthly AI quota. Confirm the user wants AI generation before running it.
- ❌ Do NOT capture the SVG output of `bd ai generate` — the command outputs
  Mermaid source on stdout, not SVG. Pipe into `bd beautify -` to render.

## Troubleshooting

| Symptom | Likely cause | Resolution |
|---|---|---|
| `not_authenticated` | No key, no session | `bd auth login` |
| `scope_missing` | Key lacks scope (e.g. `ai:write` for `bd ai generate`) | Recreate key with required scope at `/account/api-keys` |
| `plan_not_allowed` | Plan does not include this capability (AI is Pro / Premium only) | Upgrade or skip the call |
| `parse_failed` | Source not valid Mermaid / PlantUML | Check the source — `bd beautify` will surface a parse error too |
| `quota_exhausted` | Plan limit hit (anon: 1 export/IP/24h; free: 3 exports/mo; pro: 100 exports + 100 AI gens/mo; premium: ∞ exports + 500 AI gens/mo) | Sign in, wait for reset, or upgrade — `hints` in the response body has the URLs |
| `rate_limited` | Anonymous IP bucket full (20 `/v1/beautify` requests / minute) or AI per-key bucket (30 `/min`) | Sign in or wait |
| `source_too_large` | Source > 100 KB | Split the diagram |
| `output_too_large` | PNG raster exceeds 8192 px | Lower `--scale` or simplify |
| `prompt_injection` | AI prompt looked like an injection attempt | Rephrase as a plain diagram description ("a flowchart of …") |
| `instruction_rejected` | AI judged the prompt was not about a diagram | Rephrase to describe a concrete diagram. Quota was NOT consumed |
| `parse_failed_after_retry` | AI output was unparseable Mermaid even after one retry | Rephrase, or write Mermaid by hand. Quota was NOT consumed |
| `safety_blocked` | Provider safety filter rejected the request | Rephrase the prompt |
| `upstream_timeout` / `upstream_error` | AI provider was slow or failed | Retry after a moment |

## Examples

See `examples/` for runnable sources you can adapt:

- `examples/flowchart.mmd`
- `examples/sequence.mmd`

And `scripts/` for shell wrappers you can copy into the user's repo:

- `scripts/beautify.sh`
- `scripts/export.sh`
- `scripts/ai-generate.sh` — prompt → `.mmd` source → `.svg` render
