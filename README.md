# Stripe payment integration (this project)

This document describes how **Stripe** is wired in this Laravel 11 app: Payment Intents, optional Stripe Checkout, webhooks, and database storage.

## Requirements

- PHP **8.2+**
- Laravel **11**
- Composer dependency: **`stripe/stripe-php`** (^20)
- MySQL (or any DB you configure); table: **`payments`**

## Install

```bash
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
```

Ensure Stripe variables exist in `.env` (see below).

## Environment variables

| Variable | Purpose |
|----------|---------|
| `STRIPE_KEY` | Publishable key (`pk_test_...` / `pk_live_...`) — used in the browser (checkout page). |
| `STRIPE_SECRET` | Secret key (`sk_test_...` / `sk_live_...`) — server-side API only. |
| `STRIPE_WEBHOOK_SECRET` | Webhook signing secret (`whsec_...`) — from Stripe Dashboard or Stripe CLI. |

Also set **`APP_URL`** to the same base URL you use in the browser (important for Stripe return URLs).

Configuration is read from `config/services.php` under the `stripe` key.

## CSRF and webhooks

Stripe sends **POST** requests to your webhook URL **without** a Laravel CSRF token. The route `POST /stripe/webhook` is excluded in `bootstrap/app.php` via `validateCsrfTokens(except: ['stripe/webhook'])`.  
Webhook authenticity is verified with **`Stripe-Signature`** and `STRIPE_WEBHOOK_SECRET` in `StripeWebhookController`.

## Routes

| Method | Path | Name | Description |
|--------|------|------|-------------|
| GET | `/payment` | `payment.checkout` | Checkout UI (Payment Element + optional hosted Checkout). |
| POST | `/payment/intent` | `payment.intent` | Creates a **PaymentIntent**, returns `clientSecret`. |
| POST | `/payment/confirm` | `payment.confirm` | Server-side check after client confirms payment. |
| GET | `/payment/success` | `payment.success` | Success page (`?payment_intent=pi_...`). |
| POST | `/payment/checkout-session` | `payment.checkout.session` | Creates **Stripe Checkout** session URL. |
| GET | `/payment/checkout/success` | `payment.checkout.success` | Return URL after hosted Checkout (`?session_id=cs_...`). |
| GET | `/payment/checkout/cancel` | `payment.checkout.cancel` | User cancelled hosted Checkout. |
| POST | `/stripe/webhook` | `stripe.webhook` | Stripe events → sync DB. |

## Project files (Stripe-related)

| Path | Role |
|------|------|
| `app/Http/Controllers/PaymentController.php` | PaymentIntent flow, Checkout Session, success handlers. |
| `app/Http/Controllers/StripeWebhookController.php` | Webhook signature verification and DB updates. |
| `app/Models/Payment.php` | Eloquent model; maps Stripe status; `formattedAmount()`, `syncFromPaymentIntent()`. |
| `database/migrations/2026_04_21_000000_create_payments_table.php` | `payments` table schema. |
| `resources/views/payment/checkout.blade.php` | Checkout page (Bootstrap + Stripe.js). |
| `resources/views/payment-success.blade.php` | Animated success page. |
| `config/services.php` | `stripe.key`, `stripe.secret`, `stripe.webhook_secret`. |

## Database: `payments` table

Stores: Stripe PaymentIntent id, optional Checkout session id, amount (smallest currency unit), currency, normalized status, optional `user_id`, optional `product_id`, timestamps.

## Local webhook testing (Stripe CLI)

```bash
stripe listen --forward-to http://127.0.0.1:8000/stripe/webhook
```

Paste the printed **`whsec_...`** into `STRIPE_WEBHOOK_SECRET`.  
Subscribe to events such as: `payment_intent.succeeded`, `payment_intent.payment_failed`, `payment_intent.canceled`, `payment_intent.processing`, `checkout.session.completed`.

## Test cards

Use [Stripe test cards](https://stripe.com/docs/testing). Examples:

- Success: **4242 4242 4242 4242**
- 3DS: **4000 0025 0000 3155**
- Decline: **4000 0000 0000 0002**

Use any future expiry, any CVC, and any postal code if asked.

## Quick manual test

1. Set `STRIPE_KEY`, `STRIPE_SECRET`, run `php artisan serve` (or your vhost).
2. Open **`/payment`**.
3. Click **Pay now**, complete test card, land on **`/payment/success`**.
4. Confirm a row in **`payments`** with `status = succeeded`.

## Troubleshooting

| Issue | What to check |
|-------|----------------|
| 503 Stripe not configured | `STRIPE_KEY` and `STRIPE_SECRET` in `.env`, config cache: `php artisan config:clear`. |
| Wrong redirect / return URL | `APP_URL` matches how you open the site. |
| Webhook 400 Invalid signature | `STRIPE_WEBHOOK_SECRET` matches the endpoint/CLI secret; raw body must not be modified. |
| Amount errors | Amount sent to Stripe is in the **smallest currency unit** (e.g. USD cents); minimum rules apply per currency. |

---

*Yeh README sirf is project ke Stripe flow ke liye hai. Poora Laravel setup alag se dekhein (`composer.json`, `.env.example`).*
