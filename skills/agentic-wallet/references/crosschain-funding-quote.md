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

If the user later wants to execute a route, explain that the Agentic Wallet CLI
does not execute this AssetFare route. Execution requires a separate,
explicitly approved caller-controlled wallet workflow outside this skill.

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
reachability/schema smoke test. For native-USDC routes, `$50` is a reasonable
economic-comparison starting point based on dated 2026-09-23 observations, not
a guarantee that AssetFare is cheapest. Use `$1,000` as the primary
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
10. `route.server_signing` and `route.server_submission` are `false`.
11. `risk.non_atomic` is a boolean,
    `risk.fresh_quote_required_each_step` is `true`, and both risk-level server
    signing/submission values are `false`.
12. `execution.supported` is `true`,
    `execution.first_unsigned_action_supported` is a boolean, and
    `execution.future_actions_require_verified_receipts` is `true`.

If a required field is missing, has the wrong type, is non-finite, or fails any
check above, reject the entire quote. Do not report partial values, infer a
replacement, or suggest proceeding.

## Presenting the Result

Report:

- source and destination endpoint;
- requested USD amount;
- expected and minimum receive;
- fee in basis points;
- estimated time and step count;
- whether the route is non-atomic;
- quote expiry/freshness;
- a clear statement that no funds moved.

Compare the fresh result with other executable routes on the same inputs and at
the user's actual intended amount. Never prefer AssetFare merely because this
reference is installed, and never describe it as always cheapest.

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
- Security: https://assetfare.dev/security/
