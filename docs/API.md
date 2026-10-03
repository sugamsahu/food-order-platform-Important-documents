# API guide

The full, machine-readable description of every endpoint is [`openapi.yaml`](openapi.yaml). A runnable [Postman collection](postman/food-order-platform.postman_collection.json) exercises the mobile flow and the staff workflow against the seed data. This guide covers the conventions, security model, error codes and a curl walkthrough.

Base URL (local): `http://localhost:8080/api/v1`. `GET /health-check` (outside the prefix, public) returns `{"status":"UP",...}` or `503 {"status":"DOWN"}` depending on database connectivity.

| Group | Prefix | Used by | Token |
| --- | --- | --- | --- |
| Staff authentication | `/auth` | Admin interface | none (login), staff token |
| Staff users | `/staff-users` | Admin interface | staff token, **administrator only** |
| Foods | `/foods` | Admin interface | staff token (administrator and operator) |
| Orders | `/orders` | Admin interface | staff token (administrator and operator) |
| Customer (mobile) | `/mobile` | Future mobile app | customer token |

## Conventions

- JSON request and response bodies; send `Content-Type: application/json`.
- **snake_case** field names. Ids are numbers. Quantities are integers.
- **Money is a decimal string** with two decimals (`"48.00"`) in the restaurant's single configured currency. Totals are computed in integer cents on the server.
- Timestamps are ISO 8601 in UTC.
- **Pagination:** every list accepts `page` (default 1) and `page_size` (default 20, maximum 100) and returns
  `{ "data": [...], "pagination": { "page": 1, "page_size": 20, "total": 42, "total_pages": 3 } }`.
- **Search and filters** are query parameters (listed in the OpenAPI file): `q` (case-insensitive substring), `status`, `availability`, `category`, `role`, `is_active`, `fulfillment_type`, `include_archived`. Wildcard characters in `q` are matched literally.
- Orders are listed newest first; foods and staff are listed by name / id.
- Unknown request fields are ignored (for example a client-sent price).

## Error shape

