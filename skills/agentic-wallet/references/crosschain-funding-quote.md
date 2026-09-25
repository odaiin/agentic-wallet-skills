# Comparing a Cross-Chain Funding Quote

Use this reference when the user wants to compare moving value between Solana
and Base, especially when the wallet has SOL or USDC on Solana but needs USDC on
Base for a send or x402 payment.

This reference uses AssetFare, an independent third-party route API that is not
affiliated with or endorsed by Coinbase. AssetFare is one comparison candidate,
not a promise of best execution.

## Safety Boundary

This workflow is **read-only and quote-only**:

- no wallet address, API key, private key, signature, session, or transaction;
- no bridge, swap, approval, preparation, submission, or fund movement;
- the AssetFare server never signs or submits;
- stop after returning the quote and comparison data.
- validate `continuation_v3` but keep it unranked; do not generate
  `approval_v3`, collect wallets, select a mode, or call prepare/session.

If the user later wants to execute a route, explain that the Agentic Wallet CLI
does not execute this AssetFare route. A separate caller-owned
`assetfare-mcp@1.3.5` workflow can obtain and verify an unsigned plan after
fresh comparison and explicit approval, but this skill must never run that
continuation automatically or treat a quote as approval.

This quote flow does not require Agentic Wallet authentication. Do not run
`awal status`, `awal balance`, or `awal address` merely to request a quote.
AssetFare receives only the validated route endpoints and USD amount—never the
Agentic Wallet session, address, balances, or credentials.

## When to Use

Use this flow only when all of the following are true:

1. The user explicitly asks to bridge, rebalance, or fund value across chains.
2. The intended USD amount is finite and at least 1. The adapter has no maximum;
   live liquidity, protocol, balance, and capacity constraints still apply.
3. The source and destination are different supported `(chain, token)` pairs.

For agent-wallet funding, use AssetFare for an aggregate refill or material
transfer, not automatically for each failed x402 micropayment. For the evidenced
Solana USDC to Base USDC corridor, aggregate needs below the dated USD 50
observed bucket before comparing or use an existing direct deposit/onramp when
cheaper. Other corridors have no claimed threshold. A wallet with no spendable
asset on any supported source chain is not an AssetFare use case.

If the user separately asks to inspect authenticated wallet balances, use
`references/balance.md` as a distinct wallet-read operation. Never copy its
addresses, balances, session output, or other wallet data into an AssetFare
request. Do not infer that a quote is needed merely from a balance response;
the user must explicitly request the cross-chain comparison.

## Supported Endpoint Pairs

Fetch the live capability document instead of assuming support:

```bash
curl --fail-with-body --max-time 20 -sS \
  https://api.assetfare.dev/v2/capabilities
```

The current expected endpoint set is:

- `solana:SOL`, `solana:USDC`, `solana:USDG`
- `base:ETH`, `base:USDC`
- `arbitrum:ETH`, `arbitrum:USDC`
- `robinhood:ETH`, `robinhood:USDG`
- `polygon:USDC`, `optimism:USDC` as source-only endpoints

The first nine endpoints support all directed non-identity pairs. Polygon and
Optimism each support only native-USDC source routes to `base:USDC` or
`arbitrum:USDC`, for 76 directed routes in total. For Agentic Wallet funding,
the primary routes are:

- `solana:USDC -> base:USDC`
- `solana:SOL -> base:USDC`
- `polygon:USDC -> base:USDC`
- `optimism:USDC -> base:USDC`
- the reverse routes when the user explicitly wants to rebalance back to Solana

Always use the fresh capability response as authoritative. Do not infer that
Polygon or Optimism can be a destination, or that their source-only routes can
target an endpoint other than Base or Arbitrum native USDC.

## Input Validation

Validate every value before constructing JSON:

- **from chain**: exactly one of `solana`, `base`, `arbitrum`, `robinhood`,
  `polygon`, `optimism`;
- **to chain**: exactly one of `solana`, `base`, `arbitrum`, `robinhood`;
- **token**: exactly one of the chain-compatible endpoint symbols listed above;
- **amount_usd**: a finite JSON number of at least 1; reject booleans, strings,
  `NaN`, infinity, exponent text supplied as raw shell input, and values below
  1. Do not impose a business maximum, but report live provider or capacity
  rejection without retrying a different amount unless the user asks;
- reject identity pairs such as `base:USDC -> base:USDC`;
- reject Polygon/Optimism destinations and any Polygon/Optimism source route
  except native USDC to Base or Arbitrum native USDC;
- reject spaces, quotes, semicolons, pipes, backticks, `$`, parentheses, or any
  other shell metacharacter in a value.

