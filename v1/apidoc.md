# Claims API

Version **1.0.0** of the Claims API lets clients create and manage Health Payer insurance
claims, then submit draft claims for review.

## Base URL

All endpoints are relative to:

```text
https://api.sun.com/v1
```

Replace `https://api.example.com` with the host for your environment. Requests
and responses use JSON (`Content-Type: application/json`). Authentication
requirements are not defined by this API contract.

## Claim lifecycle

A newly created claim has `draft` status. Draft claims can be updated and
submitted. A successful submission returns the claim with `submitted` status;
claims in any other status cannot be submitted again or edited through the
draft-update endpoint.

Statuses returned by the API:

| Status | Meaning |
| --- | --- |
| `draft` | Claim is being prepared. |
| `submitted` | Claim has been submitted for review. |
| `under_review` | Claim is being reviewed. |
| `approved` | Claim has been approved. |
| `rejected` | Claim has been rejected. |
| `paid` | Payment has been made. |
| `withdrawn` | Claim has been withdrawn. |

## Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/claims` | Create a claim in draft status. |
| `GET` | `/claims` | List and filter claims. |
| `GET` | `/claims/{claimId}` | Retrieve a claim. |
| `PATCH` | `/claims/{claimId}` | Update a draft claim. |
| `POST` | `/claims/{claimId}/submit` | Submit a draft claim for review. |

`claimId` is the UUID returned in the claim's `id` field.

## Create a claim

`POST /claims`

Required fields: `policyNumber`, `claimant`, `incident`, and `requestedAmount`.
The server assigns the claim ID, status, and timestamps.

```http
POST /v1/claims HTTP/1.1
Host: api.example.com
Content-Type: application/json
```

```json
{
  "policyNumber": "POL-84721",
  "claimant": {
    "firstName": "Alex",
    "lastName": "Morgan",
    "email": "alex.morgan@example.com",
    "phone": "+1-555-0100"
  },
  "incident": {
    "date": "2026-10-28",
    "description": "Water damage in the kitchen following a pipe leak.",
    "location": "Kitchen"
  },
  "requestedAmount": {
    "amount": 1250.5,
    "currency": "USD"
  }
}
```

Successful response: `201 Created`. The response includes a `Location` header
with the URL of the new claim.

```json
{
  "id": "8b4e8d18-61c5-4f4a-9b2d-2fc64818e749",
  "policyNumber": "POL-84721",
  "claimant": {
    "firstName": "Alex",
    "lastName": "Morgan",
    "email": "alex.morgan@example.com",
    "phone": "+1-555-0100"
  },
  "incident": {
    "date": "2026-10-28",
    "description": "Water damage in the kitchen following a pipe leak.",
    "location": "Kitchen"
  },
  "requestedAmount": {
    "amount": 1250.5,
    "currency": "USD"
  },
  "status": "draft",
  "createdAt": "2026-11-15T00:00:00Z",
  "updatedAt": "2026-11-15T00:00:00Z",
  "submittedAt": null
}
```

## List claims

`GET /claims`

| Query parameter | Type | Description |
| --- | --- | --- |
| `limit` | integer | Maximum results per page; from 1 to 100. Defaults to 20. |
| `cursor` | string | Cursor from a previous response to retrieve the next page. |
| `status` | string | Filter by claim status. |
| `policyNumber` | string | Filter by policy number. |
| `incidentDateFrom` | date | Include incidents on or after this date (`YYYY-MM-DD`). |
| `incidentDateTo` | date | Include incidents on or before this date (`YYYY-MM-DD`). |

Example:

```http
GET /v1/claims?status=draft&limit=20 HTTP/1.1
Host: api.example.com
```

The response contains a `data` array and `pagination` object. When another page
is available, `nextCursor` contains the cursor to pass in the next request;
otherwise it is `null`.

```json
{
  "data": [],
  "pagination": {
    "limit": 20,
    "nextCursor": null
  }
}
```

## Retrieve a claim

`GET /claims/{claimId}`

Returns the claim with the given UUID (`200 OK`), or `404 Not Found` if it does
not exist.

```http
GET /v1/claims/8b4e8d18-61c5-4f4a-9b2d-2fc64818e749 HTTP/1.1
Host: api.example.com
```

## Update a draft claim

`PATCH /claims/{claimId}`

Provide one or more of `policyNumber`, `claimant`, `incident`, and
`requestedAmount`. The request body cannot be empty. Nested objects must include
their required fields. `id`, `status`, and timestamps are server-managed and
cannot be updated here.

```json
{
  "requestedAmount": {
    "amount": 1400,
    "currency": "USD"
  }
}
```

Returns the updated claim (`200 OK`). Returns `404 Not Found` if the claim does
not exist, or `409 Conflict` if it is not in draft status.

## Submit a claim

`POST /claims/{claimId}/submit`

No request body is required. A successful request returns the submitted claim
(`200 OK`). Returns `404 Not Found` if the claim does not exist, or
`409 Conflict` if it is not in draft status.

```http
POST /v1/claims/8b4e8d18-61c5-4f4a-9b2d-2fc64818e749/submit HTTP/1.1
Host: api.example.com
```

## Data formats

- Dates use ISO 8601 calendar dates (`YYYY-MM-DD`).
- Timestamps use ISO 8601 date-time strings.
- `requestedAmount.amount` must be greater than zero.
- `requestedAmount.currency` is a three-letter uppercase ISO 4217 currency
  code, such as `USD`.
- Claimant `email` must be a valid email address.

## Errors

Errors use a consistent JSON shape:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The request is invalid.",
  "details": [
    {
      "field": "requestedAmount.amount",
      "message": "Must be greater than zero."
    }
  ]
}
```

`details` is optional. Each detail identifies a `field` and explains the
problem.

| HTTP status | Meaning |
| --- | --- |
| `400 Bad Request` | The request or its parameters are invalid. |
| `404 Not Found` | No claim exists for the supplied ID. |
| `409 Conflict` | The requested action is not allowed in the claim's current status. |
