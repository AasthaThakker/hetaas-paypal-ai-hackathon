# Testing instructions for judges

Everything below works without installing anything. Judges can test the hosted demo, the agent endpoint, or run the project locally.

## Option A: hosted demo (fastest)

1. Open **https://hetaas-demo.pages.dev**. A banner marks the site as the PayPal sandbox demo.
2. **Try the AI concierge (Resi).** Open the chat and ask, for example, "a gift under ₹500 for my sister" or "how do I care for a resin coaster?". Resi answers only from the shop's catalogue and FAQs. Ask something unrelated, such as "write me a poem", and it politely declines.
3. **Build a basket.** Add two or three items from the shop. The **Customization** page (custom.html) lets you design a custom piece and add it to the basket.
4. **Check the delivery charge.** Open the basket and checkout. Enter a name, a 10-digit Indian mobile number, an email, an address, city, state, and a 6-digit PIN code, for example `560001`. A live courier charge from Shiprocket appears as a separate line.
5. **Pay with PayPal (sandbox).** Choose **Pay now & get it delivered**. You will be sent to PayPal's sandbox.
   - Log in with a sandbox **Personal** buyer account. Create one in the PayPal Developer Dashboard under **Sandbox → Accounts** if you don't have one. No real money is charged.
   - Approve the payment. You return to the site's order-success page with a receipt you can copy.
6. **Other ordering options.** The checkout also offers **Self pickup in Ahmedabad** (free) and **Confirm first, pay later**. These do not use PayPal. The order is sent to the artist on Instagram, and the buyer pays after confirmation.

## Option B: agent endpoint (agentic commerce)

The agent endpoint speaks JSON-RPC 2.0 and uses MCP-style tool calls. Replace the example values as needed.

List the tools:

```bash
curl -s -X POST https://hetaas-api.automated-aesthetix.workers.dev/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Search products under ₹500:

```bash
curl -s -X POST https://hetaas-api.automated-aesthetix.workers.dev/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search_products","arguments":{"query":"gift","max_price":500}}}'
```

Create a PayPal checkout for a buyer. The response includes an `approveUrl`. Open it in a browser and pay with a sandbox buyer account. `id` is a product id from the search result.

```bash
curl -s -X POST https://hetaas-api.automated-aesthetix.workers.dev/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"create_checkout","arguments":{"items":[{"id":67,"qty":1}],"customer":{"name":"Test Buyer","email":"buyer@example.com","phone":"9876543210","address":"12 Test Road","city":"Ahmedabad","state":"Gujarat","pincode":"380052"}}}}'
```

The agent never handles payment. It only prepares the order. The buyer approves the payment on PayPal.

## Option C: run it locally

Requirements: Node.js 18 or newer. See the repository README for the full steps.

```bash
git clone https://github.com/AasthaThakker/hetaas-paypal-ai-hackathon.git
cd hetaas-paypal-ai-hackathon
npx serve . -l 8080 &          # static site at http://localhost:8080
cd worker && npm install
cp .dev.vars.example .dev.vars # fill in PayPal sandbox keys and a Gemini or Groq key
npx wrangler dev               # API at http://127.0.0.1:8787
```

Localhost runs in demo mode automatically. Without PayPal sandbox keys, checkout will not work. Without a Gemini or Groq key, Resi will not answer. Without Shiprocket keys, the delivery quote at checkout will fail. The catalogue and basket work either way.

## Health check

```bash
curl -s https://hetaas-api.automated-aesthetix.workers.dev/api/health
```

Expected response: `{"ok":true}`

## Notes

- The demo uses PayPal's **sandbox**. No real money is charged, and no real order is shipped.
- Delivery charges are calculated from the PIN code. Use a real Indian PIN code, for example `560001` or `380052`.
- The AI concierge uses a free model quota. If the first model is busy, the Worker falls back to a second model automatically, so a short delay is normal.
- The live shop at https://hetaas-atelier.aasthathakker.com/ is the same code without PayPal. It is not needed for testing.
- Demo video: `<add YouTube link>`
