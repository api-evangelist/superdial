---
name: Super Dial
description: Use when submitting structured data extraction requests to insurance payers, managing phone-based and digital workflows, handling webhook callbacks, and retrieving insurance claim status, benefits verification, prior authorization, and other payer data. Reach for this skill when building integrations that need to automate payer interactions, track request batches, or verify webhook signatures.
metadata:
    mintlify-proj: superdial
    version: "1.0"
---

# SuperDial Skill

## Product summary

SuperDial is an API for submitting structured data extraction jobs (called "requests") against insurance payers and retrieving results. Agents use it to automate phone calls and digital lookups to payers, extract claim status, verify benefits, check prior authorization, and other insurance workflows. Requests are submitted via `POST /v1/requests`, tracked by `requestId` and `requestBatchId`, and results are delivered via webhooks or polling. Authentication uses bearer tokens obtained from `GET /v1/auth` with API key/secret pairs. The primary documentation is at https://docs.superdial.com.

**Key endpoints:**
- `GET /v1/auth` — exchange API key/secret for a 1-hour bearer token
- `POST /v1/requests` — submit single or batch requests
- `GET /v1/requests/{requestId}` — fetch a single request result
- `GET /v1/requests` — list requests by date or batch
- `GET /v1/schemas` — discover available schemas for your account
- `GET /v1/schemas/{schemaId}/required-inputs` — look up required input fields

**Key files/concepts:**
- `schemaId` — identifies the output schema and request type (e.g., `claim-status`, `vob`)
- `requestId` — unique identifier for a single request
- `requestBatchId` — scheduling grouping; multiple requests in one submission may span multiple batch IDs if they land on different business days
- `internalId` — optional correlation ID you supply; also the idempotency key
- `payerLookup` — optional payer phone number resolution (opt-in capability)
- Webhook signature verification using HMAC-SHA256

## When to use

Reach for this skill when:
- **Submitting requests:** building a form or workflow to submit claim status, benefits verification, prior authorization, or other payer data extraction jobs
- **Handling webhooks:** receiving and verifying webhook callbacks when requests complete, parsing the compact payload, and fetching full results
- **Batch operations:** submitting multiple requests at once and tracking progress across a batch or multiple batches
- **Payer resolution:** using optional payer phone number lookup to resolve payer names to dialing numbers
- **Error handling:** interpreting `errorCode` and `errorCategory` to distinguish between "payer couldn't find the entity" vs. "system failure" vs. "missing input"
- **Polling fallback:** when webhooks aren't configured, polling `GET /v1/requests/{requestId}` to check request status
- **Debugging:** reading `callSteps`, `contributingCalls`, and `resultSources` to understand multi-call requests or follow-up efforts (e.g., benefits then prior auth)

## Quick reference

### Authentication flow
```
1. GET /v1/auth with Robodialer-API-Key and Robodialer-API-Secret headers
2. Extract token from response
3. Pass Authorization: Bearer <token> on all subsequent requests
4. Token valid for 1 hour; refresh by calling GET /v1/auth again
```

### Request states
| State | Terminal? | Meaning |
|---|---|---|
| `PROCESSING` | No | Still running; wait for webhook |
| `SUCCESS` | Yes | All required fields captured |
| `PARTIAL` | Yes | Primary call succeeded, follow-up failed; `results` partial, `missingFields` lists what's missing |
| `FAILURE` | Yes | No required fields captured; `error` object describes cause |

### Error categories and codes
| Category | Meaning | Example codes |
|---|---|---|
| `NOT_FOUND` | Payer couldn't find the entity | `MEMBER_NOT_FOUND`, `CLAIM_NOT_FOUND`, `*_MISSING`, `*_INCORRECT` |
| `SYSTEM_ERROR` | Any other failure | `IVR_FAILURE`, `PAYER_REFUSAL`, `UNABLE_TO_REACH_HUMAN_IN_TIME`, `MATCHED_PHONE_NUMBER_INCORRECT` |

### Modality (how result was obtained)
| Value | Meaning |
|---|---|
| `digital_only` | Electronic channels only; no phone call |
| `phone_only` | Phone call only |
| `digital_plus_phone` | Electronic attempted first, then phone call |
| `null` | Not yet dispatched (PROCESSING state) |

### Common input fields
- `payerName` — insurance company name (required unless lookup disabled)
- `phoneNumber` — payer phone number to dial (optional if payer lookup enabled)
- `memberId` — member/subscriber ID (schema-specific)
- `dateOfService` — service date (format: `YYYY-MM-DD`)
- `claimNumber` — claim ID (schema-specific)
- `providerNpi` — provider NPI (schema-specific)

