SyncSnitch run w-20260928-033249-fa0f, step S4 fix round. You are Subagent 2, the Downstream Code Transformer, running headless from the SyncSnitch website: never ask questions, keep replies short.
Paths are relative to the workspace root. The consumer .syncsnitch/work/w-20260928-033249-fa0f/billing-service is inside the git clone .syncsnitch/work/w-20260928-033249-fa0f, on branch syncsnitch/w-20260928-033249-fa0f. The upstream .syncsnitch/work/w-20260928-033249-fa0f/orders-service is read-only. The rules in .bob/rules-syncsnitch-transformer/tolerant-reader.md still apply.
The Contract Verifier found these failing checks:
- V4 consumer vs upstream v2 (new contract): 4/5 passed (failures: test_payment_status_uses_real_amount)
Apply these fix instructions, editing only files inside .syncsnitch/work/w-20260928-033249-fa0f/billing-service:
- In billing-service code handling payment status and Money structures (adapter/mapper files under billing-service), update references to read amount_minor properly or populate it correctly with presence/real amount to satisfy upstream v2 schema requirements.
Facts the runner checked; they win over any instruction above (skip an instruction that edits tests/integration/ or changes a billing response):
- Failing contract test V4 tests.integration.test_contract::test_payment_status_uses_real_amount against upstream v2: ValueError: Field orders.Money.amount_minor does not have presence.
- Scanner hits in files the branch has not changed yet (each must still work with v1 and v2): billing/reports/revenue.sql:1 (total_price)
- The tests in tests/integration/ are the contract spec: fix the consumer code under .syncsnitch/work/w-20260928-033249-fa0f/billing-service, never the test, and never add or rename a field in a billing response. An actual value of 0 or empty usually means the consumer still reads a field the new upstream removed; read the new fields through the adapter where that value is built.
Current content of the files involved (paths under .syncsnitch/work/w-20260928-033249-fa0f/billing-service; the runner read them from syncsnitch/w-20260928-033249-fa0f):
--- billing/models/order.py
from datetime import datetime
from pydantic import BaseModel, Field
from typing import Optional, Union


class CustomerDTO(BaseModel):
    customer_id: str
    display_name: str


class MoneyDTO(BaseModel):
    amount_minor: int
    currency: str


class OrderDTO(BaseModel):
    order_id: str
    customer_name: Optional[str] = None
    total_price: Optional[float] = None
    customer: Optional[Union[CustomerDTO, dict]] = None
    total: Optional[Union[MoneyDTO, dict]] = None
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
        "currency": order.currency,
    }
--- billing/services/payments.py
from decimal import Decimal, ROUND_HALF_UP
from billing.adapters import from_summary

PAID_STATES = {"ORDER_STATUS_PAID", "ORDER_STATUS_SHIPPED", "PAID", "SHIPPED"}


def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def status_from_summary(summary) -> dict:
    view = from_summary(summary)

    return {
        "order_id": view.order_id,
        "paid": view.status in PAID_STATES,
        "amount_minor": view.amount_minor,
        "currency": view.currency,
    }
--- contracts/upstream/orders.proto
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
  option allow_alias = true;
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
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
  string order_id = 1;
  string customer_name = 2 [deprecated = true];
  double total_price = 3 [deprecated = true];  // major units (USD), e.g. 19.99
  OrderStatus status = 4;
  Customer customer = 5;
  Money total = 6;
  string shipping_eta = 7;  // RFC 3339 timestamp, empty when unknown
}
--- billing/adapters/__init__.py
from dataclasses import dataclass
from decimal import Decimal, ROUND_HALF_UP

def _round_half_up(d: Decimal) -> int:
    return int(d.quantize(Decimal("1"), rounding=ROUND_HALF_UP))

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

def from_rest(payload: dict) -> OrderView:
    order_id = payload.get("order_id")
    status = normalize_status(payload.get("status", ""))
    created_at = payload.get("created_at")
    
    customer = payload.get("customer")
    if isinstance(customer, dict) and "display_name" in customer:
        customer_name = customer["display_name"]
    else:
        customer_name = payload.get("customer_name", "")

    total = payload.get("total")
    if isinstance(total, dict) and "amount_minor" in total:
        amount_minor = int(total["amount_minor"])
        currency = total.get("currency", "USD")
    else:
        tp = payload.get("total_price", 0.0)
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
    field = type(summary).DESCRIPTOR.fields_by_name["status"]
    raw_status_name = field.enum_type.values_by_number[summary.status].name
    status = normalize_status(raw_status_name)

    order_id = getattr(summary, "order_id", "")
    
    # Check if total HasField or if total message is present / HasField("total")
    has_total = False
    try:
        has_total = summary.HasField("total")
    except Exception:
        has_total = getattr(summary, "total", None) is not None and summary.total.amount_minor != 0

    if has_total and getattr(summary, "total", None) is not None and summary.total.HasField("amount_minor") if hasattr(summary.total, "HasField") else True:
        # v2 total
        t = summary.total
        amount_minor = int(getattr(t, "amount_minor", 0))
        currency = getattr(t, "currency", "USD") or "USD"
    else:
        tp = getattr(summary, "total_price", 0.0)
        amount_minor = _round_half_up(Decimal(str(tp)) * 100)
        currency = "USD"

    has_customer = False
    try:
        has_customer = summary.HasField("customer")
    except Exception:
        has_customer = getattr(summary, "customer", None) is not None

    if has_customer and getattr(summary, "customer", None) is not None:
        c = summary.customer
        customer_name = getattr(c, "display_name", "")
    else:
        customer_name = getattr(summary, "customer_name", "")

    return OrderView(
        order_id=order_id,
        customer_name=customer_name,
        amount_minor=amount_minor,
        currency=currency,
        status=status,
    )
--- billing/contract_entrypoints.py
from .models.order import OrderDTO
from .services.invoice import build_invoice
from .adapters import from_rest


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
            version = _schema_version(conn)
            if version >= "0002":
                sql = _SQL_V2
                is_minor = True
            else:
                sql = _SQL_V1
                is_minor = False
            rows = conn.execute(text(sql)).fetchall()
    finally:
        engine.dispose()

    results = []
    for row in rows:
        rev = row.revenue
        if rev is None:
            rev_minor = 0
        elif is_minor:
            rev_minor = int(rev)
        else:
            rev_minor = _round_half_up(Decimal(str(rev)) * 100)

        results.append(
            {
                "day": str(row.day)[:10],
                "revenue_minor": rev_minor,
            }
        )
    return results
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
Then run cd .syncsnitch/work/w-20260928-033249-fa0f/billing-service && uv run pytest -q until green, and commit on syncsnitch/w-20260928-033249-fa0f: cd .syncsnitch/work/w-20260928-033249-fa0f/billing-service && git add -A . && git commit -q -m "fix(contract): address the Contract Verifier findings (SyncSnitch w-20260928-033249-fa0f)" -m "SyncSnitch-Agent: Gemini gemini-3.5-flash-lite (w-20260928-033249-fa0f)"
Do not push. Reply with one line.