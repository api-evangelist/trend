---
generated: '2026-08-13'
name: Submit content as a creator
method: generated
description: As a creator, apply to a campaign, upload deliverables through an S3 pre-signed URL, attach them to the partnership, and notify the brand.
api: openapi/trend-api-openapi.yml
operations: [CreatorAuthController_login, PartnershipController_submitApplication, UploaderController_getPresignedUrl, SubmitContentController_creatorAddContent, SubmitContentController_notifyBrand, MessageController_creatorSendMessage, ContentController_getContentCollectionPage, CreatorLevelController_getLevel]
source: >-
  Grounded in openapi/trend-api-openapi.yml (verbatim original in
  openapi/_original/trend-openapi-original.json, fetched from
  https://api.trend.io/docs-json). Every operationId, query parameter and DTO field
  verified against the spec. Upload mechanics per the PresignedUrlResponseDto schema;
  error handling per ../errors/trend-problem-types.yml.
---

# Submit content as a creator

The creator-side flow: apply → get hired → upload → attach → notify. Binary content never transits the Trend API; it goes straight to S3 through a pre-signed URL.

## Auth
- Base URL: `https://api.trend.io`
- `POST /auth/login` (`CreatorAuthController_login`) with `EmailPasswordDto` (`email`, `password`). Returns `TrendCreatorAuthResponse` with `idToken`. Google SSO: `POST /auth/login/google` (`CreatorAuthController_googleLogin`).
- Send `Authorization: Bearer <idToken>` on everything after that.
- Your profile: `GET /auth/profile` (`CreatorAuthController_getProfile`); creator settings: `GET /creator/profile/settings` (`ProfileController_findPersonalProfileSettings`).

## Steps

1. **Apply to a campaign** — `POST /partnership/{campaignId}/application` (`PartnershipController_submitApplication`) with `SubmitApplicationRequestDto` (`writtenApplication`). Capture the resulting `partnershipId` — every later step is keyed on it.

2. **Wait for approval.** The brand approves via `CampaignController_approvePartnership`; the platform can also activate one through `POST /system/partnership/activate` (`SystemPartnershipController_systemActivatePartnership`). Your delivery deadline arrives via `PartnershipController_setDueDate`.

3. **Mint an upload URL** — `POST /upload/pre-signed-url` (`UploaderController_getPresignedUrl`). Query parameters: `uploadType` (**required**), `fileName` (**required**), and optionally `creatorId`, `influencerUID`, `contentId`, `brandId`. Returns `PresignedUrlResponseDto`:
   - `signedUrl` — the S3 URL you `PUT` the bytes to
   - `fileName`
   - `fileDestinationUrl` — the permanent location; **this is what you hand back to Trend in step 5**

4. **Upload the file** — `PUT` the raw bytes directly to `signedUrl`. This is an S3 request, not a Trend request: no `Authorization` header, and Trend's error envelope does not apply. Pre-signed URLs expire; mint a fresh one rather than reusing a stale one.

5. **Attach it to the partnership** — `POST /content/submit/{partnershipId}` (`SubmitContentController_creatorAddContent`) with `AddContentRecordDto`: `contentPath` (**required** — the `fileDestinationUrl` from step 3), `category` (**required**), `isFree` (optional). Repeat per deliverable.

6. **Notify the brand** — `POST /content/submit/notify-brand` (`SubmitContentController_notifyBrand`) with `ContentUploadNotificationDto`.

7. **Handle revisions.** If the brand calls `ContentController_brandRequestRevision`, the `rejectionReason` reaches you through the thread. Re-upload with steps 3–5. Read the current state with `GET /content/{campaignId}/{creatorId}` (`ContentController_getContentCollectionPage`) — `pendingContents` versus `approvedContents` tells you where you stand.

8. **Message the brand** — `POST /message/creator/{threadId}/send` (`MessageController_creatorSendMessage`) with `CreatorSendMessageRequestDto`. Unread count: `GET /message/creator/unread` (`MessageController_getCreatorUnreadMessageCount`); mark read: `POST /message/creator/{threadId}/read` (`MessageController_creatorReadMessage`).

## Your standing on the platform
- `GET /creator/level` (`CreatorLevelController_getLevel`) — your level, which sets what brands pay to hire you.
- `GET /creator/level/bonus-table` (`CreatorLevelController_getBonusLevels`) — the bonus ladder. Requires auth; returns `401` anonymously.
- `POST /creator/portfolio/add` (`PortfolioController_create`) with `PortfolioDto` (`url`, `type`); remove with `DELETE /creator/portfolio/remove` (`PortfolioController_remove`).
- `PATCH /creator/profile/{creatorId}` (`ProfileController_updateProfile`) with `UpdateProfileDto`.
- Getting paid: `GET /payment/payouts` (`PaymentController_payouts`) returns `GetPayoutsByDateRangeResponse`. Your payout destination is `Creator.payPalEmail`.
- Missing a deadline sets `isOverdue` and populates `overduePartnerships` on your `Creator` record, and brands can read both.

## Notes
- The two-step upload (mint URL → PUT to S3 → register path) means a failed registration in step 5 leaves an orphaned object in S3 that Trend does not know about. Always complete step 5.
- No idempotency key exists on `POST /content/submit/{partnershipId}` — a retry can attach the same file twice. Read the collection back before retrying.
- Account deletion: `DELETE /auth/remove-my-user` (`CreatorAuthController_removeUser`).

## Errors
- `401 {"message":"Unauthorized","statusCode":401}` — token expired, or the route requires a signed-in creator.
- `400` with an array `message` — a required DTO field is missing (`contentPath` or `category` in step 5; `uploadType` or `fileName` in step 3).
- `404` — `partnershipId` or `campaignId` does not resolve.
- S3 errors from step 4 are XML from AWS, not Trend's JSON envelope. Handle them separately.