### Webhook payload (compact)
```json
{
  "requestId": "...",
  "requestBatchId": "...",
  "state": "SUCCESS|PARTIAL|FAILURE",
  "internalId": "...",  // only if supplied at create
  "internalTag": "..."  // only if supplied at create
}
```

## Decision guidance

### When to use webhooks vs polling
| Scenario | Use webhooks | Use polling |
|---|---|---|
| High-volume, latency-sensitive | ✓ | — |
| No public endpoint available | — | ✓ |
| Cost-conscious (fewer API calls) | ✓ | — |
| Simple, low-volume integration | Either | ✓ |
| Webhook endpoint not yet built | — | ✓ |

**Recommendation:** Always configure webhooks. They're lower latency, lower cost, and the recommended pattern. Polling is a fallback.

### When to supply phoneNumber vs rely on payer lookup
| Scenario | Supply phoneNumber | Use payer lookup |
|---|---|---|
| Payer lookup disabled on account | ✓ | — |
| You have a verified payer number | ✓ | — |
| You want SuperDial to find the number | — | ✓ |
| You want to override SuperDial's match | Set `useMatchedPayerPhone: false` (default) | — |
| You want SuperDial's number with fallback | Set `useMatchedPayerPhone: true` | ✓ |

**Recommendation:** If lookup is enabled, omit `phoneNumber` and let SuperDial resolve it. If you supply a number, it's dialed as-is unless you set `useMatchedPayerPhone: true`.

### When to use internalId
| Scenario | Use internalId |
|---|---|
| You need to correlate results to your own records | ✓ |
| You need idempotency (retry safety) | ✓ |
| Simple, one-off requests | Optional |
| Batch submissions | ✓ (one per request for dedup) |

**Gotcha:** `internalId` is the idempotency key. Reusing the same ID returns the original request, not a new one. Use a unique value per request (UUID, your primary key).

## Workflow

### Typical request submission and result retrieval

1. **Authenticate:** Call `GET /v1/auth` with your API key and secret. Extract the `token` from the response. Store it; it's valid for 1 hour.

2. **Discover schemas:** Call `GET /v1/schemas` to list available schemas for your account. Note the `schemaId` and `requestType` of the schema you need (e.g., `claim-status`).

3. **Look up required inputs:** Call `GET /v1/schemas/{schemaId}/required-inputs` to see which input fields are required and optional for that schema.

4. **Prepare the request body:** Build a JSON object with:
   - `schemaId` — from step 2
   - `inputs` — a flat object with string values for all required fields (and optional ones if you have them)
   - `internalId` — optional, but recommended for correlation and idempotency
   - `internalTag` — optional, for tagging (e.g., `"march-batch"`)
   - `webhookUrl` — optional, to override the account default

5. **Submit the request:** Call `POST /v1/requests` with the body. For a single request, send the object directly. For a batch, wrap in `{ "requests": [...] }`. Capture the `requestId` and `requestBatchId` from the response.

6. **Wait for completion:** If webhooks are configured, wait for the webhook callback. Otherwise, poll `GET /v1/requests/{requestId}` until `state` is not `PROCESSING`.

7. **Fetch the full result:** When the webhook fires (or polling shows a terminal state), call `GET /v1/requests/{requestId}` to get the full response with `results`, `missingFields`, `modality`, and `error` (if `FAILURE`).

8. **Handle the result:** Check `state`. If `SUCCESS`, extract fields from `results`. If `PARTIAL`, check `missingFields` to see what a follow-up call didn't capture. If `FAILURE`, inspect `error.errorCode` to understand why.

### Webhook verification and handling

1. **Receive the webhook POST** at your configured endpoint.

2. **Extract the signature:** Read the `X-Webhook-Signature` header (a 64-character hex string).

3. **Verify the signature:** Compute HMAC-SHA256 of the raw request body (not parsed JSON) using your production API key (or webhook secret if set) as the secret. Compare with the header using a constant-time comparator.

4. **Dedup:** Check if you've already processed this `requestId`. If yes, return 200 immediately.

5. **Ack quickly:** Return HTTP 200 as soon as you've durably enqueued the event. Don't do synchronous work that could exceed the 10-second timeout.

6. **Fetch the full result:** In a background job, call `GET /v1/requests/{requestId}` to get `results`, `missingFields`, `modality`, and `error`.

7. **Process:** Handle the result based on `state` and `error.errorCode`.

### Batch submission with multi-day scheduling

1. **Prepare the batch:** Build an array of request objects, each with `schemaId`, `inputs`, and optional `internalId`.

2. **Submit:** Call `POST /v1/requests` with `{ "requests": [...] }`.

3. **Capture batch IDs:** The response is `{ "requests": [...] }` with one entry per submitted request. **Each entry may have a different `requestBatchId`** if the batch spans multiple business days. Iterate the response and store each `requestBatchId`.

