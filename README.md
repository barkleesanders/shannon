# Shannon

Shannon is a CLI and SDK wrapper around the interactive Claude Code CLI. It runs a real `claude` session inside tmux, sends a prompt, and emits stream JSON.

Public fork: https://github.com/barkleesanders/shannon

## Fork notes

This fork keeps Shannon's core behavior intact: drive a real interactive Claude Code session through tmux and emit CLI/SDK-friendly output.

What was fixed and why:

- Package metadata now consistently points at `@dexh/shannon`, `@dexh/shannon-agent-sdk`, and `dexhorthy/shannon`, so installs and repository links resolve to the actual project identity.
- `shannon --help` now prints normal Commander help and exits successfully, so the CLI has a working smoke path before a prompt is executed.
- CI installs `tmux`, runs the fake-Claude tmux regression, and supports manual dispatch, so the runtime path Shannon depends on is covered by automation.
- `@anthropic-ai/claude-agent-sdk` is pinned and overridden at `0.2.141`, with Bun release-age exceptions for the Anthropic packages, so installs stay reproducible while the audit advisory is cleared.
- The hello-world example uses `link:../..` for the local package, so example installs test the fork without recursively copying the repository into `node_modules`.

These fixes matter because Shannon's reliability depends on matching the real local Claude/tmux runtime, not only package-level imports.

```mermaid
flowchart LR
    User([Your code / shell])
    subgraph Shannon["@dexh/shannon"]
      SDK["SDK<br/>query()"]
      CLI["CLI<br/>shannon"]
    end
    subgraph Host["Local host"]
      Tmux["tmux session"]
      Claude["claude (interactive)"]
      Transcript[("~/.claude/projects<br/>transcript JSONL")]
    end
    Anthropic[["Anthropic API"]]

    User -->|"prompt"| SDK
    User -->|"prompt"| CLI
    SDK -->|"spawns"| CLI
    CLI -->|"sends keys"| Tmux
    Tmux --> Claude
    Claude <-->|"HTTPS"| Anthropic
    Claude -->|"appends"| Transcript
    CLI -->|"tails"| Transcript
    CLI -->|"stream-json / json / text"| User
```

The dashed-style boundary: Shannon never calls the Anthropic API directly — it drives a real `claude` session and reads its on-disk transcript. `claude -p` is not used internally.

## Requirements

- [Bun](https://bun.sh)
- `claude` on `PATH`
- `tmux` on `PATH`
- A working Claude Code login

## CLI

Run without installing:

```sh
npx @dexh/shannon -p "Reply with exactly: hello" --output-format=stream-json --verbose
```

Or install globally:

```sh
npm install -g @dexh/shannon
shannon -p "Reply with exactly: hello" --output-format=stream-json --verbose
```

Output formats: `stream-json` (JSONL), `json` (single array), `text` (final result text).

## SDK

```sh
npm install @dexh/shannon
```

```ts
import { query } from "@dexh/shannon";

for await (const message of query({
  prompt: "Reply with exactly: hello",
  options: { outputFormat: "stream-json", verbose: true },
})) {
  console.log(JSON.stringify(message));
}
```

Async input is also accepted for finite user-message streams:

```ts
import { query, type ShannonUserMessage } from "@dexh/shannon";

async function* messages(): AsyncIterable<ShannonUserMessage> {
  yield {
    type: "user",
    message: {
      role: "user",
      content: [{ type: "text", text: "Reply with exactly: hello" }],
    },
    parent_tool_use_id: null,
    session_id: "",
  };
}

for await (const message of query({ prompt: messages() })) {
  console.log(JSON.stringify(message));
}
```

Pass an `AbortController` in options to terminate the underlying Shannon subprocess.

## Agent SDK facade

`@dexh/shannon-agent-sdk` is a Claude Agent SDK-compatible facade that re-exports Shannon's SDK surface. Full parity is a work in progress (see `GOAL_PROGRESS.md`).

```sh
npm install @dexh/shannon-agent-sdk
```

```ts
import { query } from "@dexh/shannon-agent-sdk";

for await (const message of query({
  prompt: "Reply with exactly: hello",
  options: { outputFormat: "stream-json", verbose: true },
})) {
  console.log(JSON.stringify(message));
}
```

## Development

```sh
bun install
bun test
bun run typecheck
bun ./index.ts -p "hello" --output-format=stream-json --verbose
```
