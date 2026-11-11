# HetAas Atelier: PayPal + AI Commerce Concierge

A working commerce site for a handmade resin-art business in Ahmedabad, India, built for the PayPal Developer Platform hackathon (sandbox). Shoppers chat with an AI concierge, build a basket, get a live delivery charge from their PIN code, and pay with PayPal. AI agents can run the same flow through an MCP-style endpoint.

- 🧪 **Live demo (PayPal sandbox, all features):** https://hetaas-demo.pages.dev
- 🛍️ Live shop without PayPal (same code, separate mode): https://hetaas-atelier.aasthathakker.com/

To pay on the demo, use a PayPal **sandbox Personal** test account. No real money moves.

## What it does

| Feature | Where it lives |
|---|---|
| **PayPal Orders v2 checkout (sandbox):** the server prices the basket, creates the order, and captures it after buyer approval | `worker/src/index.js` (`handleCheckout`, `handleCapture`) |
| **PayPal webhook fallback:** `PAYMENT.CAPTURE.COMPLETED` is signature-checked with PayPal's verify-webhook-signature API. If the browser never returns, the order is still captured and the owner is emailed | `handleWebhook` |
| **Server-side pricing:** the browser cannot change prices or totals. Shop prices are in INR and converted to USD at a fixed rate, because PayPal's sandbox does not accept INR | `priceBasket`, `INR_PER_UNIT` in `worker/wrangler.toml` |
| **Live delivery charges (Shiprocket):** the cheapest courier from the pickup PIN code plus a fixed markup, shown as its own line on the PayPal bill | `handleShippingQuote`, `quoteShipping` |
| **AI concierge "Resi":** answers product, care, and order questions from the live catalogue and FAQs, and suggests gifts within a budget. Off-topic requests are refused before any model call | `handleChat`, `buildSystemPrompt` |
| **AI fallback:** Gemini models are tried in order. If all fail, Groq (`openai/gpt-oss-120b`) answers | `askModels`, `callGemini`, `callGroq` |
| **Agentic commerce endpoint:** `POST /mcp` (JSON-RPC, MCP-style tools `search_products`, `get_product`, `create_checkout`). An agent gets a PayPal approval link for the buyer | `handleMcp`, [`llms.txt`](llms.txt) |
| **Owner receipt emails:** the owner gets one receipt per PayPal order, sent through Resend. Buyers get a copyable receipt. Receipt access is protected by an HMAC key given only to the browser that made the checkout | `handleCapture`, `notifyOwnerOnce`, `sendOwnerEmail` |
| **Three ways to order:** pay now with PayPal and get it delivered, self pickup (free), or confirm first and pay later | `app.js` checkout |
| **Progressive web app:** installable, offline-capable, mobile-first | `sw.js`, `manifest.webmanifest` |

## How to test (for judges)

1. Open https://hetaas-demo.pages.dev. A banner marks it as the sandbox demo.
2. Ask Resi in the chat, for example "a gift under ₹500 for my sister". It answers only from the catalogue.
3. Add items to the basket, open checkout, enter a 6-digit Indian PIN code to see the live delivery charge, and choose **Pay now & get it delivered**.
4. Pay with a PayPal sandbox buyer account. Create one in the PayPal Developer Dashboard under Sandbox → Accounts.
5. Optional, agent flow: send a JSON-RPC `tools/call` for `create_checkout` to `POST https://hetaas-api.automated-aesthetix.workers.dev/mcp`. The response includes a PayPal `approveUrl`.

## Architecture

```
Browser (static site: HTML/CSS/JS) ──► Cloudflare Worker API (worker/) ──► PayPal REST API (sandbox)
                                                 │                      ──► Gemini API (Groq fallback)
                                                 │                      ──► Shiprocket (delivery rates)
                                                 │                      ──► Resend (owner emails)
                                                 └──► Cloudflare KV (email dedupe, Shiprocket token)
```

- Secrets (PayPal, Gemini, Groq, Resend, Shiprocket) live only in the Worker. The static site never sees them.
- `config.js` picks the mode from the site address. `demo` (the Pages site and localhost) enables PayPal and every feature. `live` (the GitHub Pages shop) runs without PayPal.
- The static site is plain HTML, CSS, and JavaScript with no build step. The Worker runs on Cloudflare's free tier.

## Run it locally

You need Node.js 18 or newer. PayPal sandbox keys are needed for checkout. A Gemini or Groq key is needed for the chat. Resend and Shiprocket are optional.

### 1. Static site

```bash
# from the repository root
npx serve .            # or: python -m http.server 8080
```

Open http://localhost:8080. Localhost runs in `demo` mode and calls the local Worker at `http://127.0.0.1:8787`.

### 2. API Worker

```bash
cd worker
npm install
cp .dev.vars.example .dev.vars    # Windows: copy .dev.vars.example .dev.vars
# fill in .dev.vars with your keys (see the table below)
npx wrangler dev
```

`wrangler dev` listens on `http://127.0.0.1:8787`. Keep it running while you use the site.

### Keys for `.dev.vars`

| Variable | Where to get it |
|---|---|
| `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET` | PayPal Developer Dashboard → Apps & Credentials → Sandbox → create an app |
| `PAYPAL_WEBHOOK_ID` | Sandbox app → Webhooks → add a webhook for `PAYMENT.CAPTURE.COMPLETED`, then copy its ID |
| `GEMINI_API_KEY` | Google AI Studio (free tier) |
| `GROQ_API_KEY` | console.groq.com (free tier, optional fallback) |
| `RESEND_API_KEY`, `OWNER_EMAIL` | resend.com (optional, for owner receipt emails) |
| `SHIPROCKET_EMAIL`, `SHIPROCKET_PASSWORD` | Shiprocket API user (optional, for live delivery charges) |

`PAYPAL_ENV` is `sandbox` in `worker/wrangler.toml`. Never commit `.dev.vars`; it is git-ignored.

### 3. Deploy your own copy (optional)

```bash
cd worker
npx wrangler deploy
npx wrangler secret put PAYPAL_CLIENT_SECRET     # repeat for each secret above
```

Then set `window.HETAAS_API` in [`config.js`](config.js) to your Worker URL, and deploy the static files to any static host (Cloudflare Pages, GitHub Pages, Netlify).

## Project structure

| Path | Purpose |
|---|---|
| `index.html`, `custom.html`, `collab.html`, `order-success.html` | Pages: shop, custom design studio, collaboration, order confirmation |
| `app.js`, `chat.js` | Catalogue, basket, checkout, and the Resi chat UI |
| `styles.css` | All styling |
| `config.js` | API URL and demo/live mode |
| `products.json`, `content.json` | Catalogue and site text (editable) |
| `worker/src/index.js` | The API: PayPal, AI, Shiprocket, email, MCP endpoint |
| `worker/wrangler.toml` | Public settings (currency, FX rate, models, weights, KV binding) |
| `llms.txt` | Agent-facing description of the shop and its endpoints |
| `sw.js`, `manifest.webmanifest` | PWA support |

## What was built for this submission

The shop was already a static site. For this submission it gained the PayPal Orders v2 checkout with server-side pricing, the webhook fallback, the Resi AI concierge with its model fallback, live Shiprocket delivery charges, the MCP agent endpoint, owner receipt emails, and a demo mode that runs these features without changing the live shop.

## License

MIT. See [LICENSE](LICENSE).