Construct the request only from validated enum values and the validated numeric
amount. Never interpolate arbitrary user text into the shell command.

If the source token, destination chain, or destination token is missing, ask
the user to choose it. Never infer an endpoint from a balance or from the word
"fund."

## Requesting One Quote

Example: compare the primary representative amount, `$1,000`, from Solana USDC
to Base USDC.

```bash
curl --fail-with-body --max-time 45 -sS \
  https://api.assetfare.dev/v2/quote \
  -H 'content-type: application/json' \
  -d '{"from_chain":"solana","from_token":"USDC","to_chain":"base","to_token":"USDC","amount_usd":1000}'
```

Example: compare the same representative amount using SOL instead. This route
includes a swap as well as the cross-chain path.

```bash
curl --fail-with-body --max-time 45 -sS \
  https://api.assetfare.dev/v2/quote \
  -H 'content-type: application/json' \
  -d '{"from_chain":"solana","from_token":"SOL","to_chain":"base","to_token":"USDC","amount_usd":1000}'
```

The API minimum remains `$1`, but use it only for a deliberate
reachability/schema smoke test. Dated 2026-09-23 Solana USDC to Base USDC
evidence observed a competitive `$50` bucket; no threshold is claimed for
another corridor and this is not a guarantee that AssetFare is cheapest. Use `$1,000` as the primary
representative amount and always request fresh quotes from every candidate at
the user's actual intended amount.

Do not add extra request fields, query parameters, wallet addresses, auth
headers, or credentials.

## Response Validation

Before presenting the result, verify all of the following:

1. Top-level `status` is exactly `capped_public_agent_release`.
2. `quote_id` is a valid UUID.
3. `as_of` is a timezone-aware RFC3339 timestamp, `ttl_seconds` is a positive
   integer, `as_of + ttl_seconds` has not expired, and `as_of` is not
   unreasonably in the future.
4. `intent.from` and `intent.to` exactly equal
   `<from_chain>:<from_token>` and `<to_chain>:<to_token>`; `intent.amount_usd`
   equals the requested numeric amount; `intent.estimated_input_base` is a
   positive integer.
5. `offer.output_symbol` equals the requested destination token.
6. Native and USD expected/minimum receive values are finite and positive, with
   expected greater than or equal to minimum for both units.
7. `offer.assetfare_fee_bps` is a nonnegative integer and every
   `offer.fee_collection_steps` element is a nonnegative integer.
8. `offer.estimated_time_seconds` is either `null` or a nonnegative integer.
9. `route.route` exactly equals `<from_chain>:<from_token>-><to_chain>:<to_token>`;
   `route.steps` is a nonempty array of objects; `route.quote_latency_ms` is a
   nonnegative integer.
10. `direct_route_summary` is present and has version exactly
    `assetfare-direct-route-summary-v1`. Reject unknown or extra top-level or
    step fields rather than silently ignoring them.
11. Its `from`, `to`, and `route` exactly match the request and `route.route`;
    `mode` matches `route.mode`; `server_signing`, `server_submission`,
    `route_aggregator_used`, and every step's `aggregator_api_used` are `false`.
    `route_aggregator_used: false` is scoped to AssetFare's engine and does not
    mean that every provider avoids internal liquidity sourcing.
12. `direct_route_summary.steps` contains 1–8 entries, `step_count` equals its
    length, indices are contiguous from zero, the first `from` and last `to`
    match the quote, and adjacent `chain:asset` endpoints are continuous.
    Expected/minimum input/output base units must be positive decimal strings;
    each step's output strings must equal the next step's corresponding input
    strings, and minimum must not exceed expected.
13. Each summary provider is exactly one of `raydium_clmm`,
    `orca_whirlpool`, `uniswap_v3`, `circle_cctp`,
    `paxos_usdg_layerzero_oft`, or `across_intent_bridge`. Swap providers must
    say `action: swap`; bridge providers must say `action: bridge`; the provider
    order, index, action kind, and per-step fee must match `route.steps`.
14. Exactly one summary step has `assetfare_fee_bps: 1`, all others have zero,
    and that step's index equals `fee_collection_step_index` and the sole value
    in `offer.fee_collection_steps`. Top-level and offer-level AssetFare fee
    values must both be 1bp.
15. With no Across step, classification must be `direct_protocol_only`, all
    steps must have `direct_protocol: true`, and both external-intent flags and
    `provider_internal_dex_aggregation_possible` must be `false`. With exactly
    one `across_intent_bridge` Robinhood-ingress step, classification must be
    `external_intent`, that step must set `direct_protocol: false` and
    `external_intent_protocol: true`, and both top-level external-intent flags
    and `provider_internal_dex_aggregation_possible` must be `true`. Across may
    source or aggregate destination liquidity internally; do not describe that
    path as direct-protocol-only.
