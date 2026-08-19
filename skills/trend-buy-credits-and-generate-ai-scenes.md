---
generated: '2026-08-13'
name: Buy credits and generate AI product scenes
method: generated
description: List Trend's prepaid credit packages, start a Stripe Checkout session to buy one, then use the AI surface to generate and enhance product scene imagery.
api: openapi/trend-api-openapi.yml
operations: [StripeController_packageProductsList, StripeController_createCustomer, StripeController_createUserPackageCheckout, StripeController_stripeSessionCard, UploaderController_getAIPresignedUrl, AIController_generatePhotos, AIController_enhanceImageQuality, AIController_getSuperResolution]
source: >-
  Grounded in openapi/trend-api-openapi.yml (verbatim original in
  openapi/_original/trend-openapi-original.json, fetched from
  https://api.trend.io/docs-json). Package values cross-checked against a live
  unauthenticated call to GET /payment/stripe/packages and the published pricing page
  — see ../plans/trend-plans-pricing.yml.
---

# Buy credits and generate AI product scenes

Credits are Trend's unit of account: buying them is a Stripe Checkout flow, and spending them hires creators. The AI surface is separate — it generates product imagery without hiring anyone.

## Auth
- Base URL: `https://api.trend.io`
- `Authorization: Bearer <idToken>` from `POST /auth/brand/login`.
- Exception: step 1 is callable **without any credential**.

## Buying credits

1. **List the packages** — `GET /payment/stripe/packages` (`StripeController_packageProductsList`). Returns an array of `PackageProduct`: `id` (the Stripe product id), `name`, `cost` (**USD cents**), `credits`, `savings`, `description`, `mostPopular`.

   This route answers HTTP 200 to anonymous callers — it is the only operation in the API you can read without signing in. As of 2026-08-13 it returns Starter ($550 / 60 credits), Essential ($1,045 / 140), Growth ($1,980 / 280, `mostPopular: true`) and Scale ($3,872 / 560). Read it live rather than hardcoding; it is the machine-readable source of truth for pricing.

2. **Make sure a Stripe customer exists** — `POST /payment/stripe/create-customer` (`StripeController_createCustomer`).

3. **Start Checkout** — `POST /payment/stripe/checkout/{productId}` (`StripeController_createUserPackageCheckout`), where `productId` is the `id` from step 1. No request body. Returns a full `StripeCheckoutSessionResponse` — the Stripe Checkout Session object, including `url` (redirect the buyer here), `id`, `amount_total`, `currency`, `payment_status`, `expires_at`, `success_url` and `cancel_url`.

4. **Redirect the buyer to `url`.** Card data never touches Trend — Stripe collects it.

5. **Confirm** — `GET /payment/stripe/session/{id}/payment-card` (`StripeController_stripeSessionCard`) reads the card used on a session. Trend receives the authoritative confirmation on its own inbound receiver `POST /payment/stripe/events` (`StripeController_stripePaymentEvents`) — **that route is Stripe's callback into Trend, not a webhook you can subscribe to.** There is no outbound webhook for a completed purchase; poll instead.

6. **Credits land on the brand.** Balance surfaces as `brandCredits` on `ContentCollectionResponse`. Credits expire 12 months after purchase.

## Generating AI product scenes

7. **Upload the source product image** — `POST /upload/ai/pre-signed-url` (`UploaderController_getAIPresignedUrl`) returns a `PresignedUrlResponseDto`; `PUT` the bytes to `signedUrl` and keep `fileDestinationUrl`.

8. **Generate scenes** — `POST /ai/generate` (`AIController_generatePhotos`) with `GenerateSceneRequestDto`: `productImage` (**required** — the `fileDestinationUrl` from step 7), `scene` (**required** — the scene description), and optional `timeOfDay`, `colorPalette`, `perspective`, `scale`, `translation`. Returns `GenerateSceneResponseDto` with a `scenes` array.

9. **Enhance a result** — `POST /ai/fal/enhance` (`AIController_enhanceImageQuality`) runs super-resolution. Retrieve the finished output with `GET /ai/super-resolution/{generationId}` (`AIController_getSuperResolution`).

## Notes
- **`cost` is in cents.** `387200` is $3,872.00. Do not render it raw.
- **No idempotency key on step 3.** Retrying a timed-out checkout creation can produce a second Stripe session. Read back before retrying.
- Enhancement in step 9 is asynchronous — poll `GET /ai/super-resolution/{generationId}`. No callback exists, and the spec declares no status enum for the result, so treat an absent result as "not ready yet".
- The credit price of hiring a specific creator is `partnershipCreditCost` on their `PublicProfileResponse` (20, 40 or 60).
- Admin credit adjustments (`POST /brand/{brandId}/add-credits`, `/remove-credits`) require the `trend-api-key` header and are staff-only.

## Errors
- `401` — missing or expired bearer token on steps 2–9 (step 1 is anonymous).
- `400` with an array `message` — missing `productImage` or `scene` in step 8.
- `404` — the `productId` is not a live Stripe product id; re-read step 1 rather than reusing a cached id.
