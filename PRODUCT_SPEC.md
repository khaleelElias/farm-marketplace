# Farm Marketplace — Product Specification

## Overview
A React Native marketplace app that makes local farms discoverable to the general public. No payment processing — pure discovery and connection. Farmers list their products, opening hours, and contact info. Consumers find nearby farms filtered by what they sell.

---

## User Types

| Role | Description |
|---|---|
| **Consumer** | General public. Browse freely, account optional (needed only for favourites). |
| **Farmer** | Signs up via same app, switches to farmer mode after login. One account per farm. |
| **Admin** | Manages the platform via a separate **web dashboard** (not in the mobile app). |

---

## Consumer Experience

### Home / Discovery
- Opens to a **map + list toggle view** (default: list sorted by distance)
- Each farm card shows: farm name, distance (km), main product categories, open/closed badge
- **Filter by product category** on both map and list views
- Location permission used to sort by proximity (closest farms first)

### Farm Profile Page
- Farm name + cover photos (multiple)
- **Opening hours** (weekly schedule)
- **"Currently Closed" badge** when farm is in temporary closure mode
- **Product listings**: name, price, free-text description, in-stock / out-of-stock badge
- **Accepted payment methods** (cash, card, Swish, etc.)
- **Contact info**: phone number and/or email (no in-app messaging)
- Distance from user's current location
- Thumbs up / Save as favourite button

### Account (optional)
- Browse without account — no friction
- Account required to: save favourites, receive notifications
- Auth: email/password or social login

### Notifications (opt-in)
- A favourite farm adds a new product
- A new farm joins the platform nearby

---

## Farmer Experience (Farmer Mode)

### Farmer Mode Toggle
- Same app as consumer — a toggle/switch in profile that reveals farmer management screens
- Farmer account must be approved by admin before farm is live

### Farm Profile Management
- Upload farm name, description, photos (farm, animals, products)
- Set weekly opening hours
- Set contact info (phone, email)
- Set accepted payment methods
- Mark farm as temporarily closed (shows "Currently Closed" badge, farm remains discoverable)
- List in their own language — buyers see auto-translated version

### Product Management
- Add / edit / remove products
- Each product: name, price, free-text description, category tag
- **Manual in-stock / out-of-stock toggle** per product
- Product categories: Fresh Produce (veg, fruit, dairy, meat, eggs) and Processed Goods (jams, honey, cheese, bread)

### Farmer Notifications
- Received a new thumbs up / favourite
- Admin alerts: listing approved, violation warnings

---

## Admin Web Dashboard (Separate Web App)

| Feature | Description |
|---|---|
| Farmer applications | Review, approve, or reject new farmer sign-ups |
| Content moderation | Edit or remove listings that violate guidelines |
| User management | Ban/suspend farmers or consumer accounts |
| Analytics | Active farms, user counts, top regions, growth trends |

---

## Technical Stack

| Layer | Technology |
|---|---|
| Mobile app | React Native (iOS + Android) |
| Backend & Auth | Supabase (PostgreSQL + Auth + Storage) |
| Maps | Mapbox + OpenStreetMap |
| Geo queries | PostGIS (via Supabase) for distance sorting |
| Translations | i18n for UI + Translation API for farm listings |
| Admin dashboard | React web app connected to same Supabase backend |
| Push notifications | Expo Notifications or Firebase Cloud Messaging |

---

## Design Direction

- **Audience**: General public including older adults — accessibility is a priority
- **Principles**: Large readable text, high contrast, simple navigation, minimal cognitive load
- **Feel**: Clean and natural — warm, approachable, not cluttered
- **Language**: App UI fully internationalised from day one

---

## Out of Scope for v1 (Planned for Future)

- Farm tools and equipment listings
- Agricultural inputs (manure, seeds, soil, feed)
- Farm experiences / agritourism (pick-your-own, visits)
- Multi-user farm account management
- In-app messaging
- Reservations / pre-orders

---

## Monetization Roadmap (Not in v1)

- **Premium farmer tier**: Featured listing placement + analytics dashboard
- **Sponsored placements**: Ads from agri-suppliers, seed companies, equipment brands

---

## Key Design Screens to Produce

1. **Onboarding / splash**
2. **Consumer home** — list view (farm cards, distance, category filter bar)
3. **Consumer home** — map view (pins, farm card on tap)
4. **Farm profile page** — full detail view
5. **Product listing section** on farm profile
6. **Consumer account / favourites**
7. **Farmer mode — dashboard** (my farm overview)
8. **Farmer mode — edit farm profile**
9. **Farmer mode — product management** (add/edit/remove)
10. **Farmer sign-up / application form**
11. **Admin web dashboard** — applications queue
12. **Admin web dashboard** — analytics overview

---

## Design Notes for Designer

- **Accessibility is non-negotiable** — large tap targets, high contrast, readable fonts for older users
- **The farmer and consumer experiences live in one app** — the mode switch needs to feel seamless but clearly separated
- **Distance is the #1 consumer signal** — it should be the most prominent piece of info on every farm card
- **"Currently closed" farms should still be discoverable** — just clearly badged, not hidden
