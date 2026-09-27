SyncSnitch run w-20260927-133555-5098, step S3. You are Subagent 1, the Schema Diff & AST Tracer, running headless from the SyncSnitch website: never ask questions, never edit source code, keep replies short.
Paths are relative to the workspace root. Upstream (the contract owner, read-only): .syncsnitch/work/w-20260927-133555-5098/orders-service. Consumer: .syncsnitch/work/w-20260927-133555-5098/billing-service.
The runner already read everything you need; it is all below. Do not read, list or search other files: answer from this message.
Breaking contract changes (drift.json):
- rest:OrderStatus.PENDING:enum_value_removed: PENDING -> None
- rest:Order.customer_name:property_removed: customer_name -> None
- rest:Order.total_price:property_removed: total_price -> None
- grpc:orders.OrderStatus.1:enum_value_renamed: ORDER_STATUS_PENDING -> ORDER_STATUS_AWAITING_PAYMENT (wire-compatible, source-breaking)
- grpc:orders.OrderSummary.2:field_removed: customer_name -> None
- grpc:orders.OrderSummary.3:field_removed: total_price -> None
- db:orders.customer_name:column_renamed: customer_name -> customer_display_name
- db:orders.status.PENDING:value_renamed: PENDING -> AWAITING_PAYMENT
- db:orders.total_price:column_dropped: total_price -> None
Consumer usages the deterministic scanner found (candidates.json; paths relative to .syncsnitch/work/w-20260927-133555-5098/billing-service):
- billing/models/order.py:7 customer_name [model_field] OrderDTO -> POST /invoices/{order_id}
- billing/models/order.py:8 total_price [model_field] OrderDTO -> POST /invoices/{order_id}
- billing/reports/revenue.sql:1 total_price [sql_column] revenue.sql -> GET /reports/revenue
- billing/services/invoice.py:4 PENDING [literal] NOT_PAYABLE -> POST /invoices/{order_id}
- billing/services/invoice.py:15 total_price [attribute] build_invoice -> POST /invoices/{order_id}
- billing/services/invoice.py:16 total_price [attribute] build_invoice -> POST /invoices/{order_id}
- billing/services/invoice.py:21 customer_name [attribute] build_invoice -> POST /invoices/{order_id}
- billing/services/payments.py:14 total_price [attribute] status_from_summary -> GET /payments/{order_id}/status
- contracts/upstream/orders.proto:18 ORDER_STATUS_PENDING [proto_field] orders.proto
- contracts/upstream/orders.proto:26 customer_name [proto_field] orders.proto
- contracts/upstream/orders.proto:27 total_price [proto_field] orders.proto
- tests/fixtures/order_paid.json:3 customer_name [fixture] order_paid.json (test)
- tests/fixtures/order_paid.json:4 total_price [fixture] order_paid.json (test)
- tests/fixtures/order_unpaid.json:3 customer_name [fixture] order_unpaid.json (test)
- tests/fixtures/order_unpaid.json:4 total_price [fixture] order_unpaid.json (test)
- tests/fixtures/order_unpaid.json:5 PENDING [fixture] order_unpaid.json (test)
- tests/unit/test_payments.py:10 customer_name [keyword] test_paid_order_summary (test)
- tests/unit/test_payments.py:11 total_price [keyword] test_paid_order_summary (test)
- tests/unit/test_revenue.py:11 PENDING [literal] ROWS (test)
- tests/unit/test_revenue.py:28 customer_name [embedded_text] db_url (test)
- tests/unit/test_revenue.py:28 total_price [embedded_text] db_url (test)
Upstream change proposal (text):
RFC-042: Orders API v2 — money in minor units, structured customer, payment-state rename
Owner
Orders team (orders-service)
Status
Approved for rollout
Affects
REST GET /orders/{order_id}, gRPC orders.OrderLookup/GetOrderSummary, database table orders
Known consumers
billing-service (invoices, payment status, revenue report)
1. Summary
Orders v2 changes how an order's money, customer and payment state are represented. The REST payload, the gRPC OrderSummary message and the orders table all change in the same release (PR "Orders API v2").
2. Motivation
Float money caused rounding incidents. Invoices built from total_price: 19.99 were off by one cent after tax in two incidents. Money must be integer minor units plus an ISO 4217 currency.
Customer identity. Consumers need a stable customer_id, not only a display name.
Clearer payment state. PENDING was read as "pending shipment" by some teams. The new name is AWAITING_PAYMENT.
3. Detailed changes
ID
Surface
v1
v2
Consumer impact
CH1
REST + gRPC + DB
total_price (decimal major units, e.g. 19.99)
total { amount_minor: 1999, currency: "USD" } (gRPC: Money total = 6; DB: total_minor + currency)
Breaking: field removed
CH2
REST + gRPC + DB
customer_name
customer { customer_id, display_name } (gRPC: Customer customer = 5; DB: customer_display_name + customer_id)
Breaking: field moved
CH3
REST + gRPC + DB
status PENDING
status AWAITING_PAYMENT (gRPC enum value 1 renamed, same number)
Breaking: value renamed. A consumer that treats PENDING as "not payable" will silently treat unpaid orders as payable
CH4
REST + gRPC + DB
—
optional shipping_eta (RFC 3339)
Non-breaking: additive
MIG
DB
migration 0001
migration 0002_money_customer_status
Readers of orders.total_price / orders.customer_name break
gRPC notes: fields 2 (customer_name) and 3 (total_price) are removed and reserved. Their numbers will never be reused, so a consumer may keep reading them from old servers during the rollout.
4. Compatibility
v2 is not backward compatible for consumers. Old clients get no error, just missing data. Old REST clients fail on the missing total_price. Old gRPC clients read total_price as 0.0, which is silent. Old SQL reports fail on the dropped column.
5. Rollout plan
Consumers ship tolerant readers that accept v1 and v2 at the same time (see §6).
Orders deploys v2 (migration 0002 runs at deploy).
After one week, consumers remove the v1 code paths in a cleanup PR.
6. Migration guidance for consumers
REST: read total.amount_minor when present, else round(total_price * 100). Read customer.display_name when present, else customer_name. Treat both PENDING and AWAITING_PAYMENT as "not payable".
gRPC: vendor the v2 proto but re-declare fields 2 and 3 as [deprecated = true] in your copy, so one set of stubs can read both server versions. Prefer total / customer when HasField(...) is true.
SQL: check alembic_version.version_num. Use SUM(total_price) for 0001 and SUM(total_minor) for 0002.
Keep your own public API unchanged.
7. Open questions
Should shipping_eta become required once all carriers report it? (Not in v2.)
8. Approvals
Orders lead ✅ · Platform architecture ✅ · Billing lead ⏳ (needs a companion change before rollout)
Consumer files (current content; paths under .syncsnitch/work/w-20260927-133555-5098/billing-service):
--- billing/models/order.py
from datetime import datetime
from pydantic import BaseModel


