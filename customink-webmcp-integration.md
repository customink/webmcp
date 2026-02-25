# Making the Custom Ink T-Shirts Page WebMCP-Enabled

**Target page:** https://www.customink.com/products/t-shirts/short-sleeve-t-shirts/16

This document outlines the engineering work required to expose the real Custom Ink product listing page as a set of WebMCP tools — allowing AI agents to search, filter, configure, and quote products without touching the DOM.

---

## What the Page Does Today

The `/products/t-shirts/short-sleeve-t-shirts/16` page is a React-rendered product listing. It shows **283 short sleeve t-shirt products** and lets humans:

- Browse a grid with badges (Best Seller, Customer Fave, Staff Pick), ratings, per-unit pricing, and color swatches
- Filter by a rich set of facets (full list below)
- Sort by Recommended, Price, Rating, etc.
- Click into a product to configure color, sizes, and decoration method
- Add configured items to a cart and request a quote

**Filters available on the page** (sourced from Algolia facets):

| Filter | Options |
|--------|---------|
| Category | Short Sleeve, Long Sleeve, Tri-Blend, Performance, Women's, Kids, Heavyweight, Tie-Dye, Tank Tops, No Minimum, Made in USA, Tall, Canada, NEW |
| No Minimum | Toggle |
| Color Family | Black, Blue, White, Gray, Green, Red, Pink, Purple, Yellow, Orange, Brown, Heathered, Camo, Tie Dye |
| Rush Delivery | 3 days (Super Rush), 1 week, 10 days, 12 days |
| Brands | A4, Adidas, AllPro, Allmade, American Apparel, Augusta, Badger, Bella+Canvas, Gildan, Hanes, Next Level, … |
| Material | 100% Cotton, Cotton/Poly, 100% Polyester, Recycled, Tri-Blend, Performance Blend, Poly/Spandex |
| Type | Unisex, Women's, Youth, Tall |
| Sizes | 2XS–4XL, LT/XLT tall sizes |
| Style | Performance, Pocket, Workwear, Ringer, Henley, Raglan |
| Neckline | Crew Neck, V-Neck, Scoop Neck |
| Product Features | Tear Away Tag, Eco-friendly, Moisture-Wicking, Tagless, UV Protection, Pocket, Garment Dyed |
| Price Tier | $ / $$ / $$$ |
| Fit | Standard, Relaxed, Semi-Fitted, Slim |
| Decoration | Printed, Embroidered |
| Delivery Options | Ships to multiple addresses (Yes/No) |

An AI agent today would need to screenshot the page, parse the visual layout, and click through UI elements to use any of these filters. WebMCP replaces all of that with direct tool calls.

---

## Tools to Expose

These tools map to what a human does on the page. Each one would be registered via `navigator.modelContext.registerTool()`.

### `searchProducts`

Search the product catalog by keyword, category, or attribute.

