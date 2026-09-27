SyncSnitch run w-20260927-103603-c9e0, step S4. You are Subagent 2, the Downstream Code Transformer, running headless from the SyncSnitch website: never ask questions, keep replies short.
Paths are relative to the workspace root. The consumer .syncsnitch/work/w-20260927-103603-c9e0/billing-service is inside the git clone .syncsnitch/work/w-20260927-103603-c9e0, already on branch syncsnitch/w-20260927-103603-c9e0. The upstream .syncsnitch/work/w-20260927-103603-c9e0/orders-service is read-only.
1. Apply .bob/rules-syncsnitch-transformer/tolerant-reader.md exactly to the impact below (.syncsnitch/runs/w-20260927-103603-c9e0/impact.json), with UPSTREAM=.syncsnitch/work/w-20260927-103603-c9e0/orders-service, HEAD_REF=159ef5cecf0e320f87ae52eb40e125c60463d956, BASE_REF=15f95af008e5060450fa81ff0a58ed50627b1b2e, UPSTREAM_REPO=kshiti26-11/demo, PR_NUMBER=null, RUN_ID=w-20260927-103603-c9e0. Edit only files inside .syncsnitch/work/w-20260927-103603-c9e0/billing-service.
   Required coverage, checked by the runner (it wins over impact.json):
- Breaking contract surfaces (drift.json): REST, gRPC, DB. Apply tolerant-reader pattern 1 (REST adapter), 2 (gRPC proto + stubs, and from_summary), 3 (SQL) even where impact.json leaves one out.
- Every consumer file the deterministic scanner hit must work with v1 and v2 when you are done (paths under .syncsnitch/work/w-20260927-103603-c9e0/billing-service): billing/models/order.py:7,8 (customer_name, total_price); billing/reports/revenue.sql:1 (total_price); billing/services/invoice.py:4,15,16,21 (PENDING, customer_name, total_price); billing/services/payments.py:14 (total_price); contracts/upstream/orders.proto:18,26,27 (ORDER_STATUS_PENDING, customer_name, total_price).
- Never edit the existing tests in .syncsnitch/work/w-20260927-103603-c9e0/billing-service/tests/integration/: they are the contract spec that V3/V4 run against the real upstream (V6 fails if they change). Keep every billing response JSON exactly as it is: no new, renamed or removed fields.
   The runner already read the files below: use them instead of reading, listing or searching (git show of the upstream is not needed).
   Batch your work: in every reply call write_file for ALL the files you can already write (adapter, proto, SQL, fixtures, services together), then regen_stubs and pytest together.
