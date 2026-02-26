# Learning Mode — Step Debugger: Implementation Plan

## Approach 3: Developer-Tools-Inspired Step Debugger

### UX Design Description

When Learning Mode is toggled ON (via a switch in the page header), a DevTools-style panel slides up from the bottom of the viewport. The main layout still shows beneath it. Every tool call is intercepted before it executes — the panel snaps to front and the page shows a subtle frozen overlay indicator. The developer reads the full request details, then clicks "Step" to let execution proceed and see the response.

The panel mimics Chrome DevTools aesthetics: compact height (~340px), tabbed interface, monospace JSON panes, muted chrome-gray tones. It feels like a Network > Request/Response inspector crossed with a debugger breakpoint.

**"Frozen" state** (before Step): Panel shows tool name as a breakpoint header, input params as formatted JSON, the inputSchema, and a plain-English explainer. The Step button is prominent (green). A "Run All" button skips stepping for the rest of the session.

**"Resumed" state** (after Step): The Response tab auto-activates and shows the result JSON. A brief "Completed" badge appears, then the panel waits for the next tool call.

**Between calls**: Panel collapses back to a minimal tab strip (32px) or hides entirely depending on whether it was previously open.

---

## HTML Components

### 1. Learning Mode Toggle in Header
- Location: inside `.header-chips` area, right side of `#header`
- Markup: `<label class="learning-toggle">` with a `<input type="checkbox">` styled as a pill switch
- Label: "Learning Mode" in Source Sans 3, white text
- Color: amber accent when active (`--accent-amber`)

### 2. Step Debugger Panel (`#step-debugger`)
- Fixed position at bottom of viewport, full width
- Height: 0 when closed, 340px when open (CSS transition)
- Structure:
  ```
  #step-debugger
    .sdb-topbar          ← tool name + status badge + Step / Run All buttons
    .sdb-tab-bar         ← Request | Schema | Explainer | Response tabs
    .sdb-content
      .sdb-pane[data-tab="request"]    ← pretty-printed input JSON
      .sdb-pane[data-tab="schema"]     ← inputSchema JSON
      .sdb-pane[data-tab="explainer"]  ← plain-English explanation per tool
      .sdb-pane[data-tab="response"]   ← result JSON (populated after Step)
  ```

### 3. Page Freeze Overlay (`#sdb-freeze-overlay`)
- Thin top border on `#main-layout` glowing amber — subtle, not blocking
- Or: a semi-transparent top strip with "Paused at breakpoint" text

---

## CSS Strategy

- Panel uses CSS variables from the existing design system
- Panel background: `var(--bg-surface)` with `border-top: 2px solid var(--accent-amber)`
- JSON panes: `background: var(--bg-raised)`, `font-family: var(--font-mono)`, `font-size: 12px`
- Tab bar: `background: var(--bg-raised)`, active tab gets `border-bottom: 2px solid var(--accent-amber)`
- Step button: `background: var(--accent-green)`, white text
- Run All button: ghost style with `var(--border)`
- Toggle switch: CSS-only with `:checked` state

---

## Execution Interception Strategy

Since WebMCP `execute()` functions are **synchronous** by the API contract (they return a value synchronously), we cannot use `async/await` in the execute body itself. Instead:

1. Each tool's `execute()` function will call `window.stepDebugger.intercept(toolName, params, schema, executeCore)` before running the actual logic.
2. `intercept()` **synchronously shows the panel** and blocks by spinning in a **busy-wait loop** using `Atomics.wait()` on a `SharedArrayBuffer` (if available) — this is the only reliable way to pause a synchronous function on the main thread.
3. **Fallback without SharedArrayBuffer**: If `Atomics.wait()` is unavailable on the main thread (Chrome restricts it), we use a different approach: the `execute()` wrapper returns a `Promise` to the WebMCP runtime. We check if the browser's `modelContext` can handle async execute — if so, we use `async execute()` returning a Promise that resolves after the user clicks Step.
4. **Practical approach**: Since Chrome's WebMCP implementation and the `@mcp-b/global` polyfill both ultimately call `execute()`, and we need to pause, we will:
   - Use `async execute()` that returns a Promise
   - The Promise is created around a user-gated `resolve()` call
   - The Step button calls `window.stepDebugger.resume()`
   - This resolves the pending Promise and returns the result to the agent

This means the execute function becomes **async** — if the WebMCP runtime supports it (the polyfill does, native WebMCP likely does too since it's a browser-mediated call).

---

## Tool Explainer Content (per tool)

### searchProducts
"The agent is searching the product catalog. `searchProducts` takes optional `query` (text search) and `category` (filter) params and returns an array of matching products with their IDs, colors, sizes, and decoration options. The agent uses this to discover what's available before placing an order."

### configureProduct
"The agent is validating a specific product + color + size combination. `configureProduct` checks that the chosen color and size exist for this product, and returns the starting unit price along with available decoration methods. Think of it as the agent 'selecting' a product variant."

### getQuote
"The agent is calculating a price quote. `getQuote` applies quantity-based pricing tiers — the more you order, the lower the unit price. It also adds the decoration cost (screen printing, embroidery, etc.) and returns a full breakdown. No cart changes happen here."

### addToCart
"The agent is adding an item to the cart. `addToCart` runs the same validation as `getQuote`, then appends the item to the cart array and triggers a live UI update via `renderCart()`. After this call, you'll see the item appear in the Cart sidebar."

### getCart
"The agent is reading the current cart state. `getCart` is read-only — it returns the full cart array, total item count, and order total. The agent typically calls this to summarize what has been added so far or to verify the cart before presenting results."

---

## What Changed Tab Content

After Step is clicked and result is available, the "Response" tab shows the raw JSON. A supplementary "What Changed" section within the Response tab highlights:
- For `addToCart`: cart item count delta, new total
- For `searchProducts`: number of products returned
- For `getQuote`: the price per unit and subtotal
- For others: general key→value highlights from the result

---

## Implementation Order

1. **Write PLAN.md** (this file) — done
2. **Add CSS** for the debugger panel, toggle, and freeze overlay (inside the existing `<style>` block)
3. **Add HTML** for the toggle in the header and the `#step-debugger` panel (before `</body>`)
4. **Add JavaScript**:
   a. `window.stepDebugger` object with state variables
   b. `stepDebugger.intercept(toolName, params, schema, explainerKey)` — shows panel, returns Promise
   c. `stepDebugger.resume()` — resolves pending Promise
   d. `stepDebugger.runAll()` — sets bypass flag
   e. Tab switching logic
   f. Learning mode toggle event handler
   g. Per-tool explainer map
5. **Wrap each tool's execute()** to be `async` and call `stepDebugger.intercept()` before the core logic
6. **Test** that learning mode OFF is identical to original behavior

---

## Key Design Decisions

- **Async execute()**: The native WebMCP spec shows synchronous execute, but the polyfill and likely browser implementation both handle Promises. Async execute is the only viable pattern without blocking the main thread via Atomics (which isn't available on the main thread in Chrome).
- **No extra dependencies**: Pure vanilla JS/CSS.
- **Panel height**: Fixed 340px with `overflow: hidden` on content area and `overflow-y: auto` on JSON panes for scrollability.
- **State machine**: `idle` | `paused` | `stepping` | `complete` — drives button states and tab availability.
- **Distinctive vs alternatives**: Unlike a simple log overlay or annotation system, this is a true execution breakpoint — the agent actually waits for the user to click Step before the tool returns. The agent "freezes" mid-session.