```js
{
  name: "searchProducts",
  description: "Search Custom Ink's t-shirt catalog. Returns matching products with IDs, names, prices, ratings, and available options.",
  inputSchema: {
    type: "object",
    properties: {
      query:          { type: "string",  description: "Keyword search (e.g. 'soft', 'moisture wicking', 'Bella+Canvas')" },
      category:       { type: "string",  description: "Subcategory (e.g. 'short-sleeve-t-shirts', 'performance-shirts', 'womens-t-shirts')" },
      brand:          { type: "array",   items: { type: "string" }, description: "Filter by brand(s): Gildan, Bella+Canvas, Hanes, Next Level, American Apparel, etc." },
      colorFamily:    { type: "array",   items: { type: "string" }, description: "Filter by color family: Black, Blue, White, Gray, Green, Red, Pink, Purple, Yellow, Orange, Brown, Heathered, Camo, Tie Dye" },
      material:       { type: "array",   items: { type: "string" }, description: "Filter by material: '100% Cotton', 'Cotton/Poly', '100% Polyester', 'Recycled Material', 'Tri-Blend'" },
      type:           { type: "string",  enum: ["Unisex", "Women's", "Youth", "Tall"], description: "Garment type/fit audience" },
      sizes:          { type: "array",   items: { type: "string" }, description: "Filter to products available in these sizes: XS, S, M, L, XL, 2XL, 3XL, 4XL, LT, XLT" },
      style:          { type: "string",  enum: ["Performance", "Pocket", "Workwear", "Ringer", "Henley", "Raglan"], description: "Shirt style" },
      neckline:       { type: "string",  enum: ["Crew Neck", "V-Neck", "Scoop Neck"], description: "Neckline style" },
      fit:            { type: "string",  enum: ["Standard Fit", "Relaxed Fit", "Semi-Fitted", "Slim Fit"], description: "Fit" },
      decoration:     { type: "string",  enum: ["Printed", "Embroidered"], description: "Decoration method" },
      features:       { type: "array",   items: { type: "string" }, description: "Product features: 'Eco-friendly', 'Moisture-Wicking', 'Tagless', 'Tear Away Tag', 'UV Protection', 'Garment Dyed'" },
      priceTier:      { type: "string",  enum: ["$", "$$", "$$$"], description: "Price tier ($ = budget, $$$ = premium)" },
      rushDelivery:   { type: "string",  enum: ["3 days", "1 week", "10 days", "12 days"], description: "Filter to products available by this deadline" },
      noMinimum:      { type: "boolean", description: "If true, only return products with no minimum order quantity" },
      multipleAddresses: { type: "boolean", description: "If true, only return products that can ship to multiple addresses" },
      sortBy:         { type: "string",  enum: ["Recommended", "Price", "Rating"], description: "Sort order. Default: Recommended" },
      limit:          { type: "integer", description: "Max results to return. Default 10." }
    }
  }
}
```

**Data source:** Products are served by **Algolia** (`react-instantsearch` 7.13.1). The tool queries the same Algolia index, passing filters as facet refinements. All filter values above correspond directly to Algolia facets already configured on the index.

---

### `getProductDetails`

Get full details for a specific product: available colors, sizes, decoration methods, and pricing tiers.

```js
{
  name: "getProductDetails",
  description: "Get full details for a product including all color/size options, decoration methods, and per-quantity pricing.",
  inputSchema: {
    type: "object",
    properties: {
      productId: { type: "string", description: "Product ID from searchProducts results" }
    },
    required: ["productId"]
  }
}
```

**Data source:** Product detail data is typically fetched lazily when a user hovers or clicks a product card. The tool would trigger the same fetch (or read from cache if already loaded).

---

### `getQuote`

Calculate the price for a configured product at a given quantity.

```js
{
  name: "getQuote",
  description: "Get a price quote for a product configuration. Returns per-unit price, setup fees, and order total.",
  inputSchema: {
    type: "object",
    properties: {
      productId:  { type: "string",  description: "Product ID" },
      quantity:   { type: "integer", description: "Number of units. Minimum order quantities vary by product." },
      color:      { type: "string",  description: "Color name exactly as returned by getProductDetails" },
      sizes:      {
        type: "array",
        items: {
          type: "object",
          properties: {
            size:     { type: "string",  description: "Size (XS, S, M, L, XL, 2XL, etc.)" },
            quantity: { type: "integer" }
          },
          required: ["size", "quantity"]
        },
        description: "Size breakdown. Total must match top-level quantity."
      },
      decoration: {
        type: "string",
        enum: ["screen-print", "embroidery", "dtg", "heat-transfer"],
        description: "Decoration method"
      },
      locations:  { type: "integer", description: "Number of print locations. Default 1." }
    },
    required: ["productId", "quantity", "color", "sizes", "decoration"]
  }
}
```

**Data source:** Custom Ink has a pricing API that takes product + quantity + decoration parameters. The tool wraps that same call.

---

### `addToCart`

Add a fully configured product to the cart and redirect to the design/checkout flow if needed.

```js
{
  name: "addToCart",
  description: "Add a configured product to the cart. Returns cart summary and a URL to continue to design/checkout.",
  inputSchema: {
    type: "object",
    properties: {
      productId:  { type: "string" },
      quantity:   { type: "integer" },
      color:      { type: "string" },
      sizes:      { type: "array", items: { type: "object" } },
      decoration: { type: "string" }
    },
    required: ["productId", "quantity", "color", "sizes", "decoration"]
  }
}
```

**Note:** This tool may require `agent.requestUserInteraction()` — the WebMCP API supports asking for user confirmation before actions with side effects. Agents cannot silently modify cart state without the page opting into that.