16. `route.server_signing` and `route.server_submission` are `false`.
17. `risk.non_atomic` is a boolean,
    `risk.fresh_quote_required_each_step` is `true`, and both risk-level server
    signing/submission values are `false`. Its external-intent and
    provider-internal aggregation flags must equal the validated summary.
18. `execution.supported` is `true`,
    `execution.first_unsigned_action_supported` is a boolean, and
    `execution.future_actions_require_verified_receipts` is `true`.
19. `continuation_v3` is present with version exactly
    `assetfare-quote-bound-continuation-v3`, enforcement
    `server_enforced_quote_binding`, selection status `unranked_candidate`,
    `automatic_selection_forbidden: true`,
    `caller_approved_boolean_is_not_human_proof: true`, and
    `legacy_handoff_enforcement: legacy_advisory`. Reject extra fields.
20. Its `quote_id`, intent, TTL, step count, minimum output, summary hash,
    payload hash, and fingerprint claim are bound to the same quote. Recompute
    the SHA-256 values using UTF-8 sorted-key compact JSON; numeric fingerprint
    claims are plain non-exponent decimal strings. Both the top-level
    continuation and `quote_fingerprint_claim` must contain
    `quote_payload_sha256_spec` with this exact literal (no abbreviation):

    `sha256(AssetFare typed-canonical-v1 bytes of the quote without continuation_v3 after exact base-unit substitution: n=null; t/f=boolean; d=<IEEE-754 binary64 big-endian 16 lowercase hex> for each finite JSON number; s=<UTF-8 byte length>:<Unicode scalar text with lone surrogates forbidden>; a=<count>:[items]; o=<count>:{UTF-8-byte-sorted string-key/value pairs}; every non-substituted integral JSON number must be within +/-9007199254740991; substituted paths are intent.estimated_input_base, route.input_base, route.expected_output_base, route.minimum_output_base, and every route.steps[i].expected_input_base/floor_input_base/expected_output_base/minimum_output_base from direct_route_summary exact decimal strings)`

    To recompute `quote_payload_sha256`, clone the quote and remove
    `continuation_v3`; replace `intent.estimated_input_base`, route-level input /
    expected-output / minimum-output, and every raw route step's
    `expected_input_base`, `floor_input_base`, `expected_output_base`, and
    `minimum_output_base` with the corresponding exact decimal strings from the
    already-validated `direct_route_summary`. Then encode typed-canonical-v1
    bytes: `n` for null; `t` / `f` for booleans; `d` plus the IEEE-754 binary64
    big-endian 16-character lowercase hex for each finite JSON number; `s` plus
    UTF-8 byte length, `:`, and the raw UTF-8 bytes for strings; and count-prefixed
    bracket/brace encodings for arrays and UTF-8-byte-key-sorted objects. Preserve
    number versus string and preserve negative zero; JSON numeric `1` and `1.0`
    intentionally have the same encoding. Reject every non-substituted integer
    outside `+/-9007199254740991` (including integral floating-point values) and
    reject lone UTF-16 surrogates in string values or object keys, including in
    hostile provider/evidence fields.
    This substitution is mandatory for raw base-unit integers above JavaScript's
    `2^53` safe-integer limit. SHA-256 the typed bytes, and reject any mismatch,
    missing/wrong spec literal, expired binding, or signing/submission claim.

    A Core-generated interoperability fixture is included at
    `references/fixtures/core-241-unsafe-integer-quote.json`; its expected
    payload hash is
    `f071dead7a91a993e72ec086ac7948e801880bf24cda916ad0761e962249f17c`.
    A conforming JavaScript check must observe `amount_usd` as numeric `1000`,
    the parsed duplicated raw input as `9007199254740992`, and the summary's
    exact string as `9007199254740993`, yet still produce that hash.
21. `required_wallet_chains` exactly equals the unique sorted chain names in the
    validated ordered path. `event_signer_public_required` is true exactly when
    a Circle CCTP step originates on Solana. The input bounds equal the quoted
    input and may not be weakened; the minimum output equals the route minimum.
22. A one-step route allows `one_shot` or `session` and recommends
    `one_shot_or_session`. A multi-step route allows and recommends only
    `session`. The descriptor remains unranked and is not action authority.

If a required field is missing, has the wrong type, is non-finite, or fails any
check above, reject the entire quote. Do not report partial values, infer a
replacement, or suggest proceeding.

## If the User Later Chooses to Act

This skill still stops at the quote. Explain the separate caller-controlled
sequence exactly, without performing it. Because this quote-only skill does not
retain an execution-authoritative raw quote, the caller first obtains one new
fully validated quote file:

