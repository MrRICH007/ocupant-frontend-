# Ocupant Backend

REST API for the Ocupant premium housing agency frontend.

## Stack
- Node.js + Express
- SQLite (via better-sqlite3) — no separate DB server needed
- JWT authentication, bcrypt password hashing

## Setup

```bash
cd ocupant-backend
npm install
cp .env.example .env        # then edit JWT_SECRET
npm run seed                # creates ocupant.db with demo data
npm start                   # API on http://localhost:4000
```

## Demo accounts (created by the seed)

| Role            | Email              | Password    |
|-----------------|--------------------|-------------|
| Admin           | admin@ocupant.com  | admin1234   |
| Premium user    | demo@ocupant.com   | demo1234    |
| Basic user      | alex@example.com   | password123 |

## API overview

| Method | Endpoint                        | Auth        | Description |
|--------|---------------------------------|-------------|-------------|
| POST   | /api/auth/register              | —           | Create account, returns JWT |
| POST   | /api/auth/login                 | —           | Login, returns JWT |
| GET    | /api/auth/me                    | Bearer      | Current user + plan status |
| GET    | /api/houses                     | optional    | List houses. Query: `search`, `type`, `status`, `minPrice`, `maxPrice`. Non-premium callers get generic location only; owner contact is omitted. |
| GET    | /api/houses/:id                 | optional    | House detail (same gating) |
| POST   | /api/houses/:id/inquire         | optional    | Log a contact-owner click |
| POST   | /api/subscription/upgrade       | Bearer      | Activate 30-day premium (see Payments) |
| POST   | /api/subscription/cancel        | Bearer      | Back to basic |
| GET    | /api/roommates/:houseId         | Bearer      | Roommate requests for a house |
| POST   | /api/roommates/:houseId         | Bearer      | Add/update your roommate request `{budget, message}` |
| POST   | /api/payments/flutterwave/initialize | Bearer | Start a Premium payment, returns `{ payment_link }` |
| GET    | /api/payments/flutterwave/callback   | —      | Flutterwave redirect target; verifies + credits, then redirects to the frontend |
| POST   | /api/payments/webhook             | verif-hash  | Flutterwave webhook (backup credit path) |
| GET    | /api/payments                   | Admin       | Payment history |
| POST   | /api/houses                     | Admin       | Create a listing |
| PUT    | /api/houses/:id                 | Admin       | Update a listing |
| DELETE | /api/houses/:id                 | Admin       | Remove a listing |

### Gating
Exact location and owner name/phone/WhatsApp are only returned when the
request carries a valid JWT for an account whose `premium_until` is in the
future. Basic users receive `genericLocation` in the `location` field and no
`owner` object — the same behaviour the current frontend simulates.

## Payments (Flutterwave)

Premium subscriptions are paid through **Flutterwave Standard** (hosted checkout):

1. Sign up at https://dashboard.flutterwave.com and get your **API keys**
   (Settings -> API Keys). Put them in `.env`.
2. Set `FRONTEND_URL` in `.env` to wherever `Ocupant.html` is served
   (default `http://localhost:3000`). The callback redirects there with
   `?payment=success|failed|cancelled`.
3. (Recommended) In the dashboard under **Settings -> Webhooks**, set the URL to
   `https://YOUR-BACKEND/api/payments/webhook` and choose the same secret for
   `FLUTTERWAVE_WEBHOOK_HASH`.
4. Adjust pricing with `PREMIUM_PRICE` and `PAYMENT_CURRENCY` in `.env`.

Flow: `POST /api/payments/flutterwave/initialize` -> user pays on Flutterwave ->
backend **verifies the transaction server-side** (amount + currency + status)
before granting 30 days of Premium. The webhook credits users who close the tab
before being redirected. Verification is idempotent, so double-callbacks are safe.

Test mode card: `5531 8866 5214 2950`, CVV `564`, expiry `09/32`, PIN `3310`, OTP `12345`.

`/api/subscription/upgrade` still exists as a manual override for testing.


## Admin panel
Serve `admin.html` from the same place as `Ocupant.html` and open it in a
browser. Log in with an admin account (seed: `admin@ocupant.com / admin1234`)
to add/edit/delete listings and view payment history. It talks to the same API.

## Frontend wiring
See FRONTEND_INTEGRATION.md for the exact changes to Ocupant.html.
(Already applied in the provided `Ocupant.html`.)
