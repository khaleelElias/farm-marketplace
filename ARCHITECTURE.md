# Farm Marketplace — Architecture

## Overview

A monorepo containing three apps and shared packages, all backed by Supabase.

```
farm-marketplace/
├── apps/
│   ├── mobile/          # Expo (React Native) — consumer + farmer in one app
│   ├── web/             # Vite + React — admin dashboard
├── packages/
│   ├── types/           # Shared TypeScript types (no runtime code)
│   └── i18n/            # Shared locale files and key constants
├── supabase/
│   ├── functions/       # Edge Functions — the sole API layer
│   ├── migrations/      # SQL schema migrations (applied via Supabase CLI)
│   └── config.toml
├── design-system/       # Claude Design output — READ-ONLY reference, never edit
├── PRODUCT_SPEC.md
├── ARCHITECTURE.md
├── RULES.md
├── package.json         # pnpm workspace root
└── pnpm-workspace.yaml
```

---

## Stack

| Layer | Technology | Notes |
|---|---|---|
| Mobile app | Expo (managed workflow) + React Native | iOS + Android |
| Web admin | Vite + React 18 + TypeScript | SPA, no SSR needed |
| API | Supabase Edge Functions (Deno) | Only thing that touches the DB |
| Database | Supabase (PostgreSQL + PostGIS) | Geo queries for distance sorting |
| Auth | Supabase Auth | JWT tokens, email/password + social |
| Storage | Supabase Storage | Farm and product photos |
| Maps | Mapbox + OpenStreetMap | Mobile: `@rnmapbox/maps`, Web: `mapbox-gl` |
| State (mobile) | Zustand (global) + TanStack Query (server) | |
| State (web) | TanStack Query | |
| i18n | react-i18next | Shared locale files in `packages/i18n` |
| Testing | Vitest + React Testing Library | |
| Package manager | pnpm workspaces | |

---

## Auth & Authorization

### How it works

No RLS. The API layer owns all authorization logic.

```
Client (mobile or web)
  │
  │  1. Sign in via Supabase Auth (email/password or social)
  │     → receives JWT
  │
  │  2. Every API request:
  │     Authorization: Bearer <jwt>
  │
  ▼
Edge Function
  │
  │  3. Validate JWT:  supabase.auth.getUser(jwt)
  │  4. Look up role:  SELECT role FROM profiles WHERE id = user.id
  │  5. Authorize:     check role vs required permission
  │  6. Execute:       DB query via service_role client
  │
  ▼
Supabase DB (service_role key — server only, never sent to client)
```

### Roles

| Role | Who | Permissions |
|---|---|---|
| `consumer` | Default for all sign-ups | Read farms/products, save favourites |
| `farmer` | Consumer who applied and was approved | CRUD their own farm and products only |
| `admin` | Manually set in DB | Full access to all resources |

### Auth middleware pattern (Edge Functions)

Every protected function calls a shared `requireAuth` helper:

```ts
// supabase/functions/_shared/auth.ts
export async function requireAuth(req: Request, requiredRole?: Role) {
  const jwt = req.headers.get('Authorization')?.replace('Bearer ', '')
  if (!jwt) throw new AuthError(401, 'missing_token')

  const { data: { user }, error } = await supabase.auth.getUser(jwt)
  if (error || !user) throw new AuthError(401, 'invalid_token')

  const { data: profile } = await adminClient
    .from('profiles')
    .select('role')
    .eq('id', user.id)
    .single()

  if (requiredRole && profile?.role !== requiredRole) {
    throw new AuthError(403, 'insufficient_role')
  }

  return { user, profile }
}
```

---

## Database Schema