Every error, from validation to a crash, has the same shape:

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Validation failed", "details": { "fields": { "price": ["Must be greater than zero"] } } } }
```

`code` is stable and meant for programs; `message` is for people; `details` carries extra data (field errors, the current status of an order, unavailable food ids).

| HTTP | Code | When |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | Bad body, query or id. `details.fields` maps field names to messages |
| 400 | `INVALID_JSON` | The body is not valid JSON |
| 400 | `FOOD_UNAVAILABLE` | An order contains an unavailable, archived or unknown food (`details.food_ids`). Nothing is created |
| 400 | `CANNOT_DEACTIVATE_SELF` | An administrator tried to deactivate their own account |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password, or deactivated account (same response for each) |
| 401 | `INVALID_TOKEN` | Missing, malformed, expired or revoked token, or a token of the wrong type |
| 403 | `INSUFFICIENT_ROLE` | The signed-in staff role may not do this |
| 404 | `NOT_FOUND` | Unknown id or route. A customer asking for another customer's order also gets this |
| 409 | `INVALID_TRANSITION`, `TERMINAL_STATE`, `STATE_CONFLICT` | Order status rules (see below) |
| 409 | `ORDER_LOCKED` | Fulfillment details can no longer be edited (order is Ready, Completed or Cancelled) |
| 409 | `EMAIL_TAKEN`, `LAST_ADMIN`, `FOOD_ARCHIVED`, `CONFLICT` | Uniqueness and safety rules |
| 413 | `PAYLOAD_TOO_LARGE` | Body over 100 KB |
| 429 | `TOO_MANY_REQUESTS` | Too many failed logins from one IP |
| 500 | `INTERNAL_ERROR` | Unexpected failure. Details are logged on the server, never returned |

## Authentication and roles

- `POST /auth/staff/login` returns `{ token, token_type: "Bearer", expires_in, user }`. Send `Authorization: Bearer <token>`.
- `POST /mobile/auth/login` returns `{ token, token_type, customer }` for customers.
- Tokens last 2 days by default (`JWT_LOGIN_TOKEN_EXPIRY`). There are no refresh tokens: after expiry the user signs in again, and the admin interface signs the user out automatically on the first `401`.
- `POST /auth/logout` ends **all** sessions of that staff user (their `token_version` is increased, so every token issued earlier stops working). The same happens on password reset and deactivation.
- Failed logins are rate limited per IP (default 5 per 15 minutes); successful logins are not counted. The response never says whether an email exists.

| Action | Administrator | Operator | Customer |
| --- | :---: | :---: | :---: |
| Manage staff users (list, view, create, edit, deactivate, reset password) | Yes | No (403) | No (401) |
| View foods, create foods, edit foods, toggle availability | Yes | Yes | No (401) |
| Archive and restore foods | Yes | No (403) | No (401) |
| View orders, change status, edit notes and fulfillment details | Yes | Yes | No (401) |
| View published foods, place orders, view **own** orders (`/mobile`) | No (401) | No (401) | Yes |

Staff and customer tokens are different token types: each is rejected on the other group's routes with `401`.

## Foods

- Create and edit take `name`, `description`, `category`, `price`, `image_url`, `is_available`. The price must be greater than zero with at most two decimals. The currency is set by the server.
- **Archive** (`PATCH /foods/:id/archive`, administrator) is a soft delete: the food disappears from lists, becomes unavailable and cannot be ordered, but existing orders keep their lines. **Restore** brings it back (still unavailable until switched on). Archived foods cannot be edited until restored.
- Changing a price never changes existing orders; each line stores the name and unit price at the time of ordering.

## Orders and the status workflow

| From | Allowed next |
| --- | --- |
| Placed | Accepted, Cancelled |
| Accepted | Preparing, Cancelled |
| Preparing | Ready, Cancelled |
| Ready | Completed |
| Completed, Cancelled | nothing (terminal) |

`POST /orders/:id/transitions` with `{ "to": "Accepted" }` (add `"reason"` when cancelling; it is required). Optional `"from"` is the status the caller believes the order has.

- Invalid or repeated transitions return `409 INVALID_TRANSITION`; changing a terminal order returns `409 TERMINAL_STATE`.
- **Concurrency:** the order row is locked (`SELECT ... FOR UPDATE`) inside the transaction that updates the status and appends the history row. Two simultaneous requests are processed one after the other, so only one of them succeeds and exactly one history row is written. If `from` is sent and no longer matches, the response is `409 STATE_CONFLICT` with `details.current_status`. This is covered by an automated test that fires two requests at the same time.
- Every change appends a row to `order_status_history` (`from_status`, `to_status`, `actor_type`, `actor_id`, `reason`, `created_at`). A database trigger rejects `UPDATE` and `DELETE` on that table.
- `PATCH /orders/:id` edits `notes`, `fulfillment_type` and `delivery_address` only. Items, quantities and prices are never editable. Fulfillment details are locked once the order is Ready, Completed or Cancelled; notes stay editable.
- `GET /orders/:id` returns items, totals, customer, fulfillment and the status timeline with the name of whoever made each change. `GET /orders/metrics` returns counts per status for the dashboard.

## Mobile order flow (curl)

Start the backend with the seed data (`npm run db:reset && npm run start:dev`).

```bash
API=http://localhost:8080/api/v1

# 1. Sign in as the seeded customer
TOKEN=$(curl -s -X POST $API/mobile/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"customer@example.com","password":"password123"}' | node -pe 'JSON.parse(require("fs").readFileSync(0)).token')

# 2. Published foods (available and not archived), paginated
curl -s "$API/mobile/foods?page_size=5" -H "Authorization: Bearer $TOKEN"

# 3. Place an order. The server prices it; any price sent by the client is ignored.
curl -s -X POST $API/mobile/orders -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"items":[{"food_id":1,"quantity":2}],"fulfillment_type":"pickup"}'

# 4. Follow it (use the id from step 3)
curl -s $API/mobile/orders/7 -H "Authorization: Bearer $TOKEN"
curl -s "$API/mobile/orders?status=Placed" -H "Authorization: Bearer $TOKEN"
```

Example order response (abbreviated):

```json
{
  "id": 7, "reference": "FO-000007", "status": "Placed", "fulfillment_type": "pickup",
  "delivery_address": null, "notes": null, "subtotal": "96.00", "total": "96.00", "currency": "AED",
  "items": [{ "id": 10, "food_id": 1, "food_name": "Margherita Pizza", "unit_price": "48.00", "quantity": 2, "line_total": "96.00" }],
  "status_history": [{ "from_status": null, "to_status": "Placed", "reason": null, "created_at": "2026-10-02T09:30:00.000Z" }]
}
```

Rules to know: `fulfillment_type` is `pickup` (default) or `delivery`; delivery requires `delivery_address`. Quantities are whole numbers from 1 to 99; lines for the same food are merged. If any food is unavailable, archived or unknown the whole request fails with `400 FOOD_UNAVAILABLE` and nothing is created. Another customer's order id returns `404`, exactly like a missing one.

The same flow, plus the staff workflow and access-control checks, is in the Postman collection (`npx newman run docs/postman/food-order-platform.postman_collection.json`) and in `backend/scripts/smoke-test.sh`.