class OrderDTO(BaseModel):
    order_id: str
    customer_name: str
    total_price: float
    status: str
    created_at: datetime
--- billing/reports/revenue.sql
SELECT date(created_at) AS day, SUM(total_price) AS revenue
FROM orders
WHERE status IN ('PAID', 'SHIPPED')
GROUP BY day
ORDER BY day
--- billing/services/invoice.py
from decimal import Decimal, ROUND_HALF_UP

TAX_RATE = Decimal("0.0825")
NOT_PAYABLE = {"PENDING", "CANCELLED"}


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def build_invoice(order) -> dict | None:
    if order.status in NOT_PAYABLE:
        return None

    subtotal = _round_half_up(Decimal(str(order.total_price)) * 100)
    tax = _round_half_up(Decimal(str(order.total_price)) * 100 * TAX_RATE)
    total = subtotal + tax

    return {
        "order_id": order.order_id,
        "customer": order.customer_name,
        "subtotal_minor": subtotal,
        "tax_minor": tax,
        "total_minor": total,
        "currency": "USD",
    }
--- billing/services/payments.py
from decimal import Decimal, ROUND_HALF_UP

PAID_STATES = {"ORDER_STATUS_PAID", "ORDER_STATUS_SHIPPED"}


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def status_from_summary(summary) -> dict:
    field = type(summary).DESCRIPTOR.fields_by_name["status"]
    status_name = field.enum_type.values_by_number[summary.status].name

    amount_minor = _round_half_up(Decimal(str(summary.total_price)) * 100)

    return {
        "order_id": summary.order_id,
        "paid": status_name in PAID_STATES,
        "amount_minor": amount_minor,
        "currency": "USD",
    }
