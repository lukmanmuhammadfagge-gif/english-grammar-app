# 13. CLEAN ACCESS-RULE MATRIX (v2: corrected)

This document, `src/accessService.js` and `test/*.test.js` implement exactly the same rules.
If you change one, change all three. `test/matrix.test.js` checks every cell below automatically.

## Changes from v1

| # | Issue in v1 | Resolution |
|---|-------------|-----------|
| 1 | Cancelled subscription had no defined behaviour | A CANCELLED subscription stays **ACTIVE until `period_end`**, then is **EXPIRED** (boundary: at `now >= period_end` it is expired) |
| 2 | Premium + Purchase: Free Student shown 🔒 although a paid purchase grants access | Free Student: ✅ if `ebook_purchase.status = PAID`. Active Premium: ✅. Expired Premium: ✅ only with a PAID purchase |
| 3 | Ebook purchased, then re-classified as Premium Included: undefined | A PAID purchase is a **permanent entitlement** and survives access-type changes. Only a REFUND revokes it |
| 4 | Read/download rules ambiguous for Expired Premium and for toggles | Read/download needs **both** (a) entitlement and (b) the ebook's own read/download switch ON |
| 5 | Matrix C showed Guest = 🔒 for paid ebooks, Matrix A showed ❌ | Guests are ❌ (login/register screen) everywhere. 🔒 (upgrade/buy/renew) is only for signed-in users |
| 6 | Matrix C had no "Purchased User" result for Premium Included | Purchased User = ✅ (preserved purchase) |

## A. CONTENT ACCESS MATRIX

| Content Type | Guest | Free Student | Active Premium | Expired Premium | Admin |
|---|---|---|---|---|---|
| Public Homepage | ✅ | ✅ | ✅ | ✅ | ✅ |
| Free Courses | Preview only | ✅ | ✅ | ✅ | ✅ |
| Free Lessons | ❌* | ✅ | ✅ | ✅ | ✅ |
| Premium Courses | ❌ | 🔒 | ✅ | 🔒 | ✅ |
| Premium Lessons | ❌ | 🔒 | ✅ | 🔒 | ✅ |
| Free Quizzes | ❌* | ✅ | ✅ | ✅ | ✅ |
| Premium Quizzes | ❌ | 🔒 | ✅ | 🔒 | ✅ |
| Free Ebooks | ❌* | ✅ | ✅ | ✅ | ✅ |
| Premium-Included Ebooks | ❌ | 🔒 (✅ if purchased before, see 3) | ✅ | 🔒 (✅ if purchased before) | ✅ |
| Individually Purchased Ebooks (Purchase Required) | ❌ | ✅ if PAID, else 🔒 | ✅ if PAID, else 🔒 | ✅ if PAID, else 🔒 | ✅ |
| Student Progress | ❌ | ✅ own | ✅ own | ✅ own | View/manage all |
| Subscription Page | ❌ | ✅ | ✅ | ✅ | ✅ |
| Admin Dashboard | ❌ | ❌ | ❌ | ❌ | ✅ |

\* Guests may open a **FREE** lesson/quiz only if `is_public_preview = 1`, and a **FREE** ebook only if `is_public = 1`. A Premium lesson flagged `is_public_preview` is still ❌ for guests (its basic info may be previewed, its content may not). Admin controls these flags.

"Free Student" = signed-in user whose effective subscription is FREE or PENDING.
"Active Premium" = effective subscription ACTIVE (including CANCELLED before `period_end`).
"Expired Premium" = effective subscription EXPIRED (including CANCELLED at/after `period_end`).

## B. ACCESS SYMBOLS

- ✅ Full access
- 🔒 Locked: signed in, show an upgrade / purchase / renew option
- ❌ No access (guests: show login/register screen)
- "Preview only": basic info visible, protected content not

## C. EBOOK ACCESS MATRIX (entitlement to read; see D for the switches)

| Ebook Access Type | Guest | Free Student | Active Premium | Expired Premium | Purchased User (PAID) | Admin |
|---|---|---|---|---|---|---|
| FREE | ✅ if `is_public`, else ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| PREMIUM_INCLUDED | ❌ | 🔒 | ✅ | 🔒 | ✅ (preserved purchase) | ✅ |
| PURCHASE_REQUIRED | ❌ | 🔒 | 🔒 unless PAID | 🔒 unless PAID | ✅ | ✅ |
| PREMIUM_PLUS_PURCHASE | ❌ | ✅ if PAID, else 🔒 | ✅ | ✅ if PAID, else 🔒 | ✅ | ✅ |

A PENDING, FAILED or REFUNDED purchase never grants access.

## D. READING AND DOWNLOAD MATRIX

Reading and downloading are separate permissions. Each requires **both**:
1. the user is entitled to the ebook (matrix C), **and**
2. the ebook's own switch is ON: `read_enabled` for reading, `download_enabled` for downloading.