---

### `getFilters`

Return the available filter options and product counts so an agent knows what's possible before making a `searchProducts` call.

```js
{
  name: "getFilters",
  description: "Return all available filters for the current product listing with option counts. Useful before searchProducts to understand what filtering is possible.",
  inputSchema: { type: "object", properties: {} }
}
```

The response would include the full facet tree: brands, color families, materials, sizes, styles, necklines, fit, features, price tiers, rush delivery options, and decoration methods — each with the count of matching products.

**Data source:** Read directly from the Algolia facets already returned with the page's initial search response, via the `react-instantsearch` component state. No additional API call needed.

---

## Where the Code Lives

Custom Ink's frontend is a React app (with Rails serving the initial HTML). The WebMCP tool registration would go in:

### Option A — Page-level React component

Register tools in the top-level page component's `useEffect`, unregister on unmount:

```jsx
// In ShortSleeveListingPage.tsx (or equivalent)
useEffect(() => {
  if (!("modelContext" in window.navigator)) return;

  navigator.modelContext.registerTool(searchProductsTool);
  navigator.modelContext.registerTool(getProductDetailsTool);
  navigator.modelContext.registerTool(getQuoteTool);
  navigator.modelContext.registerTool(addToCartTool);
  navigator.modelContext.registerTool(getFiltersTool);

  return () => {
    navigator.modelContext.unregisterTool("searchProducts");
    navigator.modelContext.unregisterTool("getProductDetails");
    navigator.modelContext.unregisterTool("getQuote");
    navigator.modelContext.unregisterTool("addToCart");
    navigator.modelContext.unregisterTool("getFilters");
  };
}, []);
```

Each tool's `execute` function closes over component state or calls the same service functions the UI uses.

---

### Option B — Shared hook

Extract into a reusable hook that any product listing page can use:

```js
// useWebMCPProductTools.ts
export function useWebMCPProductTools({ products, filters, cartService, pricingService }) {
  useEffect(() => {
    if (!("modelContext" in window.navigator)) return;
    // register all 5 tools...
    return () => { /* unregister */ };
  }, [products, filters]);
}
```

This approach makes it easy to add WebMCP to other product category pages (`/products/hoodies`, `/products/polos`, etc.) with no duplication.

---

### Option C — Service worker (persistent across navigations)

The WebMCP spec also supports tool registration from a service worker via `self.agent.provideContext({ tools: [...] })`. This keeps tools available even as the user navigates between pages — useful if the agent needs to maintain context across the listing → PDP → cart flow.

```js
// service-worker.js
self.agent.provideContext({
  tools: [
    searchProductsTool,
    getProductDetailsTool,
    getQuoteTool,
    addToCartTool
  ]
});
```

Service worker tools call back into the page via `fetch` or a shared cache — they don't have direct DOM access.

**Recommendation:** Start with Option A (page-level registration) to ship fast, then evaluate Option C if agent sessions need to span multiple page navigations.

---

## Connecting to Existing Services

The tool `execute` functions should call the same internal services the UI already uses — not re-implement them.

The network activity on the live page reveals the actual data stack:

| Tool | Existing service / API |
|------|----------------------|
| `searchProducts` | **Algolia** — the page uses `react-instantsearch` 7.13.1 with Algolia index `s0rxn4tv6t`. The `searchProducts` tool can query the same Algolia index directly or read from the existing `InstantSearch` state. |
| `getProductDetails` | Product detail API — fetched lazily on product hover/click. The tool triggers the same fetch or reads from React state if already cached. |
| `getQuote` | Pricing API — same endpoint the quote calculator uses. Parameters: productId, quantity, color, sizes, decoration. |
| `addToCart` | **`/api/checkout/cart`** REST endpoint — the live page calls `GET /api/checkout/cart?include_pricing=false&src=ci-cart` to read cart state. Mutations would use `POST` to the same endpoint. |
| `getFilters` | Read from Algolia facets already in the `react-instantsearch` component state — no additional API call needed. Returns 15 filter categories with product counts per option. |

The key principle: **tools are a new interface layer, not a new implementation.** The business logic stays where it is.

---

## What Needs to Be Built vs. What Already Exists