```sql
-- Extends auth.users, created on signup via trigger
CREATE TABLE profiles (
  id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  role        TEXT NOT NULL DEFAULT 'consumer', -- 'consumer' | 'farmer' | 'admin'
  display_name TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Farms (status controlled by admin)
CREATE TABLE farms (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id    UUID NOT NULL REFERENCES profiles(id),
  name        TEXT NOT NULL,
  description TEXT,
  address     TEXT NOT NULL,
  lat         FLOAT8 NOT NULL,
  lng         FLOAT8 NOT NULL,
  location    GEOGRAPHY(POINT, 4326) GENERATED ALWAYS AS (
                ST_SetSRID(ST_MakePoint(lng, lat), 4326)
              ) STORED,
  phone       TEXT,
  email       TEXT,
  status      TEXT NOT NULL DEFAULT 'pending', -- 'pending'|'approved'|'rejected'|'suspended'
  is_closed   BOOLEAN NOT NULL DEFAULT FALSE,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX farms_location_idx ON farms USING GIST (location);

-- Weekly opening hours (one row per day per farm)
CREATE TABLE farm_hours (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  farm_id       UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  day_of_week   INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6), -- 0=Mon, 6=Sun
  open_time     TIME,
  close_time    TIME,
  is_closed     BOOLEAN NOT NULL DEFAULT FALSE,
  UNIQUE (farm_id, day_of_week)
);

-- Payment methods a farm accepts
CREATE TABLE farm_payment_methods (
  farm_id UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  method  TEXT NOT NULL, -- 'cash' | 'card' | 'swish' | 'mobile_pay' | 'bank_transfer'
  PRIMARY KEY (farm_id, method)
);

-- Farm photos (ordered)
CREATE TABLE farm_photos (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  farm_id      UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  storage_path TEXT NOT NULL,
  sort_order   INT NOT NULL DEFAULT 0,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Product categories (seeded, not user-editable)
CREATE TABLE product_categories (
  id        TEXT PRIMARY KEY, -- 'vegetables' | 'fruit' | 'dairy' | 'meat' | 'eggs' | 'honey' | 'bread' | 'jams'
  label_key TEXT NOT NULL     -- i18n key e.g. 'categories.vegetables'
);

-- Products listed by a farm
CREATE TABLE products (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  farm_id     UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  category_id TEXT REFERENCES product_categories(id),
  name        TEXT NOT NULL,
  description TEXT,
  price       TEXT,            -- free text: "35 kr", "3 kr / egg"
  in_stock    BOOLEAN NOT NULL DEFAULT TRUE,
  photo_path  TEXT,
  sort_order  INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Consumer saved farms (favourites)
CREATE TABLE saved_farms (
  user_id    UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  farm_id    UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, farm_id)
);

-- Farmer application notes (submitted alongside the farm signup form)
CREATE TABLE farm_applications (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  farm_id      UUID NOT NULL REFERENCES farms(id) ON DELETE CASCADE,
  note         TEXT,
  categories   TEXT[],           -- initial categories indicated at signup
  submitted_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## API — Edge Functions

**5 Edge Functions total.** Each function contains an internal Hono router and handles
multiple routes. All live in `supabase/functions/`. Base URL: `https://<project>.supabase.co/functions/v1/`

```
supabase/functions/
├── _shared/
│   ├── auth.ts        # requireAuth helper — JWT validation + role check
│   ├── client.ts      # Supabase admin client (service_role)
│   └── errors.ts      # ApiError class + error response helpers
├── farms/             # Public farm discovery
│   └── index.ts
├── me/                # Consumer saved farms
│   └── index.ts
├── farmer/            # Farmer management (farm, products, photos)
│   └── index.ts
├── admin/             # Admin operations (applications, moderation, analytics)
│   └── index.ts
└── auth-hook/         # Supabase DB webhook — creates profiles row on signup
    └── index.ts
```

### Function: `farms` — Public (no auth required)
| Method | Path | Description |
|---|---|---|
| GET | `/farms` | List approved farms. Query: `lat`, `lng`, `radius_km`, `category`, `limit`, `offset` |
| GET | `/farms/:id` | Get farm detail (profile, hours, products, photos, payments) |

