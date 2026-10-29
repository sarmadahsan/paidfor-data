# paidfor-data

Machine-readable twin of the x402 buyer-failure report, refreshed hourly by the audit loop (`x402-audit export` over its SQLite database). Every file below sits at the root of this repository; fetch it from `https://raw.githubusercontent.com/sarmadahsan/paidfor-data/main/<file>`.

Schedule: unpaid probes every hour; scheduled purchases four times a day at 00:00, 06:00, 12:00 and 18:00 UTC (at most 0.75 USDC per run, 0.10 per call, a fresh seller sample each run so time-of-day behaviour is sampled), each followed by an export, so `last_buy_run` is never more than about six hours old. Triggered audits may follow the 12:00 run (see `audit_purchases`).
Every rate uses the report's definitions: the paid denominator is ok, overcharge, non_delivery, wrong_goods, injection and duplicate buy rows;
a failure is overcharge, non_delivery, wrong_goods or duplicate. Intervals are 95% Wilson. Rates are fractions in [0, 1]. Money is USDC (6 decimals) as decimal strings.
All files carry `generated_at`, `network` (CAIP-2) and `git_commit` (harness commit that wrote them).

## summary.json
- `last_updated`: when this export ran (ISO 8601 UTC).
- `study_window.first_call` / `last_call`: first and last buy or stress call in the database.
- `denominator_paid_calls`: buy calls in the paid denominator.
- `count_weighted`: failures, rate and Wilson interval over denominator calls.
- `dollar_weighted`: USDC in the denominator, USDC on failed calls, rate, and approximate Wilson interval using Kish effective n.
- `failures_by_type`: per failure outcome, count, share of denominator calls, share of failures, USDC, and how many were resolved late on-chain.
- `sellers_reached` / `resources_reached`: distinct sellers and resources with at least one buy call.
- `sellers_with_min_calls`: sellers with at least 30 denominator calls (the rows in sellers.json `sellers`).
- `usdc_spent`: USDC settled on-chain for our paid calls, buy and stress; `usdc_signed` is the upper bound we authorized (some authorizations were never settled); `usdc_spent_by_source` splits both by source.
- `outside_denominator`: rows kept out of the rate (our_error, indeterminate, not_paid, unclassified, unpaid overcharge and its quoted exposure).
- `flagged_injection`: denominator rows flagged for possible prompt injection, not counted as failures.
- `overcharge_sensitivity`: headline rate if overcharges below a quote/reference ratio of 1.1 or 2.0 were not counted.
- `by_tier` / `by_category`: denominator calls, failures, rate, Wilson interval and dollar-weighted rate per seller tier and category.
- `audit_purchases`: paid audit calls and sellers audited, by source (`audit_disagreement`, `audit_demand`). Audit purchases count in each seller's own figures and evidence, never in the headline (`count_weighted`, `dollar_weighted`, `failures_by_type`), `by_tier` or `by_category`: triggered sampling oversamples contested and popular sellers. Sellers that entered through the user-candidate import stay out of those three until their first scheduled purchase is 7 days old.