| Concern | Status | Work Required |
|---------|--------|---------------|
| Product data API | Exists | Wrap in tool `execute`, handle auth headers |
| Pricing API | Exists | Wrap in tool `execute` |
| Cart service | Exists | Wrap in tool `execute`, add user confirmation |
| Filter data | Exists in React state | Read from state in `execute` |
| WebMCP feature detection | Not implemented | 5-line check: `if ("modelContext" in navigator)` |
| Tool registration | Not implemented | ~150 lines for all 5 tools + schemas |
| Error handling / structured responses | Not implemented | Standardize `execute` return format |
| User confirmation for cart actions | Not implemented | `agent.requestUserInteraction()` call |
| Testing | Not implemented | Unit tests for `execute` logic; integration test with Tool Inspector extension |

---

## Security Considerations

WebMCP tools run in the page context — the browser mediates calls, but the page still controls what the `execute` function does. A few things to think about:

- **Rate limiting**: Tool calls bypass the normal UI throttling. The tool implementations should apply the same rate limits as the underlying APIs.
- **Cart modification**: Use `agent.requestUserInteraction()` for the `addToCart` tool so the user explicitly approves cart changes.
- **No new attack surface for XSS**: Since `execute` runs in-page JavaScript (not injected from outside), it has the same security posture as any other in-page code.
- **Agent identity**: The `agent` parameter in `execute(params, agent)` provides metadata about which agent is calling. You can log this or gate behavior on it.

---

## Testing the Integration

Once registered, use the **Model Context Tool Inspector** Chrome extension (included in this repo) to call tools manually before connecting a real agent:

1. Enable the WebMCP flag in Chrome Canary (`chrome://flags` → "model context")
2. Load the extension from `model-context-tool-inspector/` (see README)
3. Navigate to the t-shirts page
4. Open the extension — it will list all 5 registered tools
5. Call `searchProducts` with `{ "query": "gildan", "limit": 5 }` and verify the response shape

For automated testing, use the `@mcp-b/chrome-devtools-mcp` server to drive calls from Claude Code (see README for setup).

---

## Estimated Effort

| Phase | Work | Estimate |
|-------|------|----------|
| 1. Tool schemas | Define all 5 `inputSchema` objects | 0.5 day |
| 2. Service wiring | Connect `execute` to existing APIs | 1–2 days |
| 3. Feature detection + registration | Page-level `useEffect` | 0.5 day |
| 4. User confirmation | `addToCart` with `requestUserInteraction` | 0.5 day |
| 5. Error handling | Standardize `execute` return format | 0.5 day |
| 6. Testing | Tool Inspector + integration tests | 1 day |
| **Total** | | **~4–5 days** |

This assumes the existing APIs are well-documented and the React component structure is familiar. The bulk of the work is wiring — not new logic.

---

## What This Unlocks

Once the page is WebMCP-enabled, an AI agent can:

- Take a natural language request ("I need 50 royal blue Gildan tees in a mix of mediums and larges for under $8 each with screen printing") and resolve it to a cart item in a single agentic turn
- Compare multiple products across price, color availability, and decoration options without any DOM interaction
- Maintain cart state across a multi-step conversation without the user having to repeat themselves
- Fail gracefully — if a color is out of stock, the tool returns a structured error the agent can act on, rather than the agent having to parse "out of stock" text from a page element

The page still works exactly the same for human visitors. WebMCP is purely additive.

---

## FAQ

### Are all the changes additive, or do we need to modify existing HTML elements?

**Entirely additive — no HTML changes required.**

WebMCP tools are registered through a JavaScript API (`navigator.modelContext.registerTool()`). They have no relationship to the DOM whatsoever. You don't need to:

- Add `data-*` attributes to any elements
- Add ARIA roles or labels for agent consumption
- Modify any existing component templates
- Change any HTML structure or CSS class names

The tools are pure JavaScript objects that the browser holds in a separate "model context" channel, completely independent of the rendered page. A human visitor sees nothing different. The existing React components, their markup, and their styles are untouched.

The only code changes are:
1. New `useEffect` (or hook) that calls `navigator.modelContext.registerTool()` on mount
2. New tool definition objects (the schemas and `execute` functions) — typically in their own files
3. A one-line feature detection guard: `if (!("modelContext" in window.navigator)) return;`

That's it. The rest of the page is unaffected.
