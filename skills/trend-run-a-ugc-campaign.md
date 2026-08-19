---
generated: '2026-08-13'
name: Run a UGC campaign end to end
method: generated
description: As a brand, draft a creative brief, submit it to the Trend creator network, hire creators, ship product, and approve delivered content.
api: openapi/trend-api-openapi.yml
operations: [BrandAuthController_brandLogin, CampaignController_findOneDraftByBrand, CampaignController_updateDraft, CampaignController_submitCampaign, CampaignController_inviteCreator, CampaignController_approvePartnership, PartnershipController_setDueDate, ShipmentController_createShipment, CampaignController_getCampaignSubmissions, ContentController_getContentCollectionPage, ContentController_brandRequestRevision, ContentController_brandApproveContent]
source: >-
  Grounded in openapi/trend-api-openapi.yml (verbatim original in
  openapi/_original/trend-openapi-original.json, fetched from
  https://api.trend.io/docs-json). Every operationId and every request-body field
  below was verified against the spec. Cross-cutting rules per
  ../conventions/trend-conventions.yml, ../errors/trend-problem-types.yml,
  ../authentication/trend-authentication.yml and ../data-model/trend-data-model.yml.
---

# Run a UGC campaign end to end

The brand-side flow: brief → listing → hire → ship → deliver → approve. Approving a partnership spends credits, so read `../plans/trend-plans-pricing.yml` before you start.

## Auth
- Base URL: `https://api.trend.io`
- `POST /auth/brand/login` (`BrandAuthController_brandLogin`) with `EmailPasswordDto` (`email`, `password`). The response `TrendBrandAuthResponse` carries `idToken`, `refreshToken`, `expiresIn`, `localId`, `customToken` and `brand`.
- Send `Authorization: Bearer <idToken>` on every subsequent call. `idToken` is a Firebase JWT and expires — refresh with `POST /auth/brand/custom-token` (`BrandAuthController_brandRefreshCustomToken`).
- Google SSO alternative: `POST /auth/brand/login/google` (`BrandAuthController_brandLoginSSO`).

## Steps

1. **Fetch your working draft** — `GET /campaign/draft` (`CampaignController_findOneDraftByBrand`). Returns the brand's current draft campaign. Capture its `campaignId`.

2. **Fill in the brief** — `PATCH /campaign/draft/{campaignId}` (`CampaignController_updateDraft`) with `UpdateDraftCampaignDto`. All fields optional at draft stage: `type`, `category`, `platform`, `styleCategory`, `gender`, `name`, `url`, `value`, `userFullName`, `companyName`. `url` is the product page; `value` is the product's retail value.

3. **Submit it to the network** — `PATCH /campaign/submit/{campaignId}` (`CampaignController_submitCampaign`) with `SubmitCampaignDto`. Here `type`, `category`, `platform`, `styleCategory`, `gender`, `name`, `url` and `value` are all **required** — a draft that passed step 2 with fields missing will fail validation here with a 400 whose `message` is an array of per-field strings.

4. **Bring in creators.** Either wait for applications, or invite directly with `PATCH /campaign/invite-creator/{campaignId}/{creatorId}` (`CampaignController_inviteCreator`) using `InviteCreatorBodyDto` — `rehireContentNeeds` and `rehireWillBrandSendProduct` are required, `rehireProductDescription` optional. Find creators first with the *Find and hire a creator* skill.

5. **Approve the hire** — `PATCH /campaign/approve-partnership/{campaignId}/{partnershipId}` (`CampaignController_approvePartnership`). No request body. **This spends credits** — 20, 40 or 60 depending on the creator's level (`PublicProfileResponse.partnershipCreditCost`). There is no idempotency key on this route; see Notes.

6. **Set the delivery deadline** — `PATCH /partnership/set-due-date/{partnershipId}` (`PartnershipController_setDueDate`).

7. **Ship the product** — `POST /shipment/create` (`ShipmentController_createShipment`) with `ShipmentDto`: `partnershipId` and `nonShippable` required, `trackingNumber` and `carrierCode` optional. Set `nonShippable: true` for digital or service products. Update later with `PATCH /shipment/update` (`ShipmentController_updateShipment`).

8. **Watch for submissions** — `GET /campaign/submissions/{campaignId}` (`CampaignController_getCampaignSubmissions`) returns `CampaignContentSubmissionsResponse` with `submissions` and `approvals`. For one creator's delivery, `GET /content/{campaignId}/{creatorId}` (`ContentController_getContentCollectionPage`) returns `ContentCollectionResponse` with `approvedContents`, `pendingContents`, `brandCredits`, `creator`, `campaign` and `messageId`.

9. **Request changes, if needed** — `PATCH /content/request-revision/{contentCollectionId}` (`ContentController_brandRequestRevision`) with `RequestRevisionDto` (`rejectionReason`, required).

10. **Approve the content** — `PATCH /content/approve/{campaignId}/{creatorId}` (`ContentController_brandApproveContent`) with `ApproveContentRequestDto` (`contentIds`, required — the array of content record ids you are accepting). Returns `ContentApprovalResponse`.

11. **Review the creator** — `PATCH /review/{reviewId}/brand-review-influencer` (`ReviewController_brandReviewInfluencer`). This feeds `overallRating`, `contentQuality` and `professionalism` on the creator's public profile.

## Campaign state management
- `PATCH /campaign/{campaignId}/unlist` and `/relist` (`CampaignController_brandUnlistCampaign` / `brandRelistCampaign`) pull the brief off the network and put it back.
- `PATCH /campaign/{campaignId}/archive` and `/reactivate` (`brandArchiveCampaign` / `brandReactivateCampaign`).
- `PATCH /campaign/{campaignId}/bump` (`CampaignController_bumpCampaign`) raises the listing in creator feeds.
- `PATCH /campaign/update/{campaignId}` (`CampaignController_updateCampaign`) edits a live campaign.
- Remove a hired creator with `PATCH /campaign/remove-creator/{campaignId}/{partnershipId}` (`CampaignController_removeCreator`).

## Notes
- **No idempotency anywhere.** No operation accepts an `Idempotency-Key` header. Steps 5 and 7 are the dangerous ones — a retry after a timeout can double-spend credits or create a duplicate shipment. Confirm state with a read before retrying, never blind-retry.
- **No rate-limit headers.** Nothing tells you your budget; see `../rate-limits/trend-rate-limits.yml`.
- **No request id** is echoed, so there is nothing to quote if you need support.
- The spec declares no error responses at all — only 200/201. Handle 400/401/404 from `../errors/trend-problem-types.yml`, and remember `message` is a string OR an array of strings.

## Errors
- `401 {"message":"Unauthorized","statusCode":401}` — expired or missing `idToken`. Re-login and retry.
- `400` with `message` as a string array — `SubmitCampaignDto` validation. Check the required-field list in step 3.
- `404 {"message":"Cannot <METHOD> <path>",...}` — wrong path or an id that does not resolve.
