---
generated: '2026-08-13'
name: Find and hire a creator
method: generated
description: Search the Trend creator network with pagination, read a creator's public profile and portfolio, then invite and approve them onto a campaign.
api: openapi/trend-api-openapi.yml
operations: [CreatorController_findAll, CreatorController_findOne, ProfileController_findOnePublicProfile, PortfolioController_getCreatorPortfolio, CreatorController_getCreatorDemographicData, CampaignController_inviteCreator, CampaignController_approvePartnership, BrandController_favoriteCreator, MessageController_brandSendMessage]
source: >-
  Grounded in openapi/trend-api-openapi.yml (verbatim original in
  openapi/_original/trend-openapi-original.json, fetched from
  https://api.trend.io/docs-json). Every operationId, parameter and DTO field
  verified against the spec. Pagination shape per
  ../conventions/trend-conventions.yml; entity graph per
  ../data-model/trend-data-model.yml.
---

# Find and hire a creator

Search the vetted creator network, evaluate a creator on published signals, then bring them onto a campaign.

## Auth
- Base URL: `https://api.trend.io`
- `Authorization: Bearer <idToken>` from `POST /auth/brand/login`. See `../authentication/trend-authentication.yml`.

## Steps

1. **Search the network** — `GET /creator` (`CreatorController_findAll`). Returns `FindAllResponse`: `documents` (array of `CreatorUserDto`), `total`, `totalPages`, `page`, `perPage`, `sortBy`, `sortOrder`, `paginationFlags`.

   Pagination: `page`, `perPage`, `sortBy`, `paginationFlags`, `returnAll`. Filters include `_id` and `userId`. Note the address query parameters (`line1`, `line2`, `city`, `postalCode`, `state`, `country`) are declared **required** on this operation in the spec — a quirk of how the address DTO was flattened into the query string. Send them if the call rejects your request without them.

2. **Read the public profile** — `GET /creator/profile/public/{id}` (`ProfileController_findOnePublicProfile`). Returns `PublicProfileResponse`, which is where the hiring signal lives:
   - `overallRating`, `totalReviews`, `contentQuality`, `professionalism`, `reviews`
   - `instagramStats` — social reach
   - `partnershipCreditCost` — **what hiring this creator will cost you in credits** (20, 40 or 60 by level)
   - `socialCategories`, `contentCategories`, `jobTypes`, `equipment`, `environment`, `ageRange`, `address`

3. **Check their work** — `GET /creator/portfolio/{username}` (`PortfolioController_getCreatorPortfolio`) returns `PortfolioDto` items (`url`, `type`).

4. **Check reliability** — `GET /creator/{id}` (`CreatorController_findOne`) returns the `Creator` record including `isOverdue`, `overduePartnerships`, `contentDeliveryScore` and `contentDeliveryAverageDays`. A creator with `isOverdue: true` is late on another brand's work right now.

5. **Audience fit** — `GET /creator/demographic-data` (`CreatorController_getCreatorDemographicData`).

6. **Shortlist** — `POST /brand/{brandId}/favorite-creator` (`BrandController_favoriteCreator`) with `UpdateFavoriteCreatorDto`; reverse with `POST /brand/{brandId}/unfavorite-creator` (`BrandController_unFavoriteCreator`).

7. **Invite them to a campaign** — `PATCH /campaign/invite-creator/{campaignId}/{creatorId}` (`CampaignController_inviteCreator`) with `InviteCreatorBodyDto`: `rehireContentNeeds` (required), `rehireWillBrandSendProduct` (required), `rehireProductDescription` (optional).

8. **Approve the partnership** — `PATCH /campaign/approve-partnership/{campaignId}/{partnershipId}` (`CampaignController_approvePartnership`). No body. **Spends `partnershipCreditCost` credits.**

9. **Talk to them** — `POST /message/brand/{threadId}/send` (`MessageController_brandSendMessage`) with `BrandSendMessageRequestDto`: `creatorId`, `text`, `type` required; `url`, `thumbnail`, `messageId` optional. Mark read with `POST /message/brand/{threadId}/read` (`MessageController_brandReadMessage`). The `threadId` for a campaign delivery is `ContentCollectionResponse.messageId`.

## Rehire flow
A creator you have worked with before can be re-invited; they accept or decline via `PATCH /partnership/{partnershipId}/accept-rehire-invite` (`PartnershipController_acceptRehireInvite`) or `.../decline-rehire-invite` (`declineRehireInvite`). Mobile-app variants exist as `appAcceptRehireInvite` / `appDeclineRehireInvite` on `/app-accept-rehire-invite` and `/app-decline-rehire-invite`. Pending invites are cleared with `PATCH /partnership/view-approve-invites` (`PartnershipController_viewApproveInvites`).

## Notes
- Pagination is page-number only — no cursors, no `Link` headers. Read `total` and `totalPages` from the body.
- `returnAll: true` bypasses pagination entirely and returns the whole collection. Use it deliberately; the network is thousands of creators.
- Creator ids are MongoDB ObjectIds (`_id`); user identity is a Firebase uid (`firebaseUserId`). They are different keys for related records — see `../data-model/trend-data-model.yml`.
- Step 8 has no idempotency key and spends money. Re-read the partnership state instead of retrying.

## Errors
- `401 {"message":"Unauthorized","statusCode":401}` — token expired.
- `400` with an array `message` — a required query parameter or DTO field is missing (most often the address parameters on `GET /creator`).
- `404` — the creator, campaign or partnership id does not resolve.
