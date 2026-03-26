# eBay Card Lister — Product Specification

> **Hand this spec to a developer and they can build it.**

---

## 1. Product Overview

**What it does:**
Snap a photo of any sports card, and the eBay Card Lister automatically identifies it, finds real market comps, prices it intelligently, writes a complete listing, and posts it live to eBay — all with minimal friction.

**Who it's for:**
Parents helping their kids sell a sports card collection on eBay. The target user has no experience with card valuation, no time for manual research, and no interest in learning the intricacies of eBay listing optimization. They just want the card to sell for a fair price without doing the work.

**The experience in one sentence:**
Text mom a photo of the card → she approves the listing → it goes live on eBay → she gets a notification when it sells.

**Why it matters:**
A 50-card collection takes 10–20 hours to research and list manually. Most parents and kids give up. Cards sit in shoeboxes. This tool makes the entire process take five minutes per card.

---

## 2. User Flow

```
[User takes photo of card]
        ↓
[Uploads to eBay Card Lister]
        ↓
[AI Vision identifies card]
  "Is this a 2019 Panini Prizm Luka Doncic Rookie Card #237?"
        ↓
[User confirms or corrects identification]
        ↓
[System searches eBay for sold comps]
  → Finds 12 cards sold in last 90 days: $18–$45 range
  → Average: $28 | Floor: $18 | Ceiling: $45
        ↓
[System presents recommended price + listing draft]
  User can adjust price, title, or description
        ↓
[User taps "Post to eBay"]
        ↓
[System uploads photos → creates inventory item → creates offer → publishes]
        ↓
[User receives link to live eBay listing]
        ↓
[System tracks listing: views, offers, ended]
        ↓
[If unsold after 7 days: auto-relist option fires]
        ↓
[Notification when card sells]
```

**Key principle: Human-in-the-loop at the identification and pricing step. Nothing goes live without user approval.**

---

## 3. Technical Architecture

### Stack (Flume Standard)

| Layer | Technology |
|-------|------------|
| **Frontend** | React 18 + Vite + Tailwind CSS |
| **Backend** | Fastify + Node.js |
| **Database** | Neon PostgreSQL (serverless) |
| **ORM** | Prisma |
| **Auth** | JWT (email/password + OAuth) |
| **Payments** | Stripe ($2.50/listing + $1 auto-relist) |
| **Notifications** | Email (Resend/SendGrid) + SMS (Twilio) |
| **AI Vision** | GPT-4o (OpenAI) |
| **eBay API** | Inventory API v1 + Media API + Browse API |
| **Hosting** | Vercel (frontend) + Railway/Render (backend) |

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND (React)                      │
│  - Card upload (photo capture or file select)               │
│  - Identification confirmation screen                       │
│  - Pricing display + listing preview                        │
│  - Dashboard: active listings, sold, earnings               │
│  - Settings: connect eBay, payment, notifications           │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      BACKEND (Fastify)                      │
│  - REST API with typed routes                               │
│  - JWT auth middleware                                       │
│  - Job queue (BullMQ + Redis) for async work                │
│  - Stripe webhook handler                                    │
│  - eBay OAuth token manager                                  │
│  - GPT-4o vision client                                      │
└──────────┬─────────────────┬─────────────────┬──────────────┘
           │                 │                 │
           ▼                 ▼                 ▼
    ┌────────────┐    ┌────────────┐    ┌────────────────────┐
    │   Neon     │    │   Redis    │    │   External APIs    │
    │ PostgreSQL │    │  (Queue)   │    │  - OpenAI (GPT-4o) │
    │            │    │            │    │  - eBay API        │
    └────────────┘    └────────────┘    │  - Stripe          │
                                        │  - Twilio          │
                                        │  - Resend/SendGrid │
                                        └────────────────────┘
