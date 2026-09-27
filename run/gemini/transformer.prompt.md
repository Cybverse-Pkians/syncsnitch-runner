SyncSnitch run w-20260927-160432-7042, step S4 fix round. You are Subagent 2, the Downstream Code Transformer, running headless from the SyncSnitch website: never ask questions, keep replies short.
Paths are relative to the workspace root. The consumer .syncsnitch/work/w-20260927-160432-7042/billing-service is inside the git clone .syncsnitch/work/w-20260927-160432-7042, on branch syncsnitch/w-20260927-160432-7042. The upstream .syncsnitch/work/w-20260927-160432-7042/orders-service is read-only. The rules in .bob/rules-syncsnitch-transformer/tolerant-reader.md still apply.
The Contract Verifier found these failing checks:
- V4 consumer vs upstream v2 (new contract): 4/5 passed (failures: test_payment_status_uses_real_amount)
Apply these fix instructions, editing only files inside .syncsnitch/work/w-20260927-160432-7042/billing-service:
- In billing/adapters/orders_v2.py or where customer display_name is read from upstream v2, replace references to orders.Customer.display_name with the correct v2 customer field or handle the absence of display_name safely so that test_payment_status_uses_real_amount passes.
Facts the runner checked; they win over any instruction above (skip an instruction that edits tests/integration/ or changes a billing response):
- Failing contract test V4 tests.integration.test_contract::test_payment_status_uses_real_amount against upstream v2: ValueError: Field orders.Customer.display_name does not have presence.
- Scanner hits in files the branch has not changed yet (each must still work with v1 and v2): billing/reports/revenue.sql:1 (total_price)
- The tests in tests/integration/ are the contract spec: fix the consumer code under .syncsnitch/work/w-20260927-160432-7042/billing-service, never the test, and never add or rename a field in a billing response. An actual value of 0 or empty usually means the consumer still reads a field the new upstream removed; read the new fields through the adapter where that value is built.
Current content of the files involved (paths under .syncsnitch/work/w-20260927-160432-7042/billing-service; the runner read them from syncsnitch/w-20260927-160432-7042):
--- billing/models/order.py
from datetime import datetime
from pydantic import BaseModel, Field


class OrderDTO(BaseModel):
    order_id: str
    customer_name: str = Field(default="")
    total_price: float = Field(default=0.0)
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
NOT_PAYABLE = {"PENDING", "AWAITING_PAYMENT", "CANCELLED"}


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def build_invoice(order) -> dict | None:
    if order.status in NOT_PAYABLE:
        return None

    subtotal = order.amount_minor
    tax = _round_half_up(Decimal(str(subtotal)) * TAX_RATE)
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
from billing.adapters.orders_contract import from_summary

PAID_STATES = {"ORDER_STATUS_PAID", "ORDER_STATUS_SHIPPED", "PAID", "SHIPPED"}


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def status_from_summary(summary) -> dict:
    view = from_summary(summary)

    return {
        "order_id": view.order_id,
        "paid": view.status in {"PAID", "SHIPPED"},
        "amount_minor": view.amount_minor,
        "currency": view.currency,
    }
--- contracts/upstream/orders.proto
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
  string customer_name = 2 [deprecated = true];
  double total_price = 3 [deprecated = true];  // major units (USD), e.g. 19.99
  string order_id = 1;
  OrderStatus status = 4;
  Customer customer = 5;
  Money total = 6;
  string shipping_eta = 7;  // RFC 3339 timestamp, empty when unknown
}
--- billing/adapters/__init__.py

--- billing/adapters/orders_contract.py
from dataclasses import dataclass
from decimal import Decimal, ROUND_HALF_UP


def normalize_status(value: str) -> str:
    if value.startswith("ORDER_STATUS_"):
        value = value[len("ORDER_STATUS_"):]
    if value == "PENDING":
        return "AWAITING_PAYMENT"
    return value


