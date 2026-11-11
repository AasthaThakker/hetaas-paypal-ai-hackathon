# HetAas Atelier: PayPal + AI Commerce Concierge

## Tagline

A handmade resin-art shop where an AI concierge answers questions, recommends gifts, and takes PayPal payments, and AI agents can buy on a customer's behalf.

## Links

- Live demo (PayPal sandbox, all features): https://hetaas-demo.pages.dev
- Code repository (MIT license): https://github.com/AasthaThakker/hetaas-paypal-ai-hackathon
- Demo video (under 3 minutes): <add YouTube link>
- Live shop without PayPal (same code, separate mode): https://hetaas-atelier.aasthathakker.com/

## Inspiration

HetAas Atelier makes handmade resin pieces in Ahmedabad: bookmarks, earrings, keychains, coasters, pooja thalis and custom name pieces. The shop already took orders over Instagram DMs, which meant every question, price check and delivery quote was a manual reply. Customers often do not know which piece suits a budget or occasion, and buyers who want a custom name piece need a clear way to order and pay. We wanted the shop to answer questions and take payment without giving up the handmade feel, and we wanted AI agents to be able to shop it too.

## What it does

HetAas Atelier is a commerce site with three parts: a shopper experience, a PayPal checkout, and an agent endpoint.

**For shoppers**
- **Resi, the AI concierge:** a chat assistant that answers questions about products, care, custom pieces and orders, using the live catalogue and FAQs. It suggests gifts within a budget and one add-on. Unrelated questions are refused before the model is called.
- **Basket and custom pieces:** shoppers add items, choose custom name stickers and similar options, and see totals.
- **Delivery charge from the PIN code:** the shopper enters a PIN code and sees the live courier charge from Shiprocket, which is shown as a separate line on the PayPal bill.
- **Three ways to order:** pay now with PayPal and get it delivered, self pickup in Ahmedabad (free), or confirm first and pay later.
- **PayPal checkout (sandbox):** the buyer approves the order on PayPal and returns to the site. The receipt page gives a copyable order summary.

**For the business owner**
- Owner receipt emails for each paid PayPal order, including buyer details, delivery charge and courier.
- A webhook fallback: if the buyer never returns to the site after paying, PayPal's capture webhook still captures the order and emails the owner.

**For AI agents**
- `POST /mcp` is an MCP-style JSON-RPC endpoint with three tools: `search_products`, `get_product` and `create_checkout`. An agent can search the catalogue, price a basket with live delivery, and get a PayPal approval link to give to the buyer. The buyer still approves the payment on PayPal.
- `llms.txt` describes the shop and its endpoints for agents.

## How we built it

- **Frontend:** plain HTML, CSS and JavaScript, no build step. It is a mobile-first installable PWA with offline support. The same code runs two modes, chosen by the site address: `demo` (PayPal sandbox and every feature) and `live` (the shop without PayPal).
- **API:** a Cloudflare Worker on the free tier holds every secret. The browser never sees PayPal, AI or shipping keys.
- **PayPal:** Orders v2 for creating and capturing orders in the sandbox. Prices and totals are calculated on the server, so the browser cannot change them. The webhook is signature-verified with PayPal's verify-webhook-signature API.
- **AI:** Gemini models are tried in order. If all of them fail, Groq (openai/gpt-oss-120b) answers. The system prompt is built from the catalogue and FAQs for each question, and a topic filter runs before any model call.
- **Shipping:** Shiprocket's serviceability API finds the cheapest courier from the pickup PIN code, and a fixed markup is added.
- **Email:** Resend sends owner receipts. A key stored in Cloudflare KV prevents duplicate emails for the same PayPal order.
- **Hosting:** Cloudflare Pages for the demo, GitHub Pages for the live shop, and a weekly GitHub Action that refreshes the Shiprocket login token.

## Challenges we ran into

- **PayPal sandbox does not accept INR.** Shop prices are in rupees, so the Worker converts them to USD at a fixed rate before creating the PayPal order. The rate is set in `worker/wrangler.toml`.
- **Shiprocket blocks logins from Cloudflare's servers.** The Worker could not log in by itself, so a GitHub Action logs in from outside Cloudflare and stores the token in KV.
- **Free AI quotas run out.** Each Gemini model has its own daily limit, so the Worker tries each configured model in turn, then falls back to Groq.
- **Keeping the AI on topic.** A word filter runs before any model call, and the prompt limits answers to the shop, so the concierge cannot be used as a general chatbot.
- **Two modes from one codebase.** The demo has PayPal and every feature. The live shop has no card payments and sends delivery orders to the artist on Instagram. Both come from the same code, so the live shop never shows PayPal.

## Accomplishments we're proud of

- A working end-to-end flow in the PayPal sandbox: from a chat question to a basket, a live delivery charge, a PayPal payment, and an owner email.
- Prices, totals and delivery charges are all calculated on the server.
- An agent-ready endpoint: an AI agent can search, price and create a PayPal checkout through standard JSON-RPC, and the buyer still approves the payment on PayPal.
- One codebase serves a real shop and the hackathon demo. PayPal stays hidden on the live shop.

## What we learned

- Agentic commerce works best when the agent prepares the order and the human approves the payment on PayPal. The agent never handles money.
- Server-side pricing and signature-checked webhooks matter more than front-end polish when a payment flow is involved.
- Small shops need the same tools as large ones, but on free tiers with fallbacks, because quotas and logins fail in unexpected ways.

## What's next

- Move the demo from sandbox to live PayPal once the business is ready (`PAYPAL_ENV=live` and real keys).

## Built with

PayPal REST API (Orders v2, webhooks), Cloudflare Workers, Cloudflare Pages, Cloudflare KV, Google Gemini API, Groq API, Shiprocket, Resend, GitHub Actions, GitHub Pages, HTML, CSS, JavaScript, PWA (service worker and web manifest), MCP-style JSON-RPC.