```

### Key API Design

**Base URL:** `https://api.ebay-card-lister.com/v1`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Login, returns JWT |
| POST | `/auth/ebay/connect` | Initiate eBay OAuth |
| GET | `/auth/ebay/callback` | OAuth callback handler |
| POST | `/cards/identify` | Upload photo, get AI identification |
| POST | `/cards/:id/confirm` | Confirm/correct AI identification |
| GET | `/cards/:id/pricing` | Get pricing comps for a confirmed card |
| POST | `/listings/create` | Create and publish listing |
| GET | `/listings` | List user's active/sold/ended listings |
| GET | `/listings/:id` | Get listing details |
| POST | `/listings/:id/relist` | Manually trigger relist |
| POST | `/webhooks/stripe` | Stripe payment webhook |
| POST | `/webhooks/ebay` | eBay notification webhook |

### Directory Structure

```
/
├── frontend/                  # React + Vite + Tailwind
│   ├── src/
│   │   ├── components/        # UI components
│   │   ├── pages/             # Route pages
│   │   ├── hooks/             # Custom hooks
│   │   ├── lib/               # API client, utils
│   │   └── stores/            # State management
│   └── index.html
├── backend/                   # Fastify + Prisma
│   ├── src/
│   │   ├── routes/            # API route handlers
│   │   ├── services/          # Business logic
│   │   │   ├── ebay.service.ts
│   │   │   ├── vision.service.ts
│   │   │   ├── pricing.service.ts
│   │   │   ├── stripe.service.ts
│   │   │   └── notification.service.ts
│   │   ├── jobs/               # BullMQ job handlers
│   │   ├── middleware/         # Auth, error handling
│   │   └── lib/                # eBay client, OpenAI client
│   ├── prisma/
│   │   └── schema.prisma
│   └── package.json
└── README.md
```

---

## 4. AI Card Identification

### How It Works

1. User uploads 1–4 photos of the card (front required, back recommended)
2. Backend sends photos to GPT-4o Vision with a structured prompt
3. GPT-4o returns extracted metadata
4. System presents identification to user for confirmation
5. User can correct any field before proceeding

### Prompt Structure

```
You are a sports card identification expert. Analyze this image(s) and extract:
- player_name: Full name of the player
- year: Year of the card (e.g., 2019)
- brand: Card brand (e.g., Topps, Panini, Upper Deck)
- brand_line: Brand tier (e.g., Prizm, Chrome, Optic)
- set_name: Full set name (e.g., Prizm Basketball)
- card_number: Card number (e.g., #237)
- parallel: Parallel variant if visible (e.g., Silver, Gold, Black)
- rookie_card: Boolean — is this a rookie card?
- auto: Boolean — does this have an autograph?
- patch: Boolean — does this have a jersey patch?
- graded: Boolean — does this appear to be professionally graded?
- grader: Which grader (PSA, BGS, CGC) if visible
- grade: Grade number if visible (e.g., 10, 9.5)
- sport: basketball, baseball, football, hockey
- estimated_value_range: Your best guess at market value range
- confidence: low/medium/high
- notes: Any special observations

Return as structured JSON.
```

### Fallback Behavior

- If confidence is low, ask user for the back of the card or cert number
- If the card is graded, prompt for cert number → verify via PSA cert lookup
- If identification fails entirely, show user a manual entry form

---

## 5. eBay API Integration

### OAuth Flow (Per-User)

```
1. User clicks "Connect eBay" in the app
2. Backend redirects to eBay OAuth URL with our Client ID + RuName
3. User logs into eBay and grants permission
4. eBay redirects to our callback URL with ?code=XXXXXXXX
5. Backend exchanges code for access_token + refresh_token
6. Store refresh_token encrypted in database
7. Use access_token for API calls; refresh automatically before expiry
```

**Required OAuth Scopes:**
- `https://api.ebay.com/oauth/api_scope/sell.inventory`
- `https://api.ebay.com/oauth/api_scope/sell.account`
- `https://api.ebay.com/oauth/api_scope/sell.product`
- `https://api.ebay.com/oauth/api_scope/sell.fulfillment`

