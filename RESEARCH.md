# Research: eBay Card Lister

> **What we learned researching this product. For anyone evaluating whether to build it.**

*March 2026 | Flume SaaS Factory*

---

## The Problem

Parents are sitting on kids' sports card collections they can't sell. Researching each card takes 10–20 minutes. Writing a good listing is a skill. Tracking and relisting is tedious. Most people give up — cards sit in shoeboxes indefinitely.

The sports card market went through a massive boom from 2020–2024. Millions of new collectors (many of them kids) entered the market. Those kids have grown up, and now there's a massive backlog of unsold collections sitting in closets.

**The target user is not a dealer.** It's a parent who wants to help their kid convert a box of cards into cash, has no idea what anything is worth, and doesn't have time to become a sports card expert just to sell a collection.

---

## The Market

- **Sports cards are a $10B+ market** (all-time peak in 2021, still elevated above pre-2020 levels)
- **PSA** (the dominant grading company) was valued at $700M+
- **eBay is the dominant marketplace** for sports cards — no serious seller bypasses it
- **Parents are the buyer persona**: time-poor, emotionally invested in helping their kid, willing to pay for convenience

The intersection of "big market," "dominant platform," "tech-savvy but non-expert users," and "pain point nobody has solved" is genuinely rare.

---

## Why Now

The infrastructure is mature. eBay has a free developer program with generous API limits (2M calls/day for Inventory API). AI vision (GPT-4o) can identify most common cards from photos with high accuracy. Stripe handles payments. This is not a science project — all the building blocks exist.

The window is now. As AI vision models improve, the hardest part (card identification from bad photos) gets easier. But the first mover in this specific UX — photo in, listing live, for casual sellers — has a real advantage.

---

## eBay API — What We Learned

### Getting Access

The eBay Developer Program is free. Sign up at developer.ebay.com, get keys immediately. Production keys require a verified eBay seller account, but that's standard for anyone who wants to sell.

### The Right API to Use

eBay has two main API approaches:

**Inventory API (REST)** — *Recommended*
- Modern REST API with 2M calls/day limit
- Listings are a 3-step process: Create inventory item → Create offer → Publish
- Listings can only be edited through the API (not Seller Hub) — important limitation

**Trading API (XML/SOAP)** — Legacy
- Single-call listing creation
- 5,000 calls/day limit (much lower)
- eBay is pushing developers to Inventory API

**Use Inventory API.** The extra step is worth the higher rate limit and modern architecture.

### OAuth Is Required

To post on behalf of a user, you need their OAuth token — Authorization Code Grant flow. This is standard OAuth: user clicks "Connect eBay," logs in, grants permission, and your app gets a refresh token you store securely.

Required scopes for listing:
- `sell.inventory` — create and manage inventory
- `sell.account` — access account info and business policies
- `sell.fulfillment` — track order fulfillment

### Business Policies Are Required

Before posting, each seller needs three policies set up (one-time per seller, stored for reuse):
- **Fulfillment Policy**: How the item ships
- **Payment Policy**: Payment methods accepted
- **Return Policy**: Return window

These are created via the Account API and stored per user.

### Photos Need Special Handling

Photos must be uploaded to eBay's servers first (via Media API) before they can be used in a listing. Maximum 24 photos per listing. Sports cards need front + back minimum.

### Sports Card Categories

| Sport | eBay Category ID |
|-------|-----------------|
| Baseball | 212 |
| Football | 27211 |
| Basketball | 27212 |
| Hockey | 27213 |
| Pokemon | 183454 |

Category-specific item specifics are required: Player name, Year, Brand, Set, Card Number, Grade (if graded).

---

## AI Card Identification — What We Learned

### Modern AI Can Do This

GPT-4o Vision (and comparable models) can identify most common sports cards from photos with high accuracy. The key fields it can extract:
- Player name
- Year
- Brand (Topps, Panini, Upper Deck)
- Card number
- Parallel variant (Silver, Gold, Black, etc.)
- Rookie/Auto/Patch flags
- Grade (if visible)

### The Hard Cases

**Blurry photos** — Kids take bad photos. The AI needs good prompting and sometimes a fallback (ask for the back of the card, or the cert number if graded).

**Similar parallels** — Silver vs. Gold parallel of the same card look nearly identical in a photo. The AI can get close but can't always distinguish.

**Graded vs. ungraded** — Without seeing the cert sticker, a PSA 10 and an ungraded gem mint look identical in a photo.

**Solution: Human-in-the-loop.** The AI presents its identification. The user confirms or corrects. Nothing goes live without human approval.

### Accuracy Enhancement

For graded cards, ask the user for the cert number and verify via PSA cert lookup (psacard.com/cert). This is a huge accuracy boost for the cards where it matters most.

---

## Pricing Data — The Hardest Problem

### The Gap in eBay's API

eBay's Browse API gives you active listings, but **does not expose sold/completed listing data** directly. This is a known gap. You can't easily pull "what did this card actually sell for in the last 90 days?"

### Workarounds

1. **Active listing analysis** — Use current lowest price as a floor indicator. Active sellers at a given price are implicitly telling you the market price.

