# eBay Card Lister

**Snap a photo. Get a fair price. Sell on eBay — automatically.**

eBay Card Lister is an AI-powered tool that takes the pain out of selling sports cards on eBay. Take a photo of any card, and the system identifies it, finds real market comps, writes the listing, and posts it live to eBay — all in a few minutes.

---

## What It Does

1. **Photo** — Snap a picture of your sports card
2. **Identify** — AI reads the photo and figures out exactly what card it is (player, year, brand, card number)
3. **Confirm** — You verify the AI got it right (or correct it)
4. **Price** — See what similar cards sold for recently (floor, average, ceiling)
5. **List** — One tap and your card is live on eBay with a professionally-written listing
6. **Sell** — Get notified when it sells

That's it. No research. No guesswork. No fighting with eBay's listing form.

---

## Who It's For

Parents helping their kids sell a collection. Casual sellers who don't know card market values. Anyone with a box of cards who's been putting off listing them because it seems like too much work.

If you've ever thought *"I have no idea what this is worth"* or *"this will take forever to figure out"* — this is the tool.

---

## Pricing

| Action | Cost |
|--------|------|
| Create a listing | **$2.50** |
| Auto-relist if it doesn't sell | **$1.00** |

No subscription. No commission on your sale. You keep the proceeds minus eBay's standard fees (~10–15% + $0.35/item, paid directly to eBay).

For a card selling at $20–30, the $2.50 fee is roughly 8–12% of the sale price — a fair price for 30 minutes of saved research and the confidence of accurate pricing.

---

## How It Works

The full product spec is in [SPEC.md](./SPEC.md) — if you're building this or evaluating the technical approach, that's the document you want.

Key technical pieces:

- **Card Identification**: GPT-4o Vision analyzes your photo and extracts player name, year, brand, card number, and more
- **Pricing Data**: Real comps from eBay's Browse API (what similar cards actually sold for recently)
- **eBay Integration**: Direct API integration via eBay's Inventory API — your listing goes live without copy-pasting anything
- **Human-in-the-loop**: You always approve before anything goes live

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React + Vite + Tailwind CSS |
| Backend | Fastify + Node.js |
| Database | Neon PostgreSQL + Prisma |
| AI | OpenAI GPT-4o (vision) |
| Payments | Stripe |
| Notifications | Email (Resend) + SMS (Twilio) |
| eBay API | Inventory API v1 + Media API + Browse API |

Stack is Flume-standard. See SPEC.md for the full architecture.

---

## Getting Started

### Prerequisites

- Node.js 20+
- npm or pnpm
- eBay Developer account (free at [developer.ebay.com](https://developer.ebay.com))
- OpenAI API key
- Stripe account

### Setup

```bash
# Clone the repo
git clone https://github.com/tylerdotai/ebay-card-lister.git
cd ebay-card-lister

# Backend
cd backend
cp .env.example .env
# Fill in environment variables (see SPEC.md for full list)
npm install
npx prisma migrate dev
npm run dev

# Frontend (separate terminal)
cd frontend
cp .env.example .env
npm install
npm run dev
```

### Environment Variables

See [SPEC.md](./SPEC.md#17-environment-variables) for the complete list of required environment variables.

---

## Project Status

**Currently:** Pre-build. This repo contains the full product specification.

**Next:** Build implementation.

---

## License

Private — All rights reserved.

---

*Built by [Flume SaaS Factory](https://flume.sh)*