### Listing Creation Pipeline

```
Step 1: Upload Photos
  POST /commerce/media/v1/image/create_image_from_file
  → Returns eBay-hosted image URLs

Step 2: Create/Update Inventory Item
  PUT /sell/inventory/v1/inventory_item/{sku}
  Payload: { product, condition, quantity, packageWeightAndSize, images }
  SKU format: {userId}-{cardId}-{timestamp}

Step 3: Create Offer
  POST /sell/inventory/v1/offer
  Payload: { sku, marketplaceId, categoryId, condition, pricingSummary, fulfillmentPolicyId, paymentPolicyId, returnPolicyId }

Step 4: Publish Offer
  POST /sell/inventory/v1/offer/{offerId}/publish
  → Returns live eBay listing ID and URL
```

### Business Policies

Each user needs three policies (created once, reused for all listings):
- **Fulfillment Policy**: How item ships (e.g., Calculated, Flat Rate)
- **Payment Policy**: Payment methods (e.g., PayPal, immediate payment required)
- **Return Policy**: Return window and method (e.g., 30-day returns)

These are created via the Account API and stored in the database per user.

### Sports Card Category IDs

| Sport | Category ID |
|-------|-------------|
| Baseball | 212 |
| Football | 27211 |
| Basketball | 27212 |
| Hockey | 27213 |
| Pokemon | 183454 |
| Non-Sport | 26450 |

### Item Specifics (Required for Sports Cards)

eBay requires specific item specifics for card listings:
- Player/Character Name
- Year
- Brand
- Set
- Card Number
- Grade (if graded)
- Professional Grader (PSA, BGS, CGC)

---

## 6. Pricing Data Strategy

### Primary: eBay Browse API

```
GET /buy/browse/v1/item_summary/search
  ?q={player_name}+{year}+{brand}+{card_number}
  &filter=condition:{grade}
  &sort=price:asc
  &limit=20
```

Returns active listings with prices. Use to find:
- **Current floor** (lowest price)
- **Current ceiling** (highest price)
- **Average active price**

### Secondary: Sold Listings (Approximation)

eBay Browse API doesn't expose sold data directly. Workarounds:

1. **Active listing analysis**: Use active listings as a floor indicator (items already at market price)
2. **Grade-adjusted estimation**: If a card has no sales at a given grade, interpolate from known grade differentials (e.g., PSA 10 typically sells for 2–3x PSA 9)
3. **Third-party price guides**: PSA price guide, Beckett as fallback for common cards

### Pricing Algorithm

```typescript
function calculatePrice(comps: EbayComp[], card: Card): PriceRecommendation {
  const activePrices = comps.map(c => c.price).filter(p => p > 0)
  
  const floor = Math.min(...activePrices)
  const avg = activePrices.reduce((a, b) => a + b, 0) / activePrices.length
  const ceiling = Math.max(...activePrices)
  
  // Adjust for grade vs. comp grades
  const gradeAdjustment = getGradeAdjustment(card.grade, compGrades)
  
  // Trend: if recent sales are moving up, bias toward ceiling
  const trend = calculateTrend(recentSales)
  
  // Recommended price: weighted blend
  const recommended = (floor * 0.3 + avg * 0.4 + ceiling * 0.3) * gradeAdjustment
  
  return {
    floor,
    average: avg,
    ceiling,
    recommended: Math.round(recommended * 100) / 100,
    confidence: activePrices.length >= 5 ? 'high' : activePrices.length >= 2 ? 'medium' : 'low',
    compCount: activePrices.length
  }
}
```

### Caching Strategy

- Cache pricing data per card (player + year + brand + card_number + grade) for 1 hour
- Cache key: `pricing:{sport}:{player_slug}:{year}:{brand}:{card_number}:{grade}`
- On cache hit, return cached data + note "data from X minutes ago"
- Invalidate cache on user request or after 1 hour

