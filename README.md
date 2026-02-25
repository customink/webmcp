# WebMCP Demo — CustomInk-Style Ordering Flow

A single-file demo of the [WebMCP browser API](https://github.com/webmachinelearning/webmcp) — a proposed W3C standard (Microsoft + Google, 2025) that lets web pages expose tools directly to AI agents via the browser, with no server or transport layer required.

Styled with [Pigment](https://pigment.customink.com/) design tokens to match the CustomInk look and feel.

> **A note on production readiness:** Since WebMCP is still a proposed standard (Microsoft + Google, 2025), the near-term value here is conceptual — understanding where browser-native agent APIs are heading. The real production path for CustomInk agent work right now is server-side MCP via the GraphQL gateway. WebMCP is something to watch, not build on yet.

---

## What Is WebMCP?

WebMCP is a browser-native JS API where pages register tools as JavaScript functions. When an AI agent calls a tool, the browser mediates the call — the agent never touches the DOM directly.

> **New to WebMCP?** See [why-webmcp.md](./why-webmcp.md) for a detailed comparison of how agents interact with a page with vs. without WebMCP — including diagrams and a side-by-side breakdown.

```js
navigator.modelContext.registerTool({
  name: "addToCart",
  description: "Add a product to the cart",
  inputSchema: { /* JSON Schema */ },
  execute(params, agent) {
    // runs in the page
    return { content: [{ type: "text", text: "Added." }] };
  }
});
```

This is **not** Anthropic MCP. There is no stdio transport, no JSON-RPC, no server process — the browser is the intermediary.

---

## Tools Registered

| Tool | Description |
|------|-------------|
| `searchProducts` | Search the catalog by keyword and/or category |
| `configureProduct` | Validate a product + color + size combination |
| `getQuote` | Calculate pricing with quantity tiers and decoration costs |
| `addToCart` | Add a validated item to the cart (updates UI live) |
| `getCart` | Read current cart contents and order total |

---

## Prerequisites

- **Chrome Canary** — [download here](https://www.google.com/chrome/canary/)
- **WebMCP flag enabled** in Chrome Canary:
  1. Open `chrome://flags`
  2. Search for `model context` or `WebMCP`
  3. Enable the flag
  4. Click **Relaunch**

---

## Running the Demo

No build step required — it's a single HTML file.

```bash
# Clone the repo
git clone <repo-url>
cd webmcp

# Open directly in Chrome Canary (after enabling the flag above)
open -a "Google Chrome Canary" webmcp-demo.html
```

Or use **File → Open File** in Chrome Canary to open `webmcp-demo.html`.

---

## Testing with the Model Context Tool Inspector

The `model-context-tool-inspector/` directory contains a Chrome extension for manually calling WebMCP tools.

### Load the extension

1. Open `chrome://extensions` in Chrome Canary
2. Enable **Developer mode** (top right toggle)
3. Click **Load unpacked**
4. Select the `model-context-tool-inspector/` folder

### Build the extension (first time only)

```bash
cd model-context-tool-inspector
npm install   # generates js-genai.js via postinstall script
```

### Use it

1. Open `webmcp-demo.html` in Chrome Canary
2. Click the **Model Context Tool Inspector** icon in your toolbar (or open the side panel)
3. The 5 registered tools will appear
4. Select a tool, fill in arguments, and execute

**Example — add a shirt to the cart:**

```json
Tool: addToCart
{
  "productId": "gildan-64000",
  "quantity": 12,
  "color": "Royal Blue",
  "size": "M",
  "decoration": "screen-print"
}
```

The Tool Activity Log on the page updates in real time as tools are called.

---

## Connecting Claude Code via MCP (Coming Soon)

> **Note:** Getting Claude Code to call WebMCP tools directly through the Chrome DevTools bridge is a work in progress. The setup is non-trivial — skip this section for now and use the Model Context Tool Inspector extension above instead.

The goal is to connect Claude Code to the page using `@mcp-b/chrome-devtools-mcp`, which would let Claude drive the demo through `list_pages`, `navigate_page`, and `call_webmcp_tool`. We'll update this section once the setup is reliable.

---

## Project Structure

```
webmcp/
├── webmcp-demo.html              # Single-file demo — all HTML, CSS, JS
└── model-context-tool-inspector/ # Chrome extension for manual tool testing
    ├── manifest.json
    ├── background.js
    ├── content.js
    ├── sidebar.html / sidebar.js
    ├── styles.css
    └── package.json              # npm install generates js-genai.js
```

---

## Further Reading

- [Why WebMCP?](./why-webmcp.md) — Pages with vs. without WebMCP: diagrams, comparison table, and mental model
- [Custom Ink Integration Guide](./customink-webmcp-integration.md) — What it would take to make the real Custom Ink t-shirts page WebMCP-enabled

---

## References

- [WebMCP Proposal](https://github.com/webmachinelearning/webmcp/blob/main/docs/proposal.md)
- [WebMCP README](https://github.com/webmachinelearning/webmcp)
- [Model Context Tool Inspector](https://github.com/beaufortfrancois/model-context-tool-inspector)
- [Pigment Design System](https://pigment.customink.com/)
