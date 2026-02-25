# Why WebMCP? Pages With vs. Without

A regular web page and a WebMCP-enabled page can look identical to a human user. The difference is entirely in how an AI agent interacts with it.

---

## Without WebMCP — Browser Automation

Without WebMCP, an agent has two options:

**1. Vision + DOM scraping**

The agent takes a screenshot, reads the layout visually, figures out where the "Add to Cart" button is, and clicks it.

```
Agent                         Browser
  │                              │
  │── "take a screenshot" ──────>│
  │<── [image of the page] ──────│
  │                              │
  │  (agent interprets pixels)   │
  │                              │
  │── "click at x:412, y:830" ──>│
  │<── [new screenshot] ─────────│
  │                              │
  │  (did it work? check DOM)    │
  │── "read text of .cart" ─────>│
  │<── "1 item added" ───────────│
```

**2. Injecting JavaScript via DevTools**

The agent uses a debugging protocol (like CDP) to evaluate arbitrary JS in the page context.

```js
// Agent guesses at internal variable names and function shapes
await page.evaluate(() => {
  addToCart("gildan-64000", "blue", "M", 12);
});
```

**Problems with both approaches:**

- The agent must understand the UI before it can act — requires vision or DOM parsing
- Internal function names and shapes are undocumented; the agent has to infer them
- A UI redesign breaks the automation completely
- No structured error responses — the agent has to parse text or check side effects
- The page has no control over what the agent does; it's just being puppeted
- Slow: multiple round-trips per action (screenshot → interpret → click → verify)

---

## With WebMCP — Structured Tool Calls

The page explicitly declares what it can do and how to do it. The browser acts as a secure intermediary — the agent never touches the DOM directly.

```
Agent                    Browser (mediator)           Page
  │                            │                        │
  │── tools/list ─────────────>│                        │
  │                            │── read modelContext ──>│
  │                            │<── [5 tool schemas] ───│
  │<── [5 tool schemas] ────────│                        │
  │                            │                        │
  │── call addToCart({         │                        │
  │     productId: "gildan",   │                        │
  │     color: "Royal Blue",   │                        │
  │     size: "M", qty: 12     │                        │
  │   }) ──────────────────────>│                        │
  │                            │── execute(params) ────>│
  │                            │                        │  (page logic runs,
  │                            │                        │   cart updates,
  │                            │                        │   UI re-renders)
  │                            │<── { success: true,    │
  │                            │     orderTotal: "$125" }│
  │<── { success: true, ... } ──│                        │
```

**What this means in code on the page:**

```js
navigator.modelContext.registerTool({
  name: "addToCart",
  description: "Add a configured product to the shopping cart.",
  inputSchema: {
    type: "object",
    properties: {
      productId:  { type: "string",  description: "Product ID from searchProducts." },
      quantity:   { type: "number",  description: "Number of items. Minimum 12." },
      color:      { type: "string",  description: "Exact color name." },
      size:       { type: "string",  description: "Exact size." },
      decoration: { type: "string",  enum: ["screen-print", "embroidery", "dtg", "heat-transfer"] }
    },
    required: ["productId", "quantity", "color", "size", "decoration"]
  },
  execute({ productId, quantity, color, size, decoration }, agent) {
    // validate, compute price, update cart state, re-render UI
    cart.push(newItem);
    renderCart(cart);
    return { content: [{ type: "text", text: JSON.stringify({ success: true, orderTotal }) }] };
  }
});
```

The agent receives a schema describing exactly what parameters exist, what types they are, and what they mean — before ever making a call. It doesn't need to look at the UI at all.

---

## Side-by-Side Comparison

| | Without WebMCP | With WebMCP |
|---|---|---|
| **How agent discovers capabilities** | Reads DOM, takes screenshots, guesses | `navigator.modelContext` — structured JSON schemas |
| **How agent calls a function** | Clicks UI elements or injects JS | Typed tool call via browser API |
| **Page's role** | Passive — being puppeted | Active — declares what it exposes |
| **Error handling** | Agent scrapes text to detect failure | Structured error returned from `execute()` |
| **Resilience to UI changes** | Breaks on any redesign | Tools are independent of UI layout |
| **Speed** | Multiple round-trips per action | Single tool call |
| **Security** | Agent has full DOM access | Browser mediates all calls; page controls scope |
| **Typing** | None | Full JSON Schema on inputs |

---

## The Mental Model

Think of it like the difference between a **website** and an **API**.

A website exposes information and actions through a visual UI designed for humans. A scraper or automation tool can interact with it, but only by pretending to be a human — it's fragile and indirect.

An API exposes the same capabilities as structured, typed, documented endpoints. Callers don't need to know anything about the UI.

WebMCP makes a web page act like an API — but one that's implemented entirely in client-side JavaScript and mediated by the browser. The page stays a normal web page for humans. For agents, it's also a structured set of callable tools.

---

## What Stays the Same

The page looks and works exactly the same for a human visitor. In this demo:

- The product grid, cart sidebar, and tool log are all visible and functional in any browser
- A human can read the page to understand what's available
- The WebMCP tools are an additive layer — they don't replace the UI, they complement it

The only requirement for the agent integration to work is a browser that supports `navigator.modelContext` (currently Chrome Canary with the WebMCP flag enabled).
