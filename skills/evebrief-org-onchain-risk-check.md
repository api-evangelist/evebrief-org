---
name: Check an EVM address for rug-pull risk (paid, A2A + x402 on XRPL)
description: Use the onchain-risk-oracle A2A agent at oracle.evebrief.org to get a none/low/high/critical risk verdict for an EVM address, paying 0.01 RLUSD per task through x402 on the XRP Ledger.
api: openapi/evebrief-org-openapi.json
operations: [x402_manifest__well_known_x402_get, healthz_healthz_get, jsonrpc__post]
agent_card: a2a/evebrief-org-agent-card.json
generated: '2026-09-19'
method: generated
---

# Check an EVM address for rug-pull risk

One provider, one skill, one price. Everything below is grounded in the provider's published contracts: the OpenAPI at `openapi/evebrief-org-openapi.json` (operationIds quoted verbatim), the agent card at `a2a/evebrief-org-agent-card.json`, and the x402 manifest at `well-known/evebrief-org-x402.json`.

## Before you call

1. **Confirm you can pay.** This agent settles in **RLUSD on the XRP Ledger mainnet (`xrpl:0`)**, not on an EVM chain. If your wallet cannot send RLUSD through `https://xrpl-facilitator-mainnet.t54.ai`, stop: there is no free path, no API key and no other payment rail (conventions/evebrief-org-conventions.yml, `payment`).
2. `GET /.well-known/x402` (`x402_manifest__well_known_x402_get`) — read `resources[0].accepts[0]`: `scheme exact`, `amount "0.01"`, `asset RLUSD`, `payTo`, `maxTimeoutSeconds 600`. This is the price you will be charged per task.
3. `GET /healthz` (`healthz_healthz_get`) — check `status == "ok"` and read `feed_records` / `feed_latest_ts` so you can tell the user how fresh the feed behind the verdict is.

## Build the request

The input contract lives in the card's `skills[0].inputSchema` (the OpenAPI's `POST /` declares no body):

- `address` (required) — must match `^0x[0-9a-fA-F]{40}$`. Validate locally first; a rejected call may still be a paid call.
- `chain` (optional) — one of `eth`, `base`, `bsc`, `arb`, `op`, `polygon`. Supply it when you know it; the card says it improves attribution.
- `deep` (optional, default false) — live evm-lab heuristics, described as more expensive and slower. No separate price is published, so do not promise the user a cost for it.
- `fork` (optional, default false) — only meaningful with `deep=true`; anvil-fork owner-privilege checks.

Input modes are `application/json` and `text/plain`; prefer JSON, e.g. `{"address": "0xb8c6ff50ea596671871784018a2030fe1852291a", "chain": "base"}` (one of the card's own examples).

## Call it

4. `POST /` (`jsonrpc__post`) as an A2A JSON-RPC 2.0 message carrying the JSON above as a DataPart.
5. Expect **HTTP 402**. Read the `PAYMENT-REQUIRED` header (base64 JSON) or the body: `accepts[0]` repeats the price and carries a per-challenge `invoiceId`. Settle exactly `0.01 RLUSD` to `payTo` through the facilitator **within 600 seconds**, then **retry the identical request** with the `PAYMENT-SIGNATURE` header.
6. Read the result against the card's `outputSchema`: `address`, `risk` (`none | low | high | critical`), `labels[]`, `reasons[]`, `evidence{}`, `sources[]` are required; `live_tier{}`, `feed{}` and `disclaimer` may be present. Surface `disclaimer` to the user verbatim when it is returned.

## Rules

- **Every accepted call is a payment.** There is no idempotency key and no documented way to re-fetch a paid result (conventions/evebrief-org-conventions.yml, `idempotency`). If a paid retry times out, tell the user a second charge may occur before retrying.
- **Nothing is reversible.** The check itself mutates nothing; the RLUSD transfer is an on-ledger settlement with no published refund path or window. Do not tell the user a payment can be undone.
- **Errors** (errors/evebrief-org-problem-types.yml): `402` = unpaid or expired challenge (re-read `invoiceId`, pay, retry); `404` `{"detail":"Not Found"}` = wrong path (the endpoint is the host root `/`, nothing else). No rate limits are documented; the 600-second challenge window is the only stated time limit.
- **Language.** The card and the verdict prose are German; the schema field names are English. Translate `reasons[]` and `labels[]` for the user when needed, but keep `risk` as the enum value.
- The card's `provider.url` points at the OpenClaw framework repository, not at a page about this operator; there is no support channel other than `hello@evebrief.org` on the apex site, which does not mention the oracle.
