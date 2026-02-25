# WebMCP Demo — CustomInk-Style Ordering Flow

A single-file demo of the [WebMCP browser API](https://github.com/webmachinelearning/webmcp) — a proposed W3C standard (Microsoft + Google, 2025) that lets web pages expose tools directly to AI agents via the browser, with no server or transport layer required.

Styled with [Pigment](https://pigment.customink.com/) design tokens to match the CustomInk look and feel.

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

## Connecting Claude Code via MCP (Advanced)

You can connect Claude Code directly to the page using the `@mcp-b/chrome-devtools-mcp` server.

### 1. Start Chrome with remote debugging

Add this alias to your shell config:

```bash
alias chrome-debug='killall "Google Chrome Canary" 2>/dev/null; sleep 1; \
  "/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary" \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-canary-debug &'
```

Then run `chrome-debug` to launch Chrome with debugging enabled.

### 2. Configure the MCP server

From your `webmcp/` project directory:

```bash
claude mcp add chrome-devtools -- npx @mcp-b/chrome-devtools-mcp@latest --browserUrl http://127.0.0.1:9222
```

### 3. Navigate to the demo and use it

Once Claude Code is open in this project, the `chrome-devtools` MCP server gives Claude access to tools like `list_pages`, `navigate_page`, and `evaluate_script` to drive the browser directly.

> **Note:** The `list_webmcp_tools` / `call_webmcp_tool` path requires the `@mcp-b/global` package working as an ES module with a tab server transport. The extension approach above is more reliable for testing.

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

## Design

Styled using [Pigment](https://pigment.customink.com/) design tokens (light theme) hardcoded as CSS custom properties:

- **Brand orange** `#fa3c00` — `semantics.light.partnership.background.bold`
- **Page background** `#fafafa` — `semantics.light.neutral.background.pageBG`
- **Header** `#363636` — `semantics.light.neutral.background.toolbar`
- **Typography** — Source Sans 3 (CustomInk brand font)

---

## References

- [WebMCP Proposal](https://github.com/webmachinelearning/webmcp/blob/main/docs/proposal.md)
- [WebMCP README](https://github.com/webmachinelearning/webmcp)
- [Model Context Tool Inspector](https://github.com/beaufortfrancois/model-context-tool-inspector)
- [Pigment Design System](https://pigment.customink.com/)