The Tracer's impact map:
--- .syncsnitch/runs/w-20260927-103603-c9e0/impact.json
{
  "run_id": "w-20260927-103603-c9e0",
  "mapping": [
    {
      "change_ids": ["rest:OrderStatus.PENDING:enum_value_removed"],
      "old": "PENDING",
      "new": "None",
      "rule": "enum_value_removed"
    },
    {
      "change_ids": ["rest:Order.customer_name:property_removed"],
      "old": "customer_name",
      "new": "None",
      "rule": "property_removed"
    },
    {
      "change_ids": ["rest:Order.total_price:property_removed"],
      "old": "total_price",
      "new": "None",
      "rule": "property_removed"
    },
    {
      "change_ids": ["grpc:orders.OrderStatus.1:enum_value_renamed"],
      "old": "ORDER_STATUS_PENDING",
      "new": "ORDER_STATUS_AWAITING_PAYMENT",
      "rule": "enum_value_renamed"
    },
    {
      "change_ids": ["grpc:orders.OrderSummary.2:field_removed"],
      "old": "customer_name",
      "new": "None",
      "rule": "field_removed"
    },
    {
      "change_ids": ["grpc:orders.OrderSummary.3:field_removed"],
      "old": "total_price",
      "new": "None",
      "rule": "field_removed"
    },
    {
      "change_ids": ["db:orders.customer_name:column_renamed"],
      "old": "customer_name",
      "new": "customer_display_name",
      "rule": "column_renamed"
    },
    {
      "change_ids": ["db:orders.status.PENDING:value_renamed"],
      "old": "PENDING",
      "new": "AWAITING_PAYMENT",
      "rule": "value_renamed"
    },
    {
      "change_ids": ["db:orders.total_price:column_dropped"],
      "old": "total_price",
      "new": "None",
      "rule": "column_dropped"
    }
  ],
  "affected": [
    {
      "file": "billing/models/order.py",
      "line": 7,
      "symbol": "customer_name",
      "change_ids": ["rest:Order.customer_name:property_removed"],
      "surface": "rest",
      "failure": "loud",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Read customer.display_name when present, else customer_name."
    },
    {
      "file": "billing/models/order.py",
      "line": 8,
      "symbol": "total_price",
      "change_ids": ["rest:Order.total_price:property_removed"],
      "surface": "rest",
      "failure": "loud",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Read total.amount_minor when present, else round(total_price * 100)."
    },
    {
      "file": "billing/reports/revenue.sql",
      "line": 1,
      "symbol": "total_price",
      "change_ids": ["db:orders.total_price:column_dropped"],
      "surface": "db",
      "failure": "loud",
      "endpoint": "GET /reports/revenue",
      "fix": "Check alembic_version.version_num. Use SUM(total_price) for 0001 and SUM(total_minor) for 0002."
    },
    {
      "file": "billing/services/invoice.py",
      "line": 4,
      "symbol": "PENDING",
      "change_ids": ["rest:OrderStatus.PENDING:enum_value_removed"],
      "surface": "rest",
      "failure": "silent",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Treat both PENDING and AWAITING_PAYMENT as 'not payable'."
    },
    {
      "file": "billing/services/invoice.py",
      "line": 15,
      "symbol": "total_price",
      "change_ids": ["rest:Order.total_price:property_removed"],
      "surface": "rest",
      "failure": "loud",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Read total.amount_minor when present, else round(total_price * 100)."
    },
    {
      "file": "billing/services/invoice.py",
      "line": 16,
      "symbol": "total_price",
      "change_ids": ["rest:Order.total_price:property_removed"],
      "surface": "rest",
      "failure": "loud",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Read total.amount_minor when present, else round(total_price * 100)."
    },
    {
      "file": "billing/services/invoice.py",
      "line": 21,
      "symbol": "customer_name",
      "change_ids": ["rest:Order.customer_name:property_removed"],
      "surface": "rest",
      "failure": "loud",
      "endpoint": "POST /invoices/{order_id}",
      "fix": "Read customer.display_name when present, else customer_name."
    },
    {
      "file": "billing/services/payments.py",
      "line": 14,
      "symbol": "total_price",
      "change_ids": ["grpc:orders.OrderSummary.3:field_removed"],
      "surface": "grpc",
      "failure": "silent",
      "endpoint": "GET /payments/{order_id}/status",
      "fix": "Vendor the v2 proto but re-declare fields 2 and 3 as deprecated in your copy, check HasField('total')."
    },
    {
      "file": "contracts/upstream/orders.proto",
      "line": 18,
      "symbol": "ORDER_STATUS_PENDING",
      "change_ids": ["grpc:orders.OrderStatus.1:enum_value_renamed"],
      "surface": "grpc",
      "failure": "none",
      "endpoint": null,
      "fix": "Vendor the v2 proto with ORDER_STATUS_AWAITING_PAYMENT."
    },
    {
      "file": "contracts/upstream/orders.proto",
      "line": 26,
      "symbol": "customer_name",
      "change_ids": ["grpc:orders.OrderSummary.2:field_removed"],
      "surface": "grpc",
      "failure": "none",
      "endpoint": null,
      "fix": "Vendor the v2 proto with customer field."
    },
    {
      "file": "contracts/upstream/orders.proto",
      "line": 27,
      "symbol": "total_price",
      "change_ids": ["grpc:orders.OrderSummary.3:field_removed"],
      "surface": "grpc",
      "failure": "none",
      "endpoint": null,
      "fix": "Vendor the v2 proto with total field."
    },
    {
      "file": "tests/fixtures/order_paid.json",
      "line": 3,
      "symbol": "customer_name",
      "change_ids": ["rest:Order.customer_name:property_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update fixture to have customer display_name structure."
    },
    {
      "file": "tests/fixtures/order_paid.json",
      "line": 4,
      "symbol": "total_price",
      "change_ids": ["rest:Order.total_price:property_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update fixture to have total amount_minor structure."
    },
    {
      "file": "tests/fixtures/order_unpaid.json",
      "line": 3,
      "symbol": "customer_name",
      "change_ids": ["rest:Order.customer_name:property_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update fixture to have customer display_name structure."
    },
    {
      "file": "tests/fixtures/order_unpaid.json",
      "line": 4,
      "symbol": "total_price",
      "change_ids": ["rest:Order.total_price:property_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update fixture to have total amount_minor structure."
    },
    {
      "file": "tests/fixtures/order_unpaid.json",
      "line": 5,
      "symbol": "PENDING",
      "change_ids": ["rest:OrderStatus.PENDING:enum_value_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update status value to AWAITING_PAYMENT in test fixture."
    },
    {
      "file": "tests/unit/test_payments.py",
      "line": 10,
      "symbol": "customer_name",
      "change_ids": ["grpc:orders.OrderSummary.2:field_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Mock gRPC field customer display_name instead of customer_name."
    },
    {
      "file": "tests/unit/test_payments.py",
      "line": 11,
      "symbol": "total_price",
      "change_ids": ["grpc:orders.OrderSummary.3:field_removed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Mock gRPC field total amount_minor instead of total_price."
    },
    {
      "file": "tests/unit/test_revenue.py",
      "line": 11,
      "symbol": "PENDING",
      "change_ids": ["db:orders.status.PENDING:value_renamed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update test assert or seed data to use AWAITING_PAYMENT."
    },
    {
      "file": "tests/unit/test_revenue.py",
      "line": 28,
      "symbol": "customer_name",
      "change_ids": ["db:orders.customer_name:column_renamed"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update test db schema or test sql assertion."
    },
    {
      "file": "tests/unit/test_revenue.py",
      "line": 28,
      "symbol": "total_price",
      "change_ids": ["db:orders.total_price:column_dropped"],
      "surface": "test",
      "failure": "none",
      "endpoint": null,
      "fix": "Update test db schema or test sql assertion."
    }
  ],
  "endpoints": [
    {
      "endpoint": "POST /invoices/{order_id}",
      "surface": "rest",
      "failure": "loud",
      "why": "Parsing the updated REST payload raises a Pydantic ValidationError in OrderDTO because customer_name and total_price are no longer present. Additionally, treating PENDING as 'not payable' fails silently since AWAITING_PAYMENT is now used."
    },
    {
      "endpoint": "GET /payments/{order_id}/status",
      "surface": "grpc",
      "failure": "silent",
      "why": "Accessing 'summary.total_price' from the gRPC OrderSummary message silently defaults to 0.0 under the new schema, as the field has been removed, returning an incorrect minor amount."
    },
    {
      "endpoint": "GET /reports/revenue",
      "surface": "db",
      "failure": "loud",
      "why": "The database column 'total_price' has been dropped from the 'orders' table, causing the SELECT query in revenue.sql to throw a column-not-found database error."
    }
  ],
  "migration_notes": "Orders v2 introduces breaking contract changes to money, customer, and status representations across REST, gRPC, and DB. To ensure backward compatibility during rollout, the migration guidance recommends consumers implement tolerant readers that support both v1 and v2. For REST, consumers should read 'total.amount_minor' and 'customer.display_name' if available, defaulting to their v1 counterparts. For gRPC, clients should vendor the v2 proto but mark deprecated fields as 'deprecated = true' in their local copies. For DB/SQL, reports should check the 'alembic_version' to use 'total_price' for 0001 and 'total_minor' for 0002."
}
--- end of files
New upstream contract files:
--- orders-service/contracts/openapi.yaml (upstream head 159ef5cecf0e, examples and schemas only)
              examples:
                ok:
                  value:
                    status: ok
              examples:
                all:
                  value:
                    - order_id: o-1001
                      customer:
                        customer_id: c-1001
                        display_name: Ada Lovelace
                      total:
                        amount_minor: 1999
                        currency: USD
                      status: PAID
                      created_at: "2026-09-01T10:00:00Z"
                    - order_id: o-1002
                      customer:
                        customer_id: c-1002
                        display_name: Alan Turing
                      total:
                        amount_minor: 500
                        currency: USD
                      status: AWAITING_PAYMENT
                      created_at: "2026-09-02T11:00:00Z"
                    - order_id: o-1003
                      customer:
                        customer_id: c-1003
                        display_name: Grace Hopper
                      total:
                        amount_minor: 12050
                        currency: USD
                      status: SHIPPED
                      created_at: "2026-09-03T12:00:00Z"
                      shipping_eta: "2026-09-05T12:00:00Z"
                    - order_id: o-1004
                      customer:
                        customer_id: c-1004
                        display_name: Edsger Dijkstra
                      total:
                        amount_minor: 4200
                        currency: USD
                      status: CANCELLED
                      created_at: "2026-09-04T13:00:00Z"
              examples:
                paid:
                  value:
                    order_id: o-1001
                    customer:
                      customer_id: c-1001
                      display_name: Ada Lovelace
                    total:
                      amount_minor: 1999
                      currency: USD
                    status: PAID
                    created_at: "2026-09-01T10:00:00Z"
                unpaid:
                  value:
                    order_id: o-1002
                    customer:
                      customer_id: c-1002
                      display_name: Alan Turing
                    total:
                      amount_minor: 500
                      currency: USD
                    status: AWAITING_PAYMENT
                    created_at: "2026-09-02T11:00:00Z"
              examples:
                not_found:
                  value:
                    detail: order not found
  schemas:
    Health:
      type: object
      additionalProperties: false
      required: [status]
      properties:
        status:
          type: string
          const: ok
    OrderStatus:
      type: string
      enum: [AWAITING_PAYMENT, PAID, SHIPPED, CANCELLED]
    Money:
      type: object
      additionalProperties: false
      required: [amount_minor, currency]
      properties:
        amount_minor:
          type: integer
          minimum: 0
          description: Amount in minor units (cents), e.g. 1999 = 19.99 USD
        currency:
          type: string
          pattern: "^[A-Z]{3}$"
    Customer:
      type: object
      additionalProperties: false
      required: [customer_id, display_name]
      properties:
        customer_id:
          type: string
          pattern: "^c-[0-9]{4}$"
        display_name:
          type: string
          minLength: 1
    Order:
      type: object
      additionalProperties: false
      required: [order_id, customer, total, status, created_at]
      properties:
        order_id:
          type: string
          pattern: "^o-[0-9]{4}$"
        customer:
          $ref: "#/components/schemas/Customer"
        total:
          $ref: "#/components/schemas/Money"
        status:
          $ref: "#/components/schemas/OrderStatus"
        created_at:
          type: string
          format: date-time
        shipping_eta:
          type: string
          format: date-time
          description: Optional; present only when the order has shipped
    Error:
      type: object
      additionalProperties: false
      required: [detail]
      properties:
        detail:
          type: string
--- orders-service/contracts/orders.proto (upstream head 159ef5cecf0e)
// Orders gRPC contract v2 - owned by orders-service (upstream).
// v2 = money in minor units, structured customer, PENDING renamed to
// AWAITING_PAYMENT (same wire number), optional shipping_eta.
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
  ORDER_STATUS_AWAITING_PAYMENT = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}

message Money {
  int64 amount_minor = 1;  // minor units, e.g. 1999 = 19.99 USD
  string currency = 2;     // ISO 4217, e.g. "USD"
}

message Customer {
  string customer_id = 1;
  string display_name = 2;
}

message OrderSummary {
  reserved 2, 3;
  reserved "customer_name", "total_price";
  string order_id = 1;
  OrderStatus status = 4;
  Customer customer = 5;
  Money total = 6;
  string shipping_eta = 7;  // RFC 3339 timestamp, empty when unknown
}
--- end of files
Consumer files (current content; paths under .syncsnitch/work/w-20260927-103603-c9e0/billing-service):
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
2. Run the consumer unit tests until they pass: cd .syncsnitch/work/w-20260927-103603-c9e0/billing-service && uv run pytest -q
3. Commit on syncsnitch/w-20260927-103603-c9e0: cd .syncsnitch/work/w-20260927-103603-c9e0/billing-service && git add -A . && git commit -q -m "fix(contract): tolerant reader for the orders-service contract change (SyncSnitch w-20260927-103603-c9e0)" -m "SyncSnitch-Agent: Gemini gemini-3.5-flash (w-20260927-103603-c9e0)"
   Do not push.
4. Reply with one line: files changed, insertions, deletions.