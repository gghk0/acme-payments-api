# Acme Payments API

Payment authorisation and refund service for the Acme commerce platform. Wraps
Stripe and keeps card data out of every other service.

- **Spec:** [`openapi.yaml`](./openapi.yaml)
- **Internal endpoint:** `https://payments.internal.acme.example/v1`
- **Owner:** Acme Payments Team

## Dependency map

### Upstream — services that call Payments

| Caller | Operations consumed | Purpose |
|---|---|---|
| `acme-orders-api` | `POST /v1/charges` | Authorise payment at checkout |
| `acme-orders-api` | `POST /v1/charges/{chargeId}/refunds` | Refund on order cancellation |

### Downstream — services Payments calls

| Dependency | Operation | Purpose |
|---|---|---|
| Stripe *(external)* | `POST /v1/payment_intents` | Authorise / capture |
| Stripe *(external)* | `POST /v1/refunds` | Refund a captured charge |
| Stripe *(external)* | `POST /v1/webhook_endpoints` | Register webhook receivers |

### Data stores

| Store | Engine | Tables |
|---|---|---|
| `acme_payments` | PostgreSQL 15 | `charges`, `refunds`, `webhook_events` |

> **PCI note:** this database is the PCI scope boundary for the platform. No PAN
> or CVV is ever persisted — only processor-side identifiers.

## Blast radius

Payments is a **leaf dependency** for most of the platform: nothing downstream
of it except the external processor. The risk runs the other way —

- `POST /v1/charges` is on the synchronous checkout path. If it degrades, every
  `POST /orders` in the platform degrades with it.
- A breaking change to `CreateChargeRequest` breaks `acme-orders-api` checkout
  immediately, and therefore the public `POST /orders` contract via
  `acme-gateway`.
- The `Stripe-Version` header is pinned per request. Bumping it is a
  platform-wide risk and should be treated as a breaking change.

## Running the Postman CLI here

```bash
postman init
postman workspace create
git add .postman/resources.yaml && git commit -m "link workspace"
```

`postman init` adopts `openapi.yaml` at the repo root automatically. Do **not**
create the `postman/` directory by hand — `init` refuses to run if it already
contains source code.
