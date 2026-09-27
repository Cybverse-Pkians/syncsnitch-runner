## Why this PR exists

Upstream PR https://github.com/kshiti26-11/demo/compare/15f95af008e5...main changes the Orders contract
(`15f95af008e5060450fa81ff0a58ed50627b1b2e` → `main`): **9 breaking** of 26 changes
(REST 3 · gRPC 3 · DB 3).
This companion PR makes the consumer a tolerant reader, so it works against contract v1 **and** v2 during the rollout.

## Contract drift

| Surface | Kind | Location | Old → New | Breaking |
|---|---|---|---|---|
| rest | schema_added | `Money` | – → Money | no |
| rest | schema_added | `Customer` | – → Customer | no |
| rest | enum_value_removed | `OrderStatus.PENDING` | PENDING → – | ⚠️ yes |
| rest | enum_value_added | `OrderStatus.AWAITING_PAYMENT` | – → AWAITING_PAYMENT | no |
| rest | property_removed | `Order.customer_name` | customer_name → – | ⚠️ yes |
| rest | property_removed | `Order.total_price` | total_price → – | ⚠️ yes |
| rest | property_added | `Order.customer` | – → customer | no |
| rest | property_added | `Order.total` | – → total | no |
| rest | property_added | `Order.shipping_eta` | – → shipping_eta | no |
| grpc | message_added | `orders.Customer` | – → orders.Customer | no |
| grpc | message_added | `orders.Money` | – → orders.Money | no |
| grpc | enum_value_renamed | `orders.OrderStatus.1` | ORDER_STATUS_PENDING → ORDER_STATUS_AWAITING_PAYMENT | ⚠️ yes |
| grpc | field_removed | `orders.OrderSummary.2` | customer_name → – | ⚠️ yes |
| grpc | field_removed | `orders.OrderSummary.3` | total_price → – | ⚠️ yes |
| grpc | field_added | `orders.OrderSummary.5` | – → customer | no |
| grpc | field_added | `orders.OrderSummary.6` | – → total | no |
| grpc | field_added | `orders.OrderSummary.7` | – → shipping_eta | no |
| db | column_added | `orders.total_minor` | – → total_minor | no |
| db | column_added | `orders.currency` | – → currency | no |
| db | column_added | `orders.customer_id` | – → customer_id | no |
| db | column_added | `orders.shipping_eta` | – → shipping_eta | no |
| db | column_renamed | `orders.customer_name` | customer_name → customer_display_name | ⚠️ yes |
| db | value_renamed | `orders.status.PENDING` | PENDING → AWAITING_PAYMENT | ⚠️ yes |
| db | column_dropped | `orders.total_price` | total_price → – | ⚠️ yes |

## Impact

| Endpoint | Surface | Failure | Why |
|---|---|---|---|
| `POST /invoices/{order_id}` | rest | loud | Pydantic validation and invoice attribute access fail on removed fields, and PENDING status change causes silent incorrect payable checks. |
| `GET /payments/{order_id}/status` | grpc | silent | Old gRPC client reads removed field total_price as 0.0 without error, resulting in incorrect amount_minor values. |
| `GET /reports/revenue` | db | loud | SQL query references dropped column total_price, raising database execution errors. |

## Verification

| Check | Name | Result | Details |
|---|---|---|---|
| V1 | consumer unit tests | ✅ | 8 passed, 1 skipped, 6 warnings in 0.49s |
| V2 | v2 fixtures match the new contract | ✅ | 3 v2 fixtures match schema |
| V3 | consumer vs upstream v1 (backward compatible) | ✅ | 5/5 passed |
| V4 | consumer vs upstream v2 (new contract) | ✅ | 5/5 passed |
| V5 | Prism contract examples | ✅ | test_rest_contract_examples_parse passed against v1 and v2 |
| V6 | diff scope | ✅ | 14 files changed within scope |

**Verdict:** GREEN
- All checks V1-V6 passed successfully.

## Migration notes

Orders v2 changes how money, customer and payment state are represented across REST, gRPC and DB. Consumers must ship tolerant readers that accept both v1 and v2 simultaneously during rollout. Specifically, read total.amount_minor when present, else round(total_price * 100). Read customer.display_name when present, else customer_name. Treat both PENDING and AWAITING_PAYMENT as not payable. For SQL reports, check alembic_version and use appropriate columns.

## Rollout plan

1. Merge this tolerant reader.
2. Deploy upstream v2.
3. Remove the v1 paths in a cleanup PR.

## Evidence

- SyncSnitch run `w-20260927-120026-5c8d`
- Built with IBM Bob - see bob_sessions/ in kshiti26-11/demo
