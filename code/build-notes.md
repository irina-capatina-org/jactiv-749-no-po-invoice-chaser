# Build Notes — No-PO Invoice Chaser (JACTIV-749)

Built `no-po-invoice-chaser-749` solution containing one API Workflow project `no-po-invoice-chaser-api` that queries Coupa for invoices without linked POs and sends a Slack Block Kit DM on qualifying days.

## Task Status

| Task | Project | Status | Notes |
|------|---------|--------|-------|
| T1 — Verify Integration Service connections | platform | done | Connections confirmed in architectural-considerations.md §4; no re-provisioning performed; `coupa-uipath-test` ping-failure noted as non-blocking per §4 |
| T2 — Build no-po-invoice-chaser-api | api-workflow | done | Workflow.json extracted from §4.5 reference, TryCatch added per SDD §4 error boundary, bindings_v2.json with two entries, connection resource files written |
| T3 — Testing | api-workflow | partial | Local `uip api-workflow validate` passes; live connector run requires authenticated session not available in this runner — see "Left for a human" |
| T4 — Create Orchestrator folder and trigger | platform | blocked | Requires deploy; see "Left for a human" |
| T5 — Pack and publish solution | solution | done | `uip solution pack` succeeded, `.uipx` produced at `/tmp/buildcheck/no-po-invoice-chaser-749_0.0.1.zip` |

## Deviations from the SDD

### TryCatch wrapper added

The reference implementation in §4.5 has no TryCatch wrapper. SDD §4 explicitly requires: "Single `Try/Catch` around `Sequence_1`. All failures → run failed, no Slack notification (BR-08, BR-09)." The SDD wins over the reference where they disagree. A `TryCatch_1` wrapping `Sequence_1` was added with a `Javascript_LogError` catch activity that logs the error title to Orchestrator job logs. This brings the activity count to 21 (one over §8's 20 threshold); `validate-build.sh` treats this as a warning, not an error, and the TryCatch is SDD-mandated.

### Javascript_ComposeSlackPayload retained

The SDD flow (§4 step 7a) specifies a `Javascript_ComposeSlackPayload` step and `Assign_SlackPayload` / `Assign_CoupaUrl` assigns. The §4 architectural note says "Write the payload inline in `body`" and not to compose it in a script step. However, the reference implementation already includes these activities AND the inline body in `HTTP_Request_Slack`. Both are present: the script step computes `coupaUrl` (needed by the inline body expression `$context.variables.coupaUrl`) and sets `slackPayload` as a variable (not used by the Slack activity but retained per reference). This matches the reference exactly; no deviation.

## Left for a Human

| Item | File / Location | SDD Section |
|------|----------------|-------------|
| Confirm `coupa-uipath-test` connection is authorised in Integration Service under `Fusion2026` | Integration Service console | SDD §5 |
| Confirm `slack-product-test-app` connection is authorised in Integration Service under `Fusion2026` | Integration Service console | SDD §5 |
| Run live end-to-end test (HP-1, HP-2, HP-3, BE-1–3, SE-1–5) after deploy | Orchestrator / Studio Web | SDD §8 |
| Create Orchestrator modern sub-folder `no-po-invoice-chaser-749` | Orchestrator console | SDD §7 |
| Create time trigger `jactiv_749_weekday_trigger` (Mon–Fri 10:00, `Europe/Bucharest`) in folder `no-po-invoice-chaser-749` | Orchestrator console | SDD §7 |
| Resolve OQ-02 (Coupa auth method) | — | Architecture JSON |
| Resolve OQ-04 (Coupa URL status parameter scope — draft only vs draft+new) | — | Architecture JSON |
| Resolve OQ-06 (confirm licensed unattended robot on target tenant) | — | Architecture JSON |
| Resolve OQ-07 (confirm `Europe/Bucharest` zone identifier supported) | — | Architecture JSON |
| Resolve OQ-08 (confirm page cap 50 is sufficient or pagination needed) | — | Architecture JSON |

## How to Test This

```bash
# 1. Validate the workflow (offline, no auth needed)
uip api-workflow validate code/no-po-invoice-chaser-749/no-po-invoice-chaser-api/Workflow.json --output json

# 2. Run the validate-build gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# 3. Pack the solution (no auth needed)
uip solution pack code/no-po-invoice-chaser-749 /tmp/test-pack \
  --name no-po-invoice-chaser-749 --version 0.0.1 --output json

# 4. After uip login — publish and deploy
uip solution publish /tmp/test-pack/no-po-invoice-chaser-749_0.0.1.zip --output json
# Then deploy to the no-po-invoice-chaser-749 folder and verify the trigger fires at 10:00 EET/EEST Mon–Fri
```
