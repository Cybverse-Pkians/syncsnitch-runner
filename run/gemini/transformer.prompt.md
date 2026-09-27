SyncSnitch run w-20260927-063615-d1f5, step S4 fix round. You are Subagent 2, the Downstream Code Transformer, running headless from the SyncSnitch website: never ask questions, keep replies short.
Paths are relative to the workspace root. The consumer .syncsnitch/work/w-20260927-063615-d1f5/billing-service is inside the git clone .syncsnitch/work/w-20260927-063615-d1f5, on branch syncsnitch/w-20260927-063615-d1f5. The upstream .syncsnitch/work/w-20260927-063615-d1f5/orders-service is read-only. The rules in .bob/rules-syncsnitch-transformer/tolerant-reader.md still apply.
The Contract Verifier found these failing checks:
- V4 consumer vs upstream v2 (new contract): 0/5 passed (failures: test_invoice_paid_order, test_unpaid_order_is_not_invoiced, test_payment_status_uses_real_amount, test_revenue_report, test_rest_contract_examples_parse)
- V5 Prism contract examples: test_rest_contract_examples_parse not passed in both runs
Apply exactly these fix instructions, editing only files inside .syncsnitch/work/w-20260927-063615-d1f5/billing-service:
- Update `billing-service/src/main/java/com/service/billing/InvoiceProcessor.java` to handle the new fields in the upstream v2 payment event schema.
- Modify `billing-service/src/main/java/com/service/billing/RevenueReportService.java` to align with the changes in the revenue data structure required by the new contract.
- Regenerate Prism contract examples in `billing-service/contracts/` to ensure they conform to the actual schema expected by the upstream v2 implementation.
Then run cd .syncsnitch/work/w-20260927-063615-d1f5/billing-service && uv run pytest -q until green, and commit on syncsnitch/w-20260927-063615-d1f5: cd .syncsnitch/work/w-20260927-063615-d1f5/billing-service && git add -A . && git commit -q -m "fix(contract): address the Contract Verifier findings (SyncSnitch w-20260927-063615-d1f5)" -m "SyncSnitch-Agent: Gemini gemini-3.1-flash-lite (w-20260927-063615-d1f5)"
Do not push. Reply with one line.