---

## 7. Database Schema

### Prisma Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
}

model User {
  id              String    @id @default(cuid())
  email           String    @unique
  passwordHash    String
  name            String?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  // eBay integration
  ebayUserId          String?   @unique
  ebayRefreshToken    String?   // encrypted
  ebayAccessToken     String?   // encrypted
  ebayTokenExpiresAt  DateTime?
  ebayRuName          String?

  // Business policies
  ebayFulfillmentPolicyId String?
  ebayPaymentPolicyId     String?
  ebayReturnPolicyId      String?

  // Stripe
  stripeCustomerId   String?
  stripeAccountId    String?   // for payouts

  // Notifications
  notifyEmail        Boolean  @default(true)
  notifySms          Boolean  @default(false)
  phoneNumber        String?

  cards     Card[]
  listings  Listing[]
  payments  Payment[]
}

model Card {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])

  // AI-extracted identification
  playerName      String
  year            Int
  brand           String
  brandLine       String?
  setName         String?
  cardNumber      String
  parallel        String?
  rookieCard      Boolean  @default(false)
  auto            Boolean  @default(false)
  patch           Boolean  @default(false)
  graded          Boolean  @default(false)
  grader          String?  // PSA, BGS, CGC
  grade           String?  // 10, 9.5, etc.
  certNumber      String?

  // Condition (for ungraded)
  condition        String?  // MINT, NEAR_MINT, EXCELLENT, GOOD, ACCEPTABLE

  // AI metadata
  aiConfidence     String?  // low, medium, high
  aiNotes          String?

  // Status
  confirmed        Boolean  @default(false)
  confirmedAt      DateTime?

  // Photos (eBay-hosted URLs after upload)
  photoUrls        String[]

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  listings         Listing[]

  @@index([userId])
  @@index([playerName, year, brand, cardNumber])
}

model Listing {
  id              String   @id @default(cuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])
  cardId          String
  card            Card     @relation(fields: [cardId], references: [id])

  // eBay fields
  ebayListingId   String?  @unique
  ebayItemId      String?
  ebaySku         String?
  listingUrl      String?

  // Listing details
  title           String
  description     String?
  price           Decimal  @db.Decimal(10, 2)
  quantity        Int      @default(1)

  // Status tracking
  status          ListingStatus @default(DRAFT)
  // DRAFT → PUBLISHED → ENDED → SOLD | RELISTED

  startsAt        DateTime?
  endsAt          DateTime?
  endedAt         DateTime?
  soldAt          DateTime?
  soldPrice       Decimal? @db.Decimal(10, 2)

  // Metrics
  views           Int      @default(0)
  watchers        Int      @default(0)
  offers          Int      @default(0)

  // Pricing data snapshot
  pricingData     Json?    // snapshot of comps used at listing time

  // Relist tracking
  relistCount     Int      @default(0)
  parentListingId String?  // if this is a relist of another listing

  // Stripe
  paymentId       String?
  payment         Payment? @relation(fields: [paymentId], references: [id])

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@index([userId])
  @@index([status])
  @@index([ebayListingId])
}

enum ListingStatus {
  DRAFT
  PENDING_PAYMENT
  PUBLISHED
  ENDED
  SOLD
  CANCELLED
  RELISTED
}

model Payment {
  id              String   @id @default(cuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])

  stripePaymentId String   @unique
  amount          Decimal  @db.Decimal(10, 2)
  currency        String   @default("usd")
  type            PaymentType

  // For listing payments
  listingId       String?
  listing         Listing? @relation(fields: [listingId], references: [id])

  // For subscription/relist
  description     String?

  status          String   @default("pending")

  createdAt       DateTime @default(now())
  @@index([userId])
}