4. **Track progress:** For each unique `requestBatchId`, call `GET /v1/requests?requestBatchId=<id>` to list all requests in that batch and count by `state`.

5. **Handle errors:** In the batch response, each failed entry carries an `error` object. Treat each entry independently; successful entries are real requests even if others failed.

## Common gotchas

- **Idempotency key reuse:** `internalId` is the idempotency key. Reusing the same ID returns the original request, not a new one. Always use a unique value per request. Don't use a constant batch label as the ID.

- **Batch IDs don't match:** When you submit a batch, entries may land on different business days and get different `requestBatchId` values. Always read each entry's ID from the response; don't assume they match.

- **Webhook signature verification is mandatory:** Without verification, anyone who learns your endpoint URL can forge events. Always verify the `X-Webhook-Signature` header using a constant-time comparator.

- **Webhooks are signed with production key even for sandbox:** Sandbox requests fire webhooks signed with your production API key, not your sandbox key. Use the production key for verification.

- **Webhook timeout is 10 seconds:** Return HTTP 200 immediately. Push processing to a background job. If you exceed 10 seconds, the webhook is retried.

- **Read-after-write lag on single-request GET:** For a few seconds after `POST /v1/requests`, a `GET /v1/requests/{requestId}` can return `404 REQUEST_NOT_FOUND` even though the POST succeeded. Tolerate 404s for the first few seconds, or use webhooks to avoid polling.

- **All input values must be strings:** The `inputs` object must have string values, even for dates and numbers. Pass `"2026-03-15"` not `2026-03-15`.

- **Payer lookup is opt-in:** Payer phone number lookup is disabled by default. Ask your account team to enable it if you want to omit `phoneNumber` and let SuperDial resolve it.

- **Per-payer required inputs:** Some payers require additional inputs beyond the schema's fields. If enabled for your account, a missing per-payer input is reported in the `INVALID_INPUTS` error. Check the [Per-Payer Required Inputs guide](/guides/payer-required-inputs) if you see unexpected validation errors.

- **Webhook payload is compact:** The webhook body only carries `requestId`, `requestBatchId`, `state`, and optional `internalId`/`internalTag`. Fetch the full result via `GET /v1/requests/{requestId}` to get `results`, `missingFields`, `modality`, and `error`.

- **Multi-call requests:** A request can involve more than one phone call (redials) or more than one step (e.g., benefits then prior auth). The top-level `transcript` and `callSummary` only describe the first call. Use `callSteps`, `contributingCalls`, and `resultSources` to see the full picture.

- **Token expiration:** Tokens are valid for 1 hour. For long-running batches or background workers, refresh at the start of each work cycle or on `401` responses.

## Verification checklist

Before submitting work with SuperDial:

- [ ] **Authentication:** Verified that `GET /v1/auth` returns a valid token and subsequent requests pass it as `Authorization: Bearer <token>`
- [ ] **Schema discovery:** Called `GET /v1/schemas` and confirmed the schema you need exists for your account
- [ ] **Required inputs:** Called `GET /v1/schemas/{schemaId}/required-inputs` and verified all required fields are present in your request body
- [ ] **Input format:** All values in `inputs` are strings (dates as `YYYY-MM-DD`, numbers as quoted strings)
- [ ] **Idempotency:** If using `internalId`, confirmed it's unique per request (not reused across requests)
- [ ] **Webhook signature:** If handling webhooks, verified the `X-Webhook-Signature` header using HMAC-SHA256 and a constant-time comparator
- [ ] **Webhook ack:** Webhook handler returns HTTP 200 within 10 seconds; processing is pushed to a background job
- [ ] **Batch handling:** If submitting a batch, confirmed each entry's `requestBatchId` is captured (they may differ)
- [ ] **Error handling:** Checked for `INVALID_INPUTS` errors and confirmed all required fields are supplied
- [ ] **Payer lookup:** If using payer lookup, confirmed it's enabled for your account; if not, ensured `phoneNumber` is supplied
- [ ] **Result retrieval:** Confirmed that after a webhook fires (or polling shows a terminal state), `GET /v1/requests/{requestId}` returns the full result with `results`, `missingFields`, and `error`

## Resources

- **Comprehensive page listing:** https://docs.superdial.com/llms.txt — full navigation of all documentation pages
- **Introduction & Quickstart:** https://docs.superdial.com/introduction — overview, authentication, and end-to-end flow
- **Creating a Request:** https://docs.superdial.com/guides/creating-a-request — single and batch submission, required fields, idempotency, error handling
- **Webhooks:** https://docs.superdial.com/guides/webhooks — payload shape, signature verification, delivery and retries, idempotency

---

> For additional documentation and navigation, see: https://docs.superdial.com/llms.txt