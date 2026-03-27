# Pre-Build Idea Validator

Before OpenClaw starts building anything new, it automatically checks whether the idea already exists across GitHub and Hacker News — and adjusts its approach based on what it finds.

## What It Does

- Scans real data sources (GitHub, Hacker News) before any code is written
- Returns a `reality_signal` score (0-100) indicating how crowded the space is
- Shows top competitors with star counts and descriptions
- Suggests pivot directions when the space is saturated
- Works as a pre-build gate: high signal = stop and discuss, low signal = proceed

## Pain Point

You tell your agent "build me an AI code review tool" and it happily spends 6 hours coding. Meanwhile, 143,000+ repos already exist on GitHub — the top one has 53,000 stars. The agent never checks because you never asked, and it doesn't know to look. You only discover competitors after you've invested significant time. This pattern repeats for every new project idea.

## Skills You Need

- [idea-reality-mcp](https://github.com/mnemox-ai/idea-reality-mcp) — install via pip or uvx

## How to Set It Up

### 1. Install idea-reality-mcp

```bash
uv tool install idea-reality-mcp
```

### 2. Create the CLI wrapper at /usr/local/bin/idea-check

Save this as `/usr/local/bin/idea-check` and make it executable (`chmod +x`):

```js
#!/usr/bin/env node
/**
 * CLI wrapper for idea-reality-mcp
 * Usage: idea-check "your idea description" [--depth quick|deep]
 */
import { spawn } from 'node:child_process';
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  options: {
    depth: { type: 'string', default: 'quick' },
    help: { type: 'boolean', short: 'h', default: false },
  },
  allowPositionals: true,
  strict: false,
});

if (values.help || positionals.length === 0) {
  console.log('Usage: idea-check "your idea" [--depth quick|deep]');
  process.exit(0);
}

const ideaText = positionals.join(' ');
const depth = values.depth;

function sendLine(proc, obj) {
  proc.stdin.write(JSON.stringify(obj) + '\n');
}

// Find the idea-reality-mcp binary (installed via uv tool install)
const mcpBin = '/usr/local/bin/idea-reality-mcp';

const proc = spawn(mcpBin, [], { stdio: ['pipe', 'pipe', 'pipe'] });

let buffer = '', initialized = false, done = false;

proc.stdout.on('data', chunk => {
  buffer += chunk.toString();
  const lines = buffer.split('\n');
  buffer = lines.pop();
  for (const line of lines) {
    if (!line.trim()) continue;
    let msg;
    try { msg = JSON.parse(line); } catch { continue; }
    if (msg.id === 1 && msg.result?.serverInfo && !initialized) {
      initialized = true;
      sendLine(proc, { jsonrpc: '2.0', method: 'notifications/initialized' });
      sendLine(proc, { jsonrpc: '2.0', id: 2, method: 'tools/call',
        params: { name: 'idea_check', arguments: { idea_text: ideaText, depth } } });
    }
    if (msg.id === 2 && !done) {
      done = true;
      const text = (msg.result?.content ?? []).map(c => c.text ?? '').join('\n').trim();
      console.log(text);
      proc.kill(); process.exit(0);
    }
    if (msg.error && !done) {
      done = true;
      console.error('MCP error:', msg.error.message);
      proc.kill(); process.exit(1);
    }
  }
});

proc.stderr.on('data', d => {
  const s = d.toString();
  if (!s.includes('Downloading') && !s.includes('Installed') && !s.includes('Downloaded'))
    process.stderr.write(s);
});

proc.on('error', err => { console.error('Failed to start MCP server:', err.message); process.exit(1); });
proc.on('close', code => { if (!done) { console.error('MCP server exited early (code', code + ')'); process.exit(1); } });

sendLine(proc, { jsonrpc: '2.0', id: 1, method: 'initialize',
  params: { protocolVersion: '2024-11-05', capabilities: {}, clientInfo: { name: 'idea-check-cli', version: '1.0' } } });

setTimeout(() => { if (!done) { console.error('Timeout'); proc.kill(); process.exit(1); } }, 60000);
```

### 3. Make idea-reality-mcp accessible to all users

```bash
# Install to a shared location
UV_TOOL_DIR=/opt/uv-tools uv tool install idea-reality-mcp --force
# Point /usr/local/bin symlink to the shared install
ln -sf /opt/uv-tools/idea-reality-mcp/bin/idea-reality-mcp /usr/local/bin/idea-reality-mcp
```

### 4. Add the SKILL.md to your OpenClaw skills directory

Save this as `~/.openclaw/skills/pre-build-idea-validator/SKILL.md`:

```markdown
---
name: pre-build-idea-validator
description: >-
  Validate an idea before building. Use when starting any new project, feature,
  or tool before writing any code, or when the user shares a GitHub repo URL and
  wants to understand its idea or market position.
---

# Pre-Build Idea Validator

Before starting anything new, check whether it already exists.

## Run it

\`\`\`bash
node /usr/local/bin/idea-check "describe the idea here"
\`\`\`

Deep scan:
\`\`\`bash
node /usr/local/bin/idea-check "describe the idea here" --depth deep
\`\`\`

## GitHub repo URL mode

1. Fetch the README to extract a 1-sentence description
2. Run idea-check with that description
3. Report reality_signal + what the repo does

## Decision rules

| reality_signal | Action |
|---|---|
| > 70 | STOP. Show top 3 competitors with star counts. Ask whether to proceed, pivot, or abandon. |
| 30–70 | Show results and pivot_hints. Suggest a niche angle existing projects don't cover. |
| < 30 | Proceed. The space is open. |

Always show the score and top competitors before writing any code.
```

## Real World Example

**You:** "Build me a CLI tool for AI code review"

OpenClaw runs `idea-check "AI code review tool"` and responds:

> **reality_signal: 90/100 (very high)**
>
> Top competitors:
> 1. Gitea — 53,940 stars
> 2. reviewdog — 9,104 stars
> 3. Danger (Ruby) — 5,649 stars
>
> Should I proceed with a differentiated angle, or pivot?

## Key Insights

- Prevents the most expensive mistake in building: solving a problem that's already solved
- `reality_signal` is based on real data (repo counts, star distributions, HN discussion volume)
- A high score doesn't mean "don't build" — it means "differentiate or don't bother"
- A low score means genuine white space. That's where solo builders have the best odds

## Related Links

- [idea-reality-mcp GitHub](https://github.com/mnemox-ai/idea-reality-mcp)
- [Web demo](https://mnemox.ai/check) (try without installing)
- [PyPI](https://pypi.org/project/idea-reality-mcp/)
