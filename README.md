# English Grammar App: backend + access control

Zero-dependency Node.js (22.5+) app: HTTP API, SQLite (`node:sqlite`), minimal web UI, and the centralized access-control service from Section 13.

## Run
```bash
npm run seed                                   # creates data/app.db + demo ebooks in storage/
export PAYMENT_WEBHOOK_SECRET=$(openssl rand -hex 32)
npm start                                      # http://localhost:3000
npm run content                                # loads English–Hausa Grammar Book 1 (10 lessons, 3 quizzes, 1 ebook); safe to re-run
npm test                                       # 91 tests
```
Demo logins (password `password123`): `admin`, `free`, `premium`, `expired`, `cancelledIn`, `cancelledOut`, `pending` @example.com

## Layout
| File | Purpose |
|---|---|
| `docs/SECTION-13-ACCESS-RULES.md` | The corrected rule matrix (spec) |
| `docs/DESIGN-SYSTEM.md`, `public/design-system.css` | Lukuman Learning light/dark usage rules + tokens (enforced by `test/design.test.js`) |
| `src/schema.sql` | Database = source of truth (CHECK constraints stop unverified Premium) |
| `src/accessService.js` | **The only place rules live**: `decide()` / `canAccess(user, resource, action)` |
| `src/server.js` | API; every protected route calls `guard()` before touching content or files |
| `public/index.html`, `public/app.js` | The pages: home, signup, login, dashboard, course, lesson, quiz, ebooks, subscription, profile, admin (hash-routed; UX only) |
| `src/seed.js` | One persona per matrix column + sample content |
| `test/matrix.test.js` | Every matrix cell + all read/download switch combinations |
| `test/scenarios.test.js` | Section 13-L scenarios + new cases + hardening |
| `src/content/english-hausa-book-1.js`, `src/loadContent.js` | Book 1 content (edit the Hausa here or in Admin) and the idempotent loader |
| `test/content.test.js` | Book 1 integrity + access rules, admin create/edit, validation |
| `test/pages.test.js` | Signup, profile, quiz grading (answer key never leaked), dashboard, admin overview |
| `test/api.test.js` | End-to-end over HTTP: direct URLs, direct downloads, payments |

## Using the service
```js
const access = createAccessService(db);
access.canAccess(user, { type: 'ebook', id: 7 }, 'download');   // boolean
access.decide(user, { type: 'lesson', id: 3 }, 'view');         // { allowed, outcome, reason, tier, screen }
```
Resources: `homepage course lesson quiz ebook progress subscription_page admin_dashboard`.
Actions: `preview view` (content) · `preview read download` (ebooks) · `view update manage` (progress) · `view manage` (admin).

## Payments with Paystack
Premium and individual ebooks are paid on Paystack's hosted checkout. Access is granted **only after the server confirms the payment with Paystack's verify API**: a redirect or a webhook alone never unlocks anything.

Go-live checklist:
1. Copy `.env.example` to your environment and set `PAYSTACK_SECRET_KEY`, `APP_BASE_URL`, `PREMIUM_PRICE_MINOR`, `PAYMENT_CURRENCY` (the app refuses to start with a Paystack key unless all are set; live keys require https).
2. In the Paystack dashboard set the webhook URL to `<APP_BASE_URL>/api/webhooks/paystack`.
3. Test end to end with `sk_test_` keys and Paystack test cards before switching to `sk_live_`.
4. Ebooks use the currency stored on each ebook (new ebooks default to `PAYMENT_CURRENCY`); it must also be enabled on your Paystack account.

How it works: `POST /api/subscription/initiate` (or `/api/ebooks/:id/purchase`) creates a PENDING payment and returns a Paystack `authorization_url`; the browser goes there; Paystack calls the signed webhook (HMAC-SHA512) and redirects the customer to `/payment/callback`, which calls `POST /api/payments/verify`. Both paths run the same race-safe settlement, so a payment is applied exactly once. Amount and currency must match what the server charged. Without `PAYSTACK_SECRET_KEY` the app runs in development mode with a signed dev webhook.

## Production notes (not implemented here)
Rate limiting, HTTPS/cookie sessions, email verification, password reset, and object storage/signed URLs for ebook files.