### Function: `me` — Consumer (any authenticated user)
| Method | Path | Description |
|---|---|---|
| GET | `/me/saved-farms` | List saved farms |
| PUT | `/me/saved-farms/:farmId` | Save a farm |
| DELETE | `/me/saved-farms/:farmId` | Unsave a farm |

### Function: `farmer` — Farmer (role = `farmer`)
| Method | Path | Description |
|---|---|---|
| POST | `/farmer/apply` | Submit farm application (pre-approval, any auth user) |
| GET | `/farmer/farm` | Get own farm data |
| PATCH | `/farmer/farm` | Update farm (name, description, address, contact, hours, payments, is_closed) |
| POST | `/farmer/products` | Add product |
| PATCH | `/farmer/products/:id` | Update product |
| DELETE | `/farmer/products/:id` | Delete product |
| PATCH | `/farmer/products/:id/stock` | Toggle in_stock |
| POST | `/farmer/photos` | Upload photo (multipart/form-data → Supabase Storage) |
| DELETE | `/farmer/photos/:id` | Delete photo |

### Function: `admin` — Admin (role = `admin`)
| Method | Path | Description |
|---|---|---|
| GET | `/admin/applications` | List applications (filterable by status) |
| POST | `/admin/applications/:id/approve` | Approve farm — sets `farms.status = approved`, `profiles.role = farmer` |
| POST | `/admin/applications/:id/reject` | Reject farm |
| GET | `/admin/farms` | List all farms |
| PATCH | `/admin/farms/:id` | Edit or suspend farm |
| DELETE | `/admin/farms/:id` | Remove farm |
| GET | `/admin/users` | List all users |
| POST | `/admin/users/:id/ban` | Ban user |
| GET | `/admin/analytics` | Platform stats |

### Function: `auth-hook` — Internal Supabase trigger
Fires on every new `auth.users` insert. Creates the corresponding `profiles` row
with `role = 'consumer'`. Not called by clients directly.

---

## Mobile App Structure (`apps/mobile/`)

Uses **Expo Router** (file-based routing).

```
apps/mobile/
├── app/
│   ├── _layout.tsx              # Root layout — auth gate + i18n init
│   ├── (tabs)/
│   │   ├── _layout.tsx          # Bottom tab bar
│   │   ├── index.tsx            # Discovery (list + map toggle)
│   │   ├── saved.tsx            # Saved farms
│   │   └── profile.tsx          # Account / farmer mode switch
│   ├── farm/
│   │   └── [id].tsx             # Farm profile detail
│   ├── farmer/
│   │   ├── _layout.tsx          # Farmer mode guard (role check)
│   │   ├── index.tsx            # Farmer dashboard
│   │   ├── edit.tsx             # Edit farm profile
│   │   ├── products.tsx         # Product list + stock toggle
│   │   ├── product/[id].tsx     # Edit/add product
│   │   └── apply.tsx            # Application form (pre-approval)
│   └── auth/
│       ├── login.tsx
│       └── register.tsx
├── components/
│   ├── ui/                      # Design system atoms (Button, Badge, Chip…)
│   ├── farm/                    # FarmCard, ProductRow, HoursTable…
│   └── farmer/                  # FarmerProductRow, StatTile, ActionRow…
├── hooks/
│   ├── useFarms.ts              # TanStack Query — farm list
│   ├── useFarm.ts               # TanStack Query — farm detail
│   └── useAuth.ts               # Auth state
├── lib/
│   ├── api.ts                   # Typed API client (fetch wrapper)
│   ├── supabase.ts              # Supabase client (auth only)
│   └── location.ts              # expo-location helpers
├── store/
│   └── authStore.ts             # Zustand — user + role
├── constants/
│   └── tokens.ts                # Design tokens (colors, spacing, radii) as JS constants
└── locales/ -> ../../packages/i18n/locales   # Symlink or tsconfig path alias
```

### Farmer mode guard

The `farmer/_layout.tsx` checks `authStore.role === 'farmer'` and redirects to `farmer/apply` if not yet a farmer, or to `profile` if not authenticated.

---