enum PaymentType {
  LISTING_FEE      // $2.50
  RELIST_FEE       // $1.00
  SUBSCRIPTION     // future
}

model PricingCache {
  id          String   @id @default(cuid())

  // Cache key components
  sport       String
  playerSlug  String   // lowercase, slugified
  year        Int
  brand       String
  cardNumber  String
  grade       String?

  // Cached data
  data        Json     // full pricing response
  fetchedAt   DateTime @default(now())
  expiresAt   DateTime

  @@unique([sport, playerSlug, year, brand, cardNumber, grade])
  @@index([expiresAt])
}

model EbayNotification {
  id          String   @id @default(cuid())
  topic       String   // ITEM_ENDED, ITEM_SOLD, etc.
  payload     Json
  processed   Boolean  @default(false)
  createdAt   DateTime @default(now())

  @@index([topic, processed])
}
```

---

## 8. Authentication

### Registration / Login

- **Email/password** with bcrypt hashing
- JWT access tokens (1 hour expiry) + refresh tokens (7 days)
- Refresh tokens stored in HTTP-only cookies
- Password reset via email link

### OAuth

- Optional sign-in with Google
- Same JWT flow after OAuth confirmation

### eBay Connection (Separate OAuth)

- User connects their personal eBay seller account
- Separate OAuth flow for eBay (not site OAuth)
- Refresh tokens stored encrypted at rest
- "Disconnect eBay" button immediately invalidates token
- Users can only connect one eBay account at a time (MVP)

### Permission Transparency

Clearly display what access the app has when connecting eBay:
- Create and manage listings
- View account information
- No access to payment methods, banking, or personal data

---

## 9. Stripe Integration

### Pricing

| Action | Amount |
|--------|--------|
| Create listing | $2.50 |
| Auto-relist | $1.00 |

### Flow

1. User clicks "Post to eBay"
2. Frontend shows price confirmation: "$2.50 to list this card"
3. User confirms → frontend calls `/listings/create` with Stripe payment intent
4. Backend creates Stripe PaymentIntent ($2.50)
5. Frontend collects payment via Stripe Elements
6. On payment success → listing goes live
7. On payment failure → show error, don't list
8. Webhook confirms payment completion async

### Auto-Relist Payment

- When a listing ends without a sale, user can enable auto-relist for $1
- Separate PaymentIntent for relist fee
- Relist fires only after payment confirmed

### Payouts (Future)

- In v1, Stripe is used only for collecting fees
- Future: user can connect Stripe for automatic payouts of sale proceeds
- For now, eBay handles all buyer payments directly

---

## 10. Notifications

### Triggers

| Event | Notification |
|-------|-------------|
| Card sells | Email + SMS: "Your [Card Name] sold for $XX!" |
| Listing ends unsold | Email: "Your [Card Name] didn't sell. Relist for $1?" |
| Listing gets offer | Email: "You received a $XX offer on [Card Name]" |
| Auto-relist fired | Email: "[Card Name] has been relisted automatically" |
| Identification complete | Email (optional): "We identified your card — tap to approve" |

### Delivery

- **Email**: Resend or SendGrid. HTML templates with branding.
- **SMS**: Twilio for critical alerts (sold, offer received). Opt-in only.
- **In-app**: Notification center in dashboard for all events

---

## 11. MVP Scope

### v1 — "Single Card, Single Session" (2–4 weeks)

**What ships:**
- [ ] User can create account and log in
- [ ] User can connect their eBay seller account (OAuth)
- [ ] User can upload 1 photo of a card
- [ ] GPT-4o vision identifies the card (player, year, brand, card #)
- [ ] User confirms or corrects identification
- [ ] System fetches pricing comps from eBay Browse API
- [ ] System shows recommended price (with floor/avg/ceiling)
- [ ] User can adjust price before posting
- [ ] System creates + publishes listing via eBay Inventory API
- [ ] User gets link to live listing
- [ ] User is charged $2.50 via Stripe
- [ ] User can see active listings in dashboard
- [ ] User receives email when listing ends or sells

**What's NOT in v1:**
- Batch processing (multiple cards at once)
- Auto-relist automation
- Graded card cert lookup
- Price trend analysis
- Mobile-optimized photo capture flow
- Cross-marketplace listing (Whatnot, StockX)
- User-to-user messaging

### v2 — "Real Product" (4–6 weeks after v1)

- [ ] Batch upload (up to 10 cards at once)
- [ ] Auto-relist with $1 fee trigger
- [ ] PSA cert number verification (for graded cards)
- [ ] Listing analytics (views, watchers, offer trend)
- [ ] Relisting workflow with pricing update suggestion
- [ ] SMS notifications (Twilio)

### v3 — "Smarts & Scale" (future)

- [ ] Price trend analysis (card trending up/down)
- [ ] Optimal listing timing
- [ ] Cross-listing to Whatnot, StockX
- [ ] Mobile app (React Native or PWA)
- [ ] Subscription tier for high-volume sellers

---

## 12. Feature Priorities

### Must Have (v1)
1. Card photo upload + AI identification
2. Human confirmation step
3. eBay Browse API comp pricing
4. Listing creation via Inventory API
5. Stripe payment collection
6. Email notifications (sold/ended)
7. eBay OAuth onboarding

### Should Have (v2)
1. Auto-relist workflow
2. Batch upload
3. PSA cert lookup
4. Listing analytics dashboard
5. SMS notifications

### Nice to Have (v3)
1. Price trend analysis
2. Cross-marketplace listing
3. Mobile app
4. Subscription tier

---

## 13. Pricing Model

### Flat Fee Per Listing

| Fee Type | Amount |
|----------|--------|
| Create listing | $2.50 |
| Auto-relist | $1.00 |

**How it works:**
- User pays $2.50 to create a listing
- If the card doesn't sell and the user enables auto-relist, $1 to relist
- No commission on sale proceeds — user keeps everything minus eBay's standard fees (~10–15% + $0.35)
- No subscription required — pay per listing

**Why flat fee:**
- Simple, predictable, no surprises
- Aligns with eBay's insertion fee structure
- For a card selling at $20–30, $2.50 is ~8–12% of the sale price — reasonable for the convenience
- No subscription commitment for casual sellers

**Revenue math:**
- 100 parents/month × 30 cards each = 3,000 listings
- 3,000 × $2.50 = $7,500/month revenue
- No inventory, no shipping, pure SaaS margin

---

## 14. Competitive Analysis

### Landscape

| Competitor | What They Do | Gap |
|------------|-------------|-----|
| Mass buylist services (Bleecker, Media Mailbox) | Buy collections at lowball prices | User sells TO them, not ON eBay |
| Turbo Lister (eBay native) | Bulk manual listing tool | No AI, no pricing, full manual data entry |
| CardMinder / Collector apps | Card inventory tracking | No eBay integration |
| PSA Card Tracker | Grading lookup + pricing | No listing capability |
| DeckTrade | Connects to eBay for trading | Primarily for card trades, not cash sales |
| CardListAI | AI listing tool | Similar concept, unclear if live; not focused on casual sellers |
| Charizard / Dealer tools | Dealer-focused bulk tools | Complex UX, professional dealer focus |

### Our Position

**Nobody is doing "photo → live eBay listing with AI vision + automated pricing + relisting for casual sellers."** This is genuinely open territory.

**Competitive advantages:**
- **AI vision card identification** is the hardest part. Blurry kid photos + correct identification is a real technical challenge.
- **UX built for non-experts**: Every other tool assumes you already know what you have.
- **Full automation pipeline**: Photo in, listing live, sold notification out.
- **Auto-relist**: Captures the forgotten-listing problem.

### Risks
- eBay could build it themselves
- Professional dealers might not trust AI identification
- Price accuracy is hard — we need strong confidence indicators

---

## 15. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Card misidentification | Medium | High | Human approval step before anything goes live. Show confidence level. Ask for back-of-card photo or cert number for graded cards. |
| Pricing inaccuracy | Medium | Medium | Show confidence indicators. Display floor/avg/ceiling clearly. Let user adjust before posting. Never promise "best price." |
| eBay API restriction | Low | High | Stay within rate limits. Follow eBay terms. Don't bulk-post identical listings. Maintain good standing with eBay Developer Program. |
| OAuth token loss | Medium | Medium | Reconnect eBay flow always available. Store tokens encrypted. Monitor expiry. |
| Stripe payment failure | Low | Low | Show clear error messages. Retry logic. Support contact for stuck payments. |
| User doesn't trust the AI | High | Medium | Show the AI's reasoning (what comps it found, why it picked that price). Human-in-the-loop is the trust builder. |
| Market volatility | High | Low | Never predict prices. Use current data only. Show "data from X time ago." |
| eBay blocks listings | Low | Medium | Use correct category, proper item specifics, real photos. Don't try to game anything. |
| Competition copies us | Medium | Medium | Move fast. Build user love. Card identification accuracy is the hard part — make that the moat. |

---

## 16. Environment Variables

### Backend

```env
# Database
DATABASE_URL="postgresql://user:pass@host/db?sslmode=require"