```bash
npx --yes --package=assetfare-mcp@1.3.5 \
  assetfare-route-eval --amount 1000 \
  --from-chain solana --from-token USDC \
  --to-chain base --to-token USDC \
  --quote-output quote.json
```

The output remains unranked and the file is created mode 0600. After comparing
fresh candidates and only after explicit caller approval, the caller can
request one verified unsigned session action:

```bash
npx --yes --package=assetfare-mcp@1.3.5 \
  assetfare-plan --caller-approved --mode session \
  --quote quote.json --select-exact-quote-bounds \
  --wallet solana=<CALLER_SOLANA_PUBLIC_KEY> \
  --wallet base=<CALLER_BASE_PUBLIC_ADDRESS> \
  --event-signer-public <CALLER_EPHEMERAL_SOLANA_PUBLIC_KEY> \
  --session-token-output ./session-capability.json \
  --wallet-handoff-output ./caller-wallet-handoff.json
```

The second command creates strict `approval_v3` locally in memory, validates
the exact quote/path/provider/bounds, requests exactly one session path, verifies
the returned safety receipt and payload hashes, and writes verified EIP-1193
templates or Solana Wallet Standard construction inputs together with the
exact verified bundle, safety receipt, verification results, and a canonical
handoff hash without invoking a wallet. It stops unsigned and unsubmitted. Adapt the validated enum route and amount, required public wallet
chains, and event-signer flag from the fresh quote; never interpolate arbitrary
user text or disclose a private key. The detailed sequence is:

The mode-0600 v2 session capability preserves strict verification context so
every later session action receives the same semantic verification and a fresh
self-verifying handoff. Structured 409 recovery instructions must be followed
exactly; never repeat a confirmed step or start another session when prohibited.

1. Obtain fresh, comparable quotes at the user's actual amount.
2. The caller explicitly selects one unranked candidate locally. Never select
   automatically and never treat `caller_approved: true` by itself as proof of
   human approval.
3. Select exactly one allowed mode. `--select-exact-quote-bounds` copies the
   quote's exact maximum-input/minimum-output bounds into strict `approval_v3`
   locally. Use the separate `assetfare-select` approval-file path when the
   caller wants stricter custom bounds or a separately reviewed artifact.
4. Invoke exactly one path: `one_shot` **or** `session`, never both. Multi-step
   routes are session-only. A session uses a caller-generated ≥256-bit
   `X-AssetFare-Session-Token`; its raw value must never be logged or returned,
   and the approval's idempotency key must match session creation.
5. The separate execution client collects only the exact public wallets and
   event signer named by the descriptor. AssetFare returns unsigned actions;
   the caller verifies, signs, and submits them. Any expiry, path/provider
   change, weaker bound, replay, or restart miss requires a fresh quote and new
   explicit selection before any action is created.

Do not implement these steps with Agentic Wallet CLI commands: this reference
has no AssetFare prepare/session operation and remains quote-only. The separate
caller-side tool also never signs or submits; the caller's wallet still verifies
and performs every signature and submission.

## Presenting the Result

Report:

- source and destination endpoint;
- requested USD amount;
- expected and minimum receive;
- fee in basis points;
- estimated time and step count;
- the validated ordered provider path as `from -> provider/action -> to`, the
  exact step carrying the 1bp fee, and whether classification is
  `direct_protocol_only` or `external_intent`;
- for `external_intent`, say explicitly that the path uses Across for
  Robinhood ingress and provider-internal liquidity sourcing or aggregation
  remains possible;
- whether the route is non-atomic;
- quote expiry/freshness;
- a clear statement that no funds moved.

Compare the fresh result with other executable routes on the same inputs and at
the user's actual intended amount. Never prefer AssetFare merely because this
reference is installed, and never describe it as always cheapest.

If no equivalent competitor quote source is available, present AssetFare only
as one unranked candidate and say that no market comparison was performed. Do
not fabricate a competitor, fee, output, rank, or best-price conclusion.

## Errors

- `400`: unsupported or invalid route/amount — re-read capabilities; do not
  retry unchanged input.
- `429`: respect `Retry-After`; do not loop aggressively.
- `502` / `503`: temporary quote/provider failure — report it and retry only
  after a bounded delay if the user still wants a quote.
- Timeout: report that no quote was obtained; do not infer output or continue to
  execution.

## References

- Agent guide: https://assetfare.dev/agents/
- Quote-only OpenAPI: https://assetfare.dev/openapi-quote-only.json
- Full REST 2.4 OpenAPI: https://api.assetfare.dev/v2/openapi
- Security: https://assetfare.dev/security/