| Entitled? | `read_enabled` | `download_enabled` | Read | Download |
|---|---|---|---|---|
| yes | ON  | ON  | ✅ | ✅ |
| yes | ON  | OFF | ✅ | ❌ `DOWNLOAD_DISABLED` |
| yes | OFF | ON  | ❌ `READING_DISABLED` | ✅ |
| yes | OFF | OFF | ❌ | ❌ |
| no  | any | any | 🔒/❌ | 🔒/❌ (a switch being ON never grants access) |

* Guests additionally need `public_download_enabled = 1` to download a public FREE ebook.
* Expired Premium users keep reading/downloading only what they are entitled to (FREE or PAID purchases), still subject to the switches.
* **Admin** has full access and is not blocked by the switches (admin override, section J). This is the one place the switches do not apply; change `evalEbook` if you want admins subject to them too.
* A switch being OFF returns a plain ❌ (no upgrade prompt, since paying would not help).

## E. LESSON ACCESS RULES

```
IF lesson.access_type = FREE      -> allow Free Student, Active Premium, Expired Premium, Admin
IF lesson.access_type = PREMIUM   -> allow Active Premium, Admin; deny Free Student, Expired Premium
Guests only if lesson.access_type = FREE AND lesson.is_public_preview = 1; otherwise show login/register.
```
Quizzes follow the same rules. A lesson/quiz inside an unpublished course is hidden from non-admins.

## F. SUBSCRIPTION ACCESS RULE

| Stored status | Effective status | Premium content |
|---|---|---|
| ACTIVE (verified, `now < period_end`) | ACTIVE | unlocked |
| ACTIVE (`now >= period_end`) | EXPIRED | locked |
| CANCELLED, `now < period_end` | ACTIVE | **unlocked until the paid period ends** |
| CANCELLED, `now >= period_end` | EXPIRED | locked |
| EXPIRED | EXPIRED | locked |
| PENDING | PENDING | locked |
| FREE / none | FREE | locked |

Premium is never granted because a payment was *initiated*. Only the signed payment webhook, after matching the reference, amount and currency, can mark a subscription ACTIVE or a purchase PAID. The database refuses to store ACTIVE/CANCELLED without `verified_at` and `period_end`. Malformed data fails closed (EXPIRED).

## G. EBOOK PURCHASE RULE

```
PURCHASE_REQUIRED      : access if ebook_purchase.status = PAID
PREMIUM_INCLUDED       : access if subscription effective = ACTIVE
                         OR ebook_purchase.status = PAID   (purchase made before it became Premium Included)
PREMIUM_PLUS_PURCHASE  : access if subscription effective = ACTIVE OR ebook_purchase.status = PAID
FREE                   : any signed-in user (guests only if is_public)
```

## H. LOCKED CONTENT EXPERIENCE

| Situation | Screen | Buttons |
|---|---|---|
| Premium content, Free Student | 🔒 **Premium Content**: "This lesson is available to active Premium members." | Upgrade to Premium · View Plans |
| Purchase-required ebook | 🔒 **Purchase Required**: "This ebook is available after purchase." | Buy Ebook |
| Expired subscription | **Your Premium subscription has expired.** "Renew your subscription to continue accessing Premium content." | Renew Premium |
| Guest | **Sign in required**: "Please log in or register to continue." | Log in · Register |

For Premium + Purchase ebooks the Premium/Renew screen also shows **Buy Ebook**.

## I. BACKEND AUTHORIZATION

Every protected route calls `guard()` -> `accessService.decide()` before any content or file is read. The service itself checks:
1. authenticated (session token -> user; stale/unknown user = guest) 2. role (re-read from DB) 3. content exists 4. publication status 5. content access type 6. effective subscription 7. purchase status 8. read/download switch. Direct URLs and direct downloads go through the same path as the UI. Frontend hiding is cosmetic only.

HTTP mapping: signed-out -> `401`, locked -> `403` `{locked:true, screen}`, forbidden/switch off -> `403`, unpublished/missing -> `404`.

## J. ADMIN OVERRIDE

Admins can access all educational content regardless of free/premium, subscription, purchase, publication status and the read/download switches. Unpublished content is invisible to everyone else (404).

## K. SOURCE OF TRUTH

All of these live in the database and are read fresh on every decision: content `access_type`, ebook `access_type`, `is_public_preview` / `is_public`, `read_enabled`, `download_enabled`, `public_download_enabled`, subscription status + `period_end`, purchase status. There is one authorization module: `src/accessService.js` (`decide(user, resource, action)` and `canAccess(user, resource, action)`).

## L. TEST SCENARIOS (all passing)

Original 15 (S1-S15) plus the new required ones: N1 admin views unpublished, N2 cancelled keeps Premium before `period_end`, N3 cancelled loses Premium after `period_end`, N4 free student with PAID purchase, N5 expired Premium with PAID purchase, N6 download disabled while reading allowed. Also: rule-3 preserved purchase (R3), exhaustive matrix C/D cross-product, payment-webhook flows, path traversal, role-injection and fail-closed checks. Run `npm test`.