## Web Admin Structure (`apps/web/`)

```
apps/web/
├── src/
│   ├── main.tsx
│   ├── App.tsx                  # Route setup + auth guard
│   ├── pages/
│   │   ├── Applications.tsx
│   │   ├── Analytics.tsx
│   │   ├── Farms.tsx
│   │   ├── Moderation.tsx
│   │   ├── Users.tsx
│   │   └── Login.tsx
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   └── PageHeader.tsx
│   │   └── ui/                  # Shared atoms (Button, Badge, Chip…)
│   ├── hooks/
│   │   └── useAdmin.ts          # TanStack Query hooks for admin API
│   ├── lib/
│   │   ├── api.ts               # Typed API client
│   │   └── supabase.ts          # Supabase client (auth only)
│   └── locales/ -> ../../packages/i18n/locales
└── package.json
```

---

## Shared Packages

### `packages/types`

Shared TypeScript interfaces — no runtime code, imported by both apps and Edge Functions.

```
packages/types/
├── farm.ts      # Farm, FarmHours, FarmPhoto, FarmPaymentMethod
├── product.ts   # Product, ProductCategory
├── user.ts      # Profile, UserRole
└── api.ts       # Request/response envelope types
```

### `packages/i18n`

Locale JSON files shared between mobile and web. Both apps import from here.

```
packages/i18n/
├── locales/
│   └── en.json    # English base (source of truth)
└── keys.ts        # Type-safe key constants (auto-generated or hand-maintained)
```

Translation key structure:
```json
{
  "common": { "save": "Save", "cancel": "Cancel", "loading": "Loading…" },
  "auth": { "login": "Sign in", "logout": "Sign out", "login_error": "…" },
  "farm": { "open_until": "Open until {{time}}", "currently_closed": "Currently closed" },
  "categories": { "vegetables": "Vegetables", "dairy": "Dairy", … },
  "farmer": { "dashboard_title": "Your farm", "products_title": "Products" },
  "admin": { "applications_title": "Review queue", … },
  "validation": { "required": "This field is required", … },
  "errors": { "generic": "Something went wrong. Please try again.", … }
}
```

---

## Design Token Mapping

The design system CSS variables (`colors_and_type.css`) must be translated into:
- `constants/tokens.ts` in the mobile app (React Native StyleSheet-compatible values)
- CSS custom properties re-exported in the web app (copy the CSS file as-is)

```ts
// apps/mobile/constants/tokens.ts
export const Colors = {
  paper: '#F7F2E9',
  paperDeep: '#EFE7D6',
  ink: '#1F1A14',
  field: '#2F5D3A',
  harvest: '#C9621B',
  // …
} as const

export const Radii = {
  sm: 8, md: 12, lg: 16, xl: 24, pill: 999
} as const

export const Space = {
  1: 4, 2: 8, 3: 12, 4: 16, 5: 20, 6: 24, 7: 32, 8: 40
} as const
```

---

## Key Conventions

- **API responses** always use the shape `{ data: T } | { error: { code: string, message: string } }`
- **Distance** is always in km, computed server-side using PostGIS `ST_Distance`
- **Farm status** transitions: `pending → approved | rejected`, `approved → suspended`
- **Farmer role** is granted when admin approves the application — not at sign-up
- **Photos** are stored in Supabase Storage bucket `farm-photos`, public read, farmer-write only via signed upload URL from the API
- **Translations** — all user-facing strings use `t('key')`. Zero hardcoded strings anywhere
- **TypeScript** — `strict: true` in all tsconfigs. No `any`

---

## Environment Variables

```
# supabase/.env (Edge Functions)
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=   # server only — never sent to client

# apps/mobile/.env
EXPO_PUBLIC_SUPABASE_URL=
EXPO_PUBLIC_SUPABASE_ANON_KEY=
EXPO_PUBLIC_MAPBOX_TOKEN=

# apps/web/.env
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_MAPBOX_TOKEN=
VITE_API_BASE_URL=
```
