# oMesh Marketplace — Public Technical Overview

**oMesh** is a production-grade, AI-powered multi-vendor marketplace with integrated same-day delivery — built from the ground up as a fully self-hosted platform, without relying on third-party marketplace infrastructure.

This repository contains screenshots, UI previews, and a technical overview of the platform. No source code is published here.

> **Author:** Roland Kohlhuber  
> **Website:** [ome.sh](https://ome.sh)

---

## What is oMesh?

oMesh is a full-stack marketplace platform operating in Panama, combining:
- Multi-vendor e-commerce (buyer, seller, and admin roles)
- AI-powered semantic product search
- Real-time delivery tracking ("oMesh Lightning")
- Self-hosted AI infrastructure — no third-party dependency for core features
- Blockchain governance token
- Native iOS & Android app + React web app + Seller Academy

The platform handles the entire commerce lifecycle end-to-end: product discovery → checkout → payment → fulfillment → delivery — in a single integrated system.

---

## Architecture Overview

```
oMesh Platform
├── Backend API              ← Node.js/Express, 35,000+ lines, MariaDB + MongoDB + Weaviate
├── React Native App         ← iOS & Android, 41 screens, 50,000+ lines
├── React Web App            ← PWA, React 19
├── Seller Academy           ← React TypeScript, onboarding & education platform
├── AI Services              ← Self-hosted embeddings, LLM agent, STT/TTS (Python gRPC)
└── Smart Contract           ← ERC20 governance token (Solidity)
```

---

## Backend

**Framework:** Node.js (Express, ES6 modules)  
**Primary database:** MariaDB (InnoDB, ACID-compliant, connection pooling with retry logic)  
**Document store:** MongoDB (sessions, metadata, analytics)  
**Vector database:** Weaviate (semantic search index, conversation memory)  
**Logging:** Pino (structured JSON)

The core API is a single production-hardened module (5,600+ lines) covering authentication, product catalog, search, orders, payments, delivery, subscriptions, admin operations, and background jobs — all with structured error handling and rate limiting throughout.

**Database schema highlights:**
- `users`, `sellers`, `sessions` — auth and identity, with `hash_id → acc_id` session isolation
- `products`, `categories`, `product_images` — catalog with 7 image slots per product, 3D model support
- `sells` / orders — full order lifecycle tracking with delivery status state machine
- `reviews`, `vouchers` — social proof and promotions
- `keyword_embeddings`, `product_embeddings` — pre-computed vectors cached in MariaDB to avoid regeneration on repeat queries
- `seller_subscriptions` — recurring billing state per seller
- `daily_stats` — aggregated revenue, GMV, active sellers
- `ethereum`, `arbitrum_one`, `fiat` — blockchain transactions and exchange rate tracking

---

## Search Engine

Hybrid semantic + full-text search with embedding caching and multi-language support:

- **Model:** Nomic Embed Text v2 MOE (768-dimensional vectors, self-hosted gRPC server)
- **Index:** Weaviate HNSW with cosine similarity
- **Embedding caching:** Pre-computed vectors stored in MariaDB — cold query ~180ms, cached ~50ms
- **Semantic ranking** with certainty scores (0–100% match percentage surfaced to users)
- **Multi-language:** queries translated to English/Spanish/Chinese before vectorization
- **Category-aware filtering** + stock and price constraints applied at the vector layer
- **Keyword tracking** — all user queries stored for analytics and autocomplete improvement

The search engine understands intent, not just keywords. A query for "something for a birthday dinner" surfaces the right products even with zero keyword overlap.

---

## AI Infrastructure (Self-Hosted)

All inference runs on self-hosted hardware. No raw data sent to third-party AI providers for core features.

| Capability | Implementation |
|-----------|----------------|
| **Semantic search embeddings** | Nomic Embed v2 MOE, self-hosted gRPC server |
| **AI Agent / Chatbot** | LangChain + LangGraph, OpenRouter (GPT OSS 120B) |
| **Conversation memory** | Weaviate (`ChatMessage` class, vector-indexed history) |
| **Company knowledge RAG** | Vector search over business info for grounded, hallucination-resistant answers |
| **Product listing generator** | LLM auto-generates titles and descriptions from images or text input |
| **Product recipe processor** | LLM extracts ingredients, instructions, and nutritional info |
| **Image search** | Vision model endpoint — find products by image |
| **Speech-to-text** | Python gRPC server — WhatsApp/Telegram voice message transcription |
| **Text-to-speech** | Python gRPC server |

---

## Payment Orchestration

oMesh integrates **5 payment methods** with unified webhook handling and a tokenization layer — raw card data is never stored on the platform:

| Provider | Use Case | Notes |
|---------|----------|-------|
| **BAC PowerTranz** | Primary (Panama) | 3DS2 auth, PAN tokenization, subscription billing via stored tokens |
| **Stripe** | International | Apple Pay, Google Pay, webhook charge events |
| **Yappy** | Panama mobile | QR code payments, instant push notifications |
| **PayPal** | Express Checkout | Standard integration |
| **Cash on Delivery** | Last-mile option | Driver confirms on handoff |

**Architecture:**
- PAN tokenization — card data goes directly to BAC, oMesh stores only the token
- 3DS2 challenge handling integrated into the checkout flow
- Webhook validation for all providers
- Subscription billing: recurring charges via stored BAC tokens (no customer re-authentication each cycle)
- Payment confirmation drives delivery state machine transitions

---

## Delivery System — oMesh Lightning

Real-time same-day and sub-hour delivery via distributed fulfillment centers.

**Pricing (dynamic, distance-based):**
- Base fare + per-km rate + per-minute rate
- Geofenced delivery radius validated via Haversine distance calculation

**Real-time tracking:**
- WebSocket server for live driver ↔ customer location streaming
- Google Maps API for distance matrix and route calculation
- Driver GPS updates every 5–10 seconds via background location service
- Live ETA calculation on the client
- Order state machine: `searching` → `picked_up` → `delivered`

**Mobile:**
- `LocationService.js` — background GPS polling (iOS + Android, permission-aware)
- `OMeshLightning.js` — live map UI with animated driver marker and ETA
- Address autocomplete via Google Places API

---

## Authentication & Security

**Authentication methods:**
- Email + password (bcrypt 12 rounds + SHA256 salt; legacy SHA256 accounts auto-migrated to bcrypt on next login)
- Google Sign-In (OAuth2)
- Apple Sign-In
- Facebook Login
- WhatsApp verification codes

**JWT architecture:**
- **RS256 asymmetric signing** (RSA 2048-bit private key)
- 6-month token expiration with device UUID embedded
- Firebase Cloud Messaging token stored per device for targeted push notifications
- Session table maps `hash_id → acc_id` for session isolation

**Rate limiting:**
- Global: 100 req/min per IP
- Login: 10 failed attempts per 15 min (successful logins exempt)
- `Retry-After` headers on throttled responses

**Data security:**
- PAN tokenization — no raw card data ever persisted
- bcrypt 12 rounds on all passwords
- RS256 signing — private key never leaves the server

---

## Mobile App (React Native)

**41 screens, 50,000+ lines**, built for iOS and Android — handling the full marketplace experience.

| Screen | Description |
|--------|-------------|
| `HomeScreen` (45KB) | Categories, trending products, personalized "for you" feed |
| `CheckoutScreen` (61KB) | Full checkout: address selection, payment method, order review |
| `ProfileScreen` (110KB) | Account, orders, wallet, vouchers, membership, subscription management |
| `ProductListingScreen` (41KB) | Seller dashboard — product upload with AI-assisted listing |
| `AgentScreen` (41KB) | AI chatbot with conversation memory |
| `OMeshLightning` (24KB) | Real-time delivery tracking with live driver map |
| Payment screens | BAC 3DS2 flow, Yappy QR payment |

**Technical highlights:**
- React Native 0.83.1 (latest stable)
- React Navigation v7 (bottom tabs + native stack)
- Vision Camera v4.7.2 + image picker for product photos
- Background GPS tracking via `react-native-background-actions`
- SQLite for local persistence and offline capability
- Google Sign-In, Apple Sign-In, Facebook integrated natively
- **i18n:** English, Spanish, Chinese (749 strings, full coverage)
- Built and maintained end-to-end by a single developer

---

## Blockchain & Decentralization

**oMesh Token (OMESH) — ERC20 Governance Token:**
- 200 million tokens minted at deployment (Solidity)
- Governance: proposal creation (1 OMESH fee), voting weighted by token holdings
- Batch transfer support (gas-optimized with unchecked arithmetic)
- Use cases: product pricing in crypto, affiliate rewards, on-chain order settlement

**IPFS Storage:**
- Self-hosted IPFS node + external mirror
- All product images stored on IPFS with CID versioning
- Hierarchical path structure for scalability: `/uploads/{a}/{b}/{c}/{CID}.webp`
- Automatic WebP conversion on upload
- Batch download system for catalog sync
- Fallback to `image1.ome.sh` gateway for web access

---

## Background Jobs & Automation

| Job | Schedule | Purpose |
|-----|----------|---------|
| `calculateDailyStats` | 02:00 AM daily | Revenue, orders, GMV aggregation |
| `processSubscriptionBilling` | 02:00 AM daily | Seller subscription renewal via BAC |
| `detectAbandonedCarts` | Periodic | Cart abandonment tracking |
| `sendAbandonedCartRecovery` | Periodic | Email/SMS recovery campaigns |
| `validateProductPhotos` | Periodic | Image validation & IPFS integrity check |
| `notifyUnpaidOrders` | Periodic | Reminders for pending payments |
| `calculate_financial_summary` | Monthly | P&L aggregation |

---

## Seller Ecosystem

- Self-service seller registration and verification
- Product management dashboard (upload, edit, bulk operations)
- AI-assisted product listing generator
- Order dashboard with fulfillment controls
- Subscription billing (recurring via tokenized card)
- Revenue analytics

**Seller Academy** — dedicated React TypeScript educational app:
- Course UI with progress tracking
- Certification system
- Structured onboarding flows

---

## Performance (Production)

| Metric | Value |
|--------|-------|
| Concurrent users | ~500 |
| Throughput | 200 req/sec |
| Search latency (cold) | ~180ms |
| Search latency (cached) | ~50ms |
| Product page | ~120ms |
| Checkout | ~200ms |
| Uptime (Nov–Dec 2025) | 99.7% |

---

## External Integrations

| Service | Purpose |
|---------|---------|
| Google Maps API | Distance matrix, directions, Places autocomplete |
| Firebase Cloud Messaging | Push notifications (iOS + Android) |
| WhatsApp | Auth codes, customer service, voice transcription |
| Telegram | Admin alerts, bot integration |
| Instagram | Webhook for DMs |
| Email (Nodemailer) | DKIM-signed HTML templates |
| Puppeteer | Invoice PDF generation, audio processing |
| IPFS | Decentralized image storage |

---

## Screenshots

*Screenshots and UI previews are in the `/screenshots` folder.*

---

## Contact

**Roland Kohlhuber**  
Founder & CTO  
[ome.sh](https://ome.sh)

---

*This repository is a public overview. Source code is private.*