@dataclass(frozen=True)
class OrderView:
    order_id: str
    customer_name: str
    amount_minor: int
    currency: str
    status: str
    created_at: str = None


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def from_rest(payload: dict) -> OrderView:
    order_id = payload["order_id"]
    status = normalize_status(payload["status"])
    created_at = payload.get("created_at")

    if "customer" in payload and isinstance(payload["customer"], dict):
        customer_name = payload["customer"].get("display_name")
    else:
        customer_name = payload.get("customer_name")

    if "total" in payload and isinstance(payload["total"], dict):
        total_dict = payload["total"]
        amount_minor = total_dict["amount_minor"]
        currency = total_dict.get("currency", "USD")
    else:
        tp = payload.get("total_price", 0)
        amount_minor = _round_half_up(Decimal(str(tp)) * 100)
        currency = "USD"

    return OrderView(
        order_id=order_id,
        customer_name=customer_name,
        amount_minor=amount_minor,
        currency=currency,
        status=status,
        created_at=created_at,
    )


def from_summary(summary) -> OrderView:
    order_id = summary.order_id
    
    field = type(summary).DESCRIPTOR.fields_by_name["status"]
    raw_status = field.enum_type.values_by_number[summary.status].name
    status = normalize_status(raw_status)

    if summary.HasField("customer") and summary.customer.HasField("display_name") or getattr(summary, "customer", None) and summary.customer.display_name:
        customer_name = summary.customer.display_name
    elif hasattr(summary, "customer_name") and summary.customer_name:
        customer_name = summary.customer_name
    else:
        customer_name = getattr(summary.customer, "display_name", "")

    if summary.HasField("total"):
        amount_minor = summary.total.amount_minor
        currency = summary.total.currency or "USD"
    elif hasattr(summary, "total_price") and summary.total_price:
        amount_minor = _round_half_up(Decimal(str(summary.total_price)) * 100)
        currency = "USD"
    else:
        amount_minor = 0
        currency = "USD"

    shipping_eta = getattr(summary, "shipping_eta", None)

    return OrderView(
        order_id=order_id,
        customer_name=customer_name,
        amount_minor=amount_minor,
        currency=currency,
        status=status,
    )
--- billing/contract_entrypoints.py
from .adapters.orders_contract import from_rest
from .services.invoice import build_invoice


def invoice_from_order_payload(payload: dict) -> dict | None:
    view = from_rest(payload)
    return build_invoice(view)
--- billing/reports/revenue.py
from decimal import Decimal, ROUND_HALF_UP
from pathlib import Path

from sqlalchemy import create_engine, text


_SQL_V1 = (Path(__file__).parent / "revenue_v1.sql").read_text()
_SQL_V2 = (Path(__file__).parent / "revenue_v2.sql").read_text()


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def _schema_version(conn) -> str:
    try:
        return str(conn.execute(text("SELECT version_num FROM alembic_version")).scalar() or "")
    except Exception:
        conn.rollback()
        return ""


def run_revenue_report(db_url: str) -> list[dict]:
    engine = create_engine(db_url)
    try:
        with engine.connect() as conn:
            ver = _schema_version(conn)
            sql = _SQL_V2 if ver >= "0002" else _SQL_V1
            rows = conn.execute(text(sql)).fetchall()
    finally:
        engine.dispose()

    if ver >= "0002":
        return [
            {
                "day": str(row.day)[:10],
                "revenue_minor": int(row.revenue or 0),
            }
            for row in rows
        ]
    else:
        return [
            {
                "day": str(row.day)[:10],
                "revenue_minor": _round_half_up(Decimal(str(row.revenue)) * 100),
            }
            for row in rows
        ]
--- billing/reports/revenue_v1.sql
SELECT date(created_at) AS day, SUM(total_price) AS revenue
FROM orders
WHERE status IN ('PAID', 'SHIPPED')
GROUP BY day
ORDER BY day
--- billing/reports/revenue_v2.sql
SELECT date(created_at) AS day, SUM(total_minor) AS revenue
FROM orders
WHERE status IN ('PAID', 'SHIPPED')
GROUP BY day
ORDER BY day
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
--- end of files
Then run cd .syncsnitch/work/w-20260927-160432-7042/billing-service && uv run pytest -q until green, and commit on syncsnitch/w-20260927-160432-7042: cd .syncsnitch/work/w-20260927-160432-7042/billing-service && git add -A . && git commit -q -m "fix(contract): address the Contract Verifier findings (SyncSnitch w-20260927-160432-7042)" -m "SyncSnitch-Agent: Gemini gemini-3.5-flash-lite (w-20260927-160432-7042)"
Do not push. Reply with one line.