--- contracts/upstream/orders.proto
// Orders gRPC contract v1 - owned by orders-service (upstream).
// Reference copy for SyncSnitch - do not edit.
syntax = "proto3";

package orders;

service OrderLookup {
  // Returns NOT_FOUND for an unknown order_id.
  rpc GetOrderSummary(GetOrderSummaryRequest) returns (OrderSummary);
}

message GetOrderSummaryRequest {
  string order_id = 1;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}

message OrderSummary {
  string order_id = 1;
  string customer_name = 2;
  double total_price = 3;  // major units (USD), e.g. 19.99
  OrderStatus status = 4;
}
--- billing/api.py
from fastapi import FastAPI
from fastapi.responses import JSONResponse

from . import config
from .clients.orders_rest import OrdersRestClient, OrderNotFound as RestOrderNotFound
from .clients.orders_grpc import get_order_summary, OrderNotFound as GrpcOrderNotFound
from .contract_entrypoints import invoice_from_order_payload
from .services.payments import status_from_summary
from .reports.revenue import run_revenue_report


def create_app(
    orders_rest_url=None,
    orders_rest_transport=None,
    orders_grpc_addr=None,
    reports_db_url=None,
) -> FastAPI:
    app = FastAPI(title="billing-service", version="1.0.0")

    _rest_url = orders_rest_url or config.orders_rest_url()
    _grpc_addr = orders_grpc_addr or config.orders_grpc_addr()
    _db_url = reports_db_url or config.reports_db_url()

    rest_client = OrdersRestClient(base_url=_rest_url, transport=orders_rest_transport)

    @app.get("/health")
    async def health():
        return {"status": "ok"}

    @app.post("/invoices/{order_id}", status_code=201)
    async def create_invoice(order_id: str):
        try:
            payload = await rest_client.get_order(order_id)
        except RestOrderNotFound:
            return JSONResponse(status_code=404, content={"detail": "order not found"})

        invoice = invoice_from_order_payload(payload)
        if invoice is None:
            return JSONResponse(status_code=409, content={"detail": "order not payable"})

        return JSONResponse(status_code=201, content=invoice)

    @app.get("/payments/{order_id}/status")
    def payment_status(order_id: str):
        try:
            summary = get_order_summary(_grpc_addr, order_id)
        except GrpcOrderNotFound:
            return JSONResponse(status_code=404, content={"detail": "order not found"})

        return status_from_summary(summary)

    @app.get("/reports/revenue")
    def revenue_report():
        return run_revenue_report(_db_url)

    return app
--- billing/contract_entrypoints.py
from .models.order import OrderDTO
from .services.invoice import build_invoice


def invoice_from_order_payload(payload: dict) -> dict | None:
    return build_invoice(OrderDTO.model_validate(payload))
--- billing/reports/revenue.py
from decimal import Decimal, ROUND_HALF_UP
from pathlib import Path

from sqlalchemy import create_engine, text


_SQL = (Path(__file__).parent / "revenue.sql").read_text()


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def run_revenue_report(db_url: str) -> list[dict]:
    engine = create_engine(db_url)
    try:
        with engine.connect() as conn:
            rows = conn.execute(text(_SQL)).fetchall()
    finally:
        engine.dispose()

    return [
        {
            "day": str(row.day)[:10],
            "revenue_minor": _round_half_up(Decimal(str(row.revenue)) * 100),
        }
        for row in rows
    ]
--- end of files
1. Confirm or reject each usage and add any the scanner missed, from the files above.
2. In your first reply, call write_file once to write .syncsnitch/runs/w-20260927-133555-5098/impact.json with exactly this shape:
{"run_id": "w-20260927-133555-5098", "mapping": [{"change_ids": ["..."], "old": "...", "new": "...", "rule": "..."}], "affected": [{"file": "...", "line": 1, "symbol": "...", "change_ids": ["..."], "surface": "rest|grpc|db|test", "failure": "loud|silent|none", "endpoint": "METHOD /path or null", "fix": "..."}], "endpoints": [{"endpoint": "METHOD /path", "surface": "rest|grpc|db", "failure": "loud|silent", "why": "..."}], "migration_notes": "3-6 sentences quoting the proposal"}
"loud" = the endpoint fails with an error; "silent" = it returns wrong data without any error. List every consumer endpoint that breaks.
3. Then reply with a 5-line summary.