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
| `POST /invoices/{order_id}` | rest | loud | The API validation expects customer_name and total_price on the REST payload, but they have been removed upstream in v2, resulting in a ValidationError (HTTP 500). If this loud error is resolved, the status check will silently fail and invoice unpaid orders because PENDING was renamed to AWAITING_PAYMENT, which is missing from NOT_PAYABLE. |
| `GET /payments/{order_id}/status` | grpc | silent | The gRPC OrderSummary payload removed total_price (field 3). Old compiled clients read the missing field as the default value 0.0 with no error, silently evaluating amount_minor as 0. |
| `GET /reports/revenue` | db | loud | Database migration 0002 drops the total_price column. Executing the query containing SUM(total_price) raises an OperationalError/DatabaseError (HTTP 500) because the column no longer exists. |

## Verification

| Check | Name | Result | Details |
|---|---|---|---|
| V1 | consumer unit tests | ✅ | 9 passed, 1 skipped, 1 warning in 0.45s |
| V2 | v2 fixtures match the new contract | ✅ | 3 v2 fixtures match schema |
| V3 | consumer vs upstream v1 (backward compatible) | ✅ | 5/5 passed |
| V4 | consumer vs upstream v2 (new contract) | ✅ | 5/5 passed |
| V5 | Prism contract examples | ✅ | test_rest_contract_examples_parse passed against v1 and v2 |
| V6 | diff scope | ✅ | 16 files changed within scope |

**Verdict:** GREEN
- All checks V1-V6 passed successfully.

## Migration notes

For REST, consumers should read total.amount_minor when present, else round(total_price * 100), and treat both PENDING and AWAITING_PAYMENT as 'not payable'. For gRPC, vendor the v2 proto but re-declare fields 2 and 3 as deprecated in your copy, so one set of stubs can read both server versions, preferring total/customer when HasField is true. For SQL, consumers should check alembic_version.version_num and use SUM(total_price) for 0001 and SUM(total_minor) for 0002. This rollout plan allows consumers to ship tolerant readers that accept v1 and v2 at the same time.

## Rollout plan

1. Merge this tolerant reader.
2. Deploy upstream v2.
3. Remove the v1 paths in a cleanup PR.

## Evidence

- SyncSnitch run `w-20260927-071446-0549`
- Built with IBM Bob - see bob_sessions/ in kshiti26-11/demo
