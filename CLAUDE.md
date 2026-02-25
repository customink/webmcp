# WebMCP Demo — Claude Code Context

## What This Is

A single-file WebMCP demo (`webmcp-demo.html`) simulating a CustomInk-style product ordering flow. WebMCP is a proposed W3C browser API (Microsoft + Google, 2025) where web pages expose tools to AI agents via `navigator.modelContext`. The browser mediates all tool calls — no transport, no JSON-RPC, no server process.

## Key Files

| File | Purpose |
|------|---------|
| `webmcp-demo.html` | Single-file demo — all HTML, CSS, JS in one file (~1650 lines) |
| `model-context-tool-inspector/` | Chrome extension for manually testing WebMCP tools |
| `README.md` | Setup guide for humans |

## WebMCP API (use exactly — do not substitute Anthropic MCP patterns)

```js
// Feature detection
if ("modelContext" in window.navigator) { ... }

// Register a tool
navigator.modelContext.registerTool({
  name: "toolName",
  description: "What this tool does",
  inputSchema: {
    type: "object",
    properties: {
      param: { type: "string", description: "..." }
    },
    required: ["param"]
  },
  execute(params, agent) {
    // runs synchronously on the main thread
    return { content: [{ type: "text", text: "result" }] };
  }
});

// Remove a tool
navigator.modelContext.unregisterTool("toolName");

// Replace all tools at once
navigator.modelContext.provideContext({ tools: [...] });
```

**This is NOT Anthropic MCP.** No `server.tool()`, no stdio, no HTTP/SSE.

## Tools in the Demo

| Tool | Key Params | Notes |
|------|-----------|-------|
| `searchProducts` | `query`, `category` | Returns product catalog matches |
| `configureProduct` | `productId`, `color`, `size` | Validates availability |
| `getQuote` | `productId`, `quantity`, `color`, `size`, `decoration` | Min qty 12 |
| `addToCart` | same as getQuote | Updates UI live via `renderCart()` |
| `getCart` | none | Read-only cart state |

## Testing

**Recommended:** Model Context Tool Inspector Chrome extension (`model-context-tool-inspector/`)
- Load as unpacked extension from `chrome://extensions`
- Open `webmcp-demo.html` in Chrome Canary with WebMCP flag enabled
- Flag location: `chrome://flags` → search "model context"

**Chrome MCP bridge:** `@mcp-b/chrome-devtools-mcp` is configured for this project.
- Requires Chrome running with `--remote-debugging-port=9222`
- Use the `chrome-debug` shell alias to launch Chrome correctly
- Provides `list_pages`, `navigate_page`, `evaluate_script` for browser automation
- `list_webmcp_tools` / `call_webmcp_tool` require `@mcp-b/global` ESM setup (complex — prefer the extension for tool testing)

## Design System

Pigment light theme tokens are hardcoded as CSS custom properties in `webmcp-demo.html`. Do not add dark-theme colors.

| CSS Variable | Value | Pigment Token |
|---|---|---|
| `--accent-amber` | `#fa3c00` | `semantics.light.partnership.background.bold` |
| `--accent-amber-hover` | `#e03600` | `semantics.light.partnership.background.boldAccessible` |
| `--accent-blue` | `#1e39d2` | `semantics.light.interactive.background.bold` |
| `--accent-green` | `#198038` | `semantics.light.feedback.successBold` |
| `--accent-red` | `#da1e28` | `semantics.light.feedback.dangerBold` |
| `--bg-base` | `#fafafa` | `semantics.light.neutral.background.pageBG` |
| `--bg-surface` | `#ffffff` | `semantics.light.neutral.background.primary` |
| `--bg-raised` | `#f5f5f5` | `semantics.light.neutral.background.subtleOpaque` |
| `--border` | `#0000001c` | `semantics.light.neutral.border.default` |
| `--header-bg` | `#363636` | `semantics.light.neutral.background.toolbar` |
| `--text-primary` | `#000000db` | `semantics.light.neutral.text.primary` |
| `--text-secondary` | `#00000091` | `semantics.light.neutral.text.secondary` |
| Font | Source Sans 3 | CustomInk brand font (Google Fonts) |

## Architecture Notes

- **No build step** — vanilla HTML/CSS/JS, opens directly in browser
- **No framework** — Tailwind CDN for utilities, but most styles are custom CSS variables
- **Tool log** — `window.logToolCall(name, params, result, isError, durationMs)` appends entries to `#tool-log`
- **Cart** — `window.renderCart(cartArray)` updates the sidebar UI
- **Products** — `window.renderProducts(productsArray)` populates `#product-grid`
- **Agent indicator** — `window.setAgentActive(bool)` toggles the header status pill