2. **Grade-adjusted interpolation** — If you have comps at PSA 9 but your card is PSA 10, apply known grade differentials (PSA 10 typically sells for 2–3x PSA 9 for common cards).

3. **Confidence indicators** — When data is thin, say so. Show "low confidence" when there are fewer than 3 comps. Let the user adjust.

4. **Human override** — The user always sets the final price. Our job is to give them the best information, not to promise perfect pricing.

### Caching Is Essential

Pricing data should be cached for at least 1 hour per card. eBay Browse API has a 5,000 calls/day limit (shared across all apps). With 100 users doing 30 listings each = 3,000 API calls/day just for pricing. Cache aggressively.

---

## Competitive Landscape

### What Exists

| Tool | Type | Problem |
|------|------|---------|
| Mass buylist services | Buy collections | User sells TO them, not ON eBay — they lowball |
| Turbo Lister | eBay tool | No AI, no pricing, full manual entry |
| CardMinder | Inventory app | No eBay integration |
| PSA Tracker | Grading lookup | No listing |
| DeckTrade | Trading | Card trades, not cash sales |

### The Gap

**Nobody is doing "photo → live eBay listing with AI vision + automated pricing + relisting for casual sellers."**

Every existing tool is built for one of two audiences: professional dealers (complex, powerful) or casual price-checkers (read-only). The middle — real automation with a simple, non-intimidating UX for non-experts — is empty.

### Our Moat

1. **AI vision card identification from bad photos** — This is genuinely hard. Getting it right for blurry, glare-filled, sleeve-in-photo shots is the technical differentiator.

2. **UX for non-experts** — Every other tool assumes you already know what you have. We assume you have no idea.

3. **Auto-relist** — Captures the "I listed it and forgot about it" problem. This is where casual sellers lose the most money.

---

## Revenue Model

**$2.50 per listing + $1.00 auto-relist**

Simple. No commission. No subscription. Pay per listing.

**Why this works:**
- For a card selling at $20–30, $2.50 is ~8–12% of the sale price. Reasonable for the convenience.
- No upfront risk for the user if the card doesn't sell (they only pay if the listing goes live).
- Auto-relist at $1 is a no-brainer for anyone who's already moved on.

**Scale math:**
- 100 parents/month × 30 cards each = 3,000 listings
- 3,000 × $2.50 = **$7,500/month revenue**
- No inventory. No shipping. No handling. Pure SaaS margin.

---

## Key Technical Decisions

### Why Neon PostgreSQL
Serverless PostgreSQL from the creators of Prisma. No connection pool management. Scales to zero when not in use. Works perfectly with Vercel serverless functions and Fastify.

### Why Fastify Over Express
Faster. Built-in TypeScript support. Better plugin ecosystem. Less opinionated than NestJS.

### Why BullMQ + Redis
Background job queue for async work (photo processing, eBay API calls). Retry logic built in. Prevents API rate limit issues from blocking the user-facing request.

### Why JWT + Refresh Tokens
Standard auth pattern. Short-lived access tokens (1hr) with HTTP-only refresh token cookies. Works for SPA without CORS headaches.

---

## The Hardest Problems to Solve

**1. Card identification from blurry photos**
Not every photo will work. Need a fallback UX: ask for more photos, the back of the card, or manual entry. The UX around "we couldn't figure it out" is as important as the happy path.

**2. Pricing accuracy**
Sports card markets can be thin. Some cards only have 0–2 sales in the last 6 months. The algorithm needs to say "I don't know" clearly rather than overconfidently recommend a price.

**3. User trust**
Giving a third-party app access to your eBay account is a trust leap. The UX needs to be transparent about exactly what permissions are granted and make revocation trivial.

**4. eBay compliance**
eBay has strict policies on automated listing practices. The app must stay within their terms — no bulk posting, no duplicate listings, no gaming the system. This means the app's behavior is constrained by eBay's rules.

---

## What We'd Tell a Developer Building This

1. **Start with eBay sandbox.** Set up the full OAuth flow with sandbox keys before touching production. The OAuth flow is the hardest part to debug.

2. **GPT-4o prompting matters.** Invest time in prompt engineering for sports card identification. The difference between a good prompt and a generic one is significant.

3. **Cache aggressively.** eBay Browse API has a 5K/day limit. Price caching for 1 hour means you're making 1 API call per unique card per hour, not 1 call per listing attempt.

4. **Human-in-the-loop is non-negotiable.** Not because the AI is bad, but because users won't trust an app that posts to their eBay account without their approval.

5. **Test with real cards.** Not stock photos. Get actual blurry photos from a phone camera. That's the real test.

---

## Sources

- [eBay Developer Portal](https://developer.ebay.com)
- [eBay Sell Inventory API Docs](https://developer.ebay.com/api-docs/sell/inventory)
- [eBay OAuth Guide](https://developer.ebay.com/api-docs/static/oauth-authorization-code-grant.html)
- [PSA Cert Lookup](https://www.psacard.com/cert/)
- [GPT-4o Vision Documentation](https://platform.openai.com/docs/vision)
- [Stripe Docs](https://stripe.com/docs)

---

*Research compiled by Scout | Flume SaaS Factory | March 2026*