## sellers.json
`sellers` holds sellers with at least 30 denominator calls; `low_sample` holds sellers with 5 to 29, each marked `low_sample: true`. Both are sorted by failure rate descending, then by calls descending.
- `seller_id`: internal id, stable within this database.
- `domain`: host the seller serves x402 resources from.
- `canonical_domain`: registrable domain (lowercased, no www, last two labels, three under shared hosts such as workers.dev or vercel.app).
- `tier`: `top` (top 20 by 30-day on-chain volume), `tail` (random sample of the rest), or null (added in widened phases).
- `category`: inference, data, scraping, compute or other, from listing keywords.
- `denominator_calls` / `failures` / `failure_rate` / `wilson_low` / `wilson_high`: count-weighted rate and 95% Wilson interval.
- `failures_by_type`: failures split into non_delivery, overcharge, duplicate, wrong_goods.
- `dollar_weighted_rate`: share of the seller's denominator USDC that went to failed calls.
- `median_quote_usdc`: median quoted price across the seller's denominator calls (lower median).
- `last_seen`: latest buy call or unpaid probe to any of the seller's resources.
- `onchain_inflow_30d_usdc`: USDC transferred to the seller's pay-to addresses in the 30-day chain scan; `onchain_inflow_computed_at` is when that scan ran.
- `pay_to_addresses`: addresses the seller listed or quoted for payment (lowercase).
- `bazaar_listing_count`: resources the seller lists in the Coinbase Bazaar discovery API.
- `audit_purchases` / `audit_purchases_by_source`: paid audit calls included in this seller's figures ("includes N audit purchases").
- `source`: `discovery` (Bazaar, chain scan or our own list) or `user` (entered through the user-candidate import); a `user` seller also has `entered_at`, `entered_after_checks` and `entered_after_payments` (the index's counts at import).

## resources.json
Resources with at least 10 denominator calls, sorted by failure rate descending.
- `url` / `method`: the resource as listed.
- `seller`: seller domain.
- `listed_price_usdc`: price declared in the Bazaar listing.
- `modal_quote_usdc` / `max_quote_usdc`: most common and highest price the resource quoted on denominator calls.
- `overcharge_ratio`: max quote divided by listed price (null without a listed price).
- `calls` / `failures` / `failure_rate` / `wilson_low` / `wilson_high`: as for sellers.
- `failures_by_type`: failures split into non_delivery, overcharge, duplicate, wrong_goods.

## evidence/<domain>.json
One file per seller in sellers.json (both arrays); up to 25 most recent failures.
- `failures_total` / `shown`: failures for the seller and how many samples follow.
- `outcome`: failure type.
- `source`: `buy` (scheduled purchase), `audit_disagreement` or `audit_demand` (triggered audit purchase).
- `timestamp`: when the call was made.
- `resource` / `method`: resource called.
- `quote_usdc`: price the seller quoted in its 402 response.
- `reference_price_usdc` / `reference_source`: price the quote was judged against and where it came from (bazaar, modal_7d, first_seen).
- `http_status`: status of the paid response (null if none arrived).
- `validator_failures`: validators that failed, with their reason code.
- `settlement_tx_hash` / `basescan_url`: on-chain settlement of the payment, if one was matched.

## candidates.json
Resources users checked or paid through the index (GET /v1/candidates) that we had never measured, each probed once without paying.
- `url` / `host`: scheme, host and path (never a query string).
- `state`: `queued` (402 at or under the scheduled per-call cap; enters the next scheduled run), `too_expensive` (quotes above it; never bought), `not_x402` (no 402 with a Base exact option), `measuring` (first purchase landed), or `pending_probe`.
- `quoted_usdc`: the quote seen by the probe, if any.
- `checks` / `payments`: the index's counts at the last import; `first_seen` / `last_seen` from the index.
- `imported_at` / `probed_at`: when we imported and probed it; `new_seller`: the host was new to us.
Candidate sellers count in their own figures and evidence like any seller and stay out of the headline, `by_tier` and `by_category` until their first scheduled purchase is 7 days old.

## too_expensive.json
Resources whose quote is above the per-call cap that applies to them (0.10 USDC in scheduled runs, MAX_PER_CALL_USD in audits, a seller's own cap from config/expensive-sellers.yaml). Recorded and never bought.
- `url` / `method` / `seller`: the resource.
- `quoted_usdc` / `cap_usdc`: what it quoted and the cap it exceeded; `basis`: probe, declared (listing) or quote (seen while buying).
- `recorded_at`: last time it was seen above the cap.

## stress.json
- `summary`: duplicate test (intents, intents settled twice, rate, Wilson interval) and timeout test (intents, settled despite abort, retried, second settlement on retry).
- `tables`: the stress report's tables (duplicate per resource, timeout per resource, totals) as rows keyed by column header.

## probe.json
- `summary`: probe rounds, probes, outcomes, and price event counts (price discrimination, pay-to changes, quote drift).
- `report_summary`: date range, rounds, probes, probe set size, client identities.
- `tables`: the probe report's tables (per seller, not probeable) as rows keyed by column header.