# JWT
JWT_SECRET="super-secret-jwt-key"
JWT_REFRESH_SECRET="super-secret-refresh-key"

# eBay API
EBAY_CLIENT_ID="your-ebay-client-id"
EBAY_CLIENT_SECRET="your-ebay-client-secret"
EBAY_RU_NAME="your-ebay-ru-name"
EBAY_SANDBOX_CLIENT_ID="your-sandbox-client-id"
EBAY_SANDBOX_CLIENT_SECRET="your-sandbox-client-secret"
EBAY_SANDBOX_RU_NAME="your-sandbox-ru-name"
EBAY_ENVIRONMENT="sandbox"  # or "production"

# OpenAI
OPENAI_API_KEY="sk-..."

# Stripe
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
STRIPE_PUBLISHABLE_KEY="pk_test_..."

# Twilio (SMS)
TWILIO_ACCOUNT_SID="AC..."
TWILIO_AUTH_TOKEN="..."
TWILIO_PHONE_NUMBER="+1..."

# Email (Resend)
RESEND_API_KEY="re_..."
EMAIL_FROM="listings@ebaycardlister.com"

# App
FRONTEND_URL="http://localhost:5173"
PORT=3001
```

### Frontend

```env
VITE_API_URL="http://localhost:3001"
VITE_STRIPE_PUBLISHABLE_KEY="pk_test_..."
```

---

## 17. Build Sequence

If you're building this, here's the order to do it in:

**Week 1–2: Foundation**
1. Set up eBay Developer account, get sandbox keys
2. Set up Neon PostgreSQL + Prisma schema
3. Build Fastify backend skeleton with auth routes
4. Build React frontend skeleton with login/signup
5. Connect eBay OAuth (even just manual token storage for now)
6. Set up Stripe test mode

**Week 2–3: Core Loop**
1. Photo upload endpoint + GPT-4o vision card identification
2. Frontend: upload photo → show identification → confirm
3. Pricing engine: eBay Browse API integration + caching
4. Frontend: show pricing data + listing preview
5. Stripe payment collection before listing

**Week 3–4: eBay Listing**
1. Media API photo upload to eBay
2. Inventory Item creation
3. Offer creation + publish
4. Listing URL return to user
5. Basic dashboard showing active listings

**Week 4: Polish**
1. Email notifications (sold, ended)
2. Error handling + retry logic
3. Basic testing
4. README + documentation

---

*Spec version: 1.0 | eBay Card Lister | Flume SaaS Factory | March 2026*
