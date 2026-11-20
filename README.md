# OKX API trading: which client fits your stack, how do you get from a demo key to a live order, and what breaks in production? (REST vs WebSocket, rate limits, endpoint map)

OKX runs its trading API on v5, and that version puts spot, margin, futures, perpetual swaps and options behind one REST and WebSocket layer with one set of credentials. There is no API subscription to buy. Access comes with your account, and your real cost is the trading fee on every order you send. The shortest path to a working first call is a demo key plus either `python-okx` or CCXT, depending on whether OKX is your only venue.

> **Quick pick.** Python developer trading only on OKX: `python-okx`, demo mode through `flag="1"`. Multi-exchange or multi-language stack: CCXT, `set_sandbox_mode(True)`, 110 ms default rate limiter. Go service that wants typed streams: the community Go SDK. Whichever you choose, create the key with Read + Trade permissions, add an IP whitelist, and leave Withdraw off.
>
> 👉 [Open an OKX account and create your first API key](https://okx.com/join/CASH20)

## Does OKX sell API plans or subscriptions?

No. OKX has no pricing page for API tiers, no per-request billing and no package to upgrade into. The API ships with the account, and the only money that changes hands is trading fees, which depend on your fee tier rather than on how you connect. So a plan comparison table for "OKX API packages" would be fiction. What actually differs is the client you build against.

| Route | Languages and coverage | Rate limiting and errors | Demo switch | Best for | Start |
| --- | --- | --- | --- | --- | --- |
| Native REST + WebSocket | Any language with an HTTP client; every v5 endpoint reachable | You write the throttling and the backoff; OKX returns a `code` field on every response | Header `x-simulated-trading: 1` | Learning the API, or a thin internal client you fully control | [Get an account to test against](https://okx.com/join/CASH20) |
| `python-okx` | Python 3.7+; one class per domain (`Account`, `Trade`, `MarketData`, `Funding`, `SubAccount`, `BlockTrading`, `CopyTrading` and more) | HTTP status plus OKX `code`/`sCode`; no documented built-in throttler | `flag="1"` on each client you construct | OKX-only Python work where `instId`, `tdMode`, `ordType` should read exactly as in the docs | [Create the key you'll pass to it](https://okx.com/join/CASH20) |
| CCXT | TypeScript, JavaScript, Python, PHP, C#/.NET, Go, Java, Rust; 104 exchanges, all 446 raw OKX endpoints as implicit methods | Token-bucket limiter on by default (`rateLimit` 110 ms for OKX), 41 typed exceptions in one hierarchy | `set_sandbox_mode(True)` | Multi-venue services, or a team that may switch language later | [Sign up to fund the live side](https://okx.com/join/CASH20) |
| Community Go SDK | Go; typed WebSocket channels, local order book with gap detection, 21 typed channel methods | Sentinel errors matched with `errors.Is`, per-endpoint and per-instrument limits, retries for rate-limited and expired-timestamp requests only | `WithDemoTrading()` | Go services that want compile-time-safe streams | [Open the account it connects to](https://okx.com/join/CASH20) |

## What does the v5 endpoint structure actually look like?

Every call follows `/api/v5/{category}/{action}`, so once you learn one category the rest read the same way. The categories you will touch most:

- `/api/v5/account/` for balances, positions, leverage, fee rate and risk state
- `/api/v5/trade/` for placing, amending and cancelling orders, plus algo orders
- `/api/v5/market/` for tickers, candles, order books and recent trades
- `/api/v5/public/` for instruments, funding rates, mark price and price limits
- `/api/v5/asset/` for deposits, withdrawals and transfers
- `/api/v5/copytrading/`, `/api/v5/users/subaccount/`, `/api/v5/rfq/` for copy trading, sub-accounts and block RFQ

Placing an order is one endpoint, `POST /api/v5/trade/order`, with the product type declared in the request body rather than in the URL. That is the design change v5 introduced: v3 modelled each product line separately and gave each one its own account, which meant moving funds between product accounts, capped at one transfer per second. v5 runs on the unified account model, so the same balance backs every instrument. OKX skipped v4 entirely, citing the number's association with bad luck in parts of East Asia, and v3 was retired shortly after v5 rolled out to all unified account users.

## REST or WebSocket?

Use REST when you want one answer to one question. Use WebSocket when you want the answer to keep arriving without asking again.

| Task | Better fit |
| --- | --- |
| Account balance, order history | REST |
| Placing or cancelling orders where latency is not critical | REST |
| Historical candles | REST |
| Live order book and price feeds | WebSocket |
| Live positions and P&L | WebSocket |
| High-frequency order management | WebSocket |

Public channels (tickers, candles, trades, order book) need no authentication, which makes them useful for sanity-checking connectivity before you touch keys. Private channels (account, positions, orders, algo orders) require an API key. In v5 the public and private streams sit on different URLs, so a single connection cannot carry both. If your client library does not enable `permessage-deflate`, decompression fails: v5 dropped v3's manual compression handling in favour of the standard WebSocket extension.

Order entry over WebSocket exists in v5 and comes back faster than the REST round trip. For anything with a tight loop, that matters more than the SDK convenience.

## How do you create a key that is safe to run unattended?

Six steps, and two of them are the ones people skip.

1. Go to Profile → API Management → Create API Key, and label it something you will recognise in a log line, like `trading-bot-prod`.
2. Grant Read and Trade. Leave Withdraw off unless your strategy genuinely moves funds out.
3. Add an IP whitelist. For anything in production this is not optional; an unrestricted key is a key that works from anywhere.
4. Set a passphrase and store it. It cannot be recovered later.
5. Copy the secret key immediately. It is shown once.
6. Test a public endpoint first, with no auth at all: `https://www.okx.com/api/v5/public/instruments?instType=SPOT`.

The single most common onboarding failure is not a signing bug. It is `OKX API key doesn't exist`, which almost always means the key was created in the demo environment and is being used against live, or the reverse. OKX keeps completely separate key sets for the two environments, so a working demo key proves nothing about production.

Demo trading has three common switches depending on your stack:

- Raw HTTP: add the header `x-simulated-trading: 1`
- CCXT: `exchange.set_sandbox_mode(True)`
- `python-okx`: pass `flag="1"` to each client you construct (the Go SDK uses `WithDemoTrading()`)

Regional routing is a separate axis. CCXT treats `okx`, `myokx` (EEA) and `okxus` (US) as three distinct exchange ids, and the Go SDK exposes `WithRegion(RegionEEA | RegionUS)`. If your account is under a regional entity, the wrong host produces authentication errors that look like key problems.

👉 [Create a demo key first, then a live key when the strategy holds](https://okx.com/join/CASH20)

## What does a first order need besides a key?

A client order ID, and a subscription.

`clOrdId` is your own identifier for the order. It must be case-sensitive alphanumeric, start with a letter, and stay within 32 characters. Use it. When a request times out or a response is ambiguous, `clOrdId` is how you find out whether the order exists without guessing, and it is the basis for idempotent retries.

Before you send anything, subscribe to the private `orders` channel and watch the state move from `live` to `filled`. A REST response only tells you the order was accepted. The fill, the average price and the fee arrive on the stream.

A minimal swap order body looks like this: `instId` `BTC-USDT-SWAP`, `tdMode` `cross`, `side` `buy`, `ordType` `limit`, `px` your price, `sz` your size, `posSide` `net` in net position mode. Amending supports new price and new size across all product types, and `cxlOnFail` cancels the order if the amend is rejected, which is safer than leaving a stale resting order in the book.

Two limits are worth designing around. Batch endpoints accept up to 20 orders per request and can mix product types, but a batch is not all-or-nothing: check `sCode` and `sMsg` on every returned item, or you will believe you placed 20 orders when you placed 14. And `sz` does not mean the same thing everywhere. On a swap it is contract count; on spot it is base-currency amount. Sending a spot quantity to a swap endpoint is a fast way to open a position a hundred times the size you intended.

## Which errors will you actually hit?

OKX returns a `code` field on every response, and `"0"` means success. Build your error handling around that field rather than around HTTP status alone.

| Code | Meaning | What to do |
| --- | --- | --- |
| 50001 | Key invalid or not found | Check whether the key is live or demo before anything else |
| 50102 | Timestamp expired | Sync the host clock with NTP; drift of a few seconds breaks signing |
| 50013 | IP not whitelisted | Add the server's egress IP to the key |
| 51008 | Insufficient balance | Check available, not total, balance |
| 51010 | Order size too small | Read the instrument's minimum size and lot size |
| 51603 | Order not found | Confirm the order ID and product type |

Rate limiting is the muddier one. Third-party guides cite 50011 in one place and 50026 in another for the same condition, and the Go SDK groups 50011, 50061 and HTTP 429 under one sentinel error. Treat the exact code list as something to confirm against OKX's current documentation, and implement exponential backoff regardless.

The structural facts matter more than the numbers. Public unauthenticated endpoints are metered per IP; private endpoints are metered per User ID, with sub-accounts having their own. Trading endpoints share their budget between REST and WebSocket, so moving order flow to the socket does not buy you extra headroom. Clients that handle this for you: CCXT ships a token bucket at 110 ms per request, and the Go SDK runs one bucket per endpoint with per-instrument sublimits, waiting for a token instead of returning an error.

Precision is the other silent rejection source. OKX enforces lot size, tick size and minimum order size per instrument, and contract face value varies by instrument. CCXT reads instrument metadata and gives you `amount_to_precision`, `price_to_precision` and `cost_to_precision` on top of string-based decimal maths, which is what keeps a 0.1 + 0.2 style float artefact from turning into a rejected order at 3 a.m. The Go SDK keeps prices and sizes as `okx.Number`, the exact decimal string OKX sent.

## What changed in the interface during 2026?

A community Go SDK's documentation, current to October 2026, records four changes worth checking against your existing code. Treat these as that maintainer's record and confirm against OKX's official docs before you deploy:

- A new REST domain, `openapi.okx.com`
- WebSocket moving to port 443, with port 8443 retired on 31 October 2026
- Order books identified by `seqId`, replacing the deprecated checksum method
- A new `rpi` order type

If your client hardcodes `www.okx.com` and 8443, both entries are worth an audit. The same documentation also mentions EEA and US regional domains.

## What does API trading actually cost?

Nothing to connect, and normal trading fees to trade. There is no subscription, no per-call charge and no API upsell. Two fields in the order response tell you what a fill cost: `fee` is negative when the platform charged you and positive when you received a rebate, and `rebate` carries the rebate amount separately.

Fee rates themselves depend on your account tier and are published on OKX's fee page, with maker and taker rates differing by product. Check the rate for the instrument you are actually trading before you size a strategy, because a market-making bot and a taker-heavy momentum bot on the same account do not face the same economics.

## Which client should you pick?

Three rules cover most cases.

**OKX is your only venue, you write Python, and you debug against the documentation.** Take `python-okx`. Your code says `instId`, `tdMode`, `ordType`, `px`, `sz`, `posSide`, the same words on the doc page you are reading, and the demo flag is explicit at each call site so you always know which environment a client is pointed at. It has 858 GitHub stars and about 94k PyPI installs a month, and it is maintained as a near-literal mirror of the v5 reference, so new endpoints tend to appear as methods within the week they ship.

**You may add another exchange, or another language, or you want one dependency across spot, margin, futures, swaps and options.** Take CCXT. It covers 104 exchanges from one API in eight languages, exposes all 446 OKX endpoints as implicit methods, provides 19 `watch*` methods plus WebSocket order entry, and maps OKX's error codes onto 41 typed exceptions so `except ccxt.InsufficientFunds` keeps working when you add a second venue. The install base is the largest of any option here: 43.8k stars and roughly 4.8M PyPI plus 494k npm installs a month. Both libraries are MIT licensed.

**You are writing Go and want typed channels.** The community SDK gives you `<-chan Ticker` instead of raw frames, automatic reconnect with re-login and re-subscription, a local order book that detects sequence gaps and resyncs, and retry logic that deliberately never retries an order after a network error so a retry cannot duplicate it.

👉 [Compare the options against a live account, not a demo one](https://okx.com/join/CASH20)

## Before you send the first real order

Pick by elimination: single venue and Python only, use `python-okx`; more than one exchange or more than one language on the team, use CCXT; Go with heavy streaming, use the Go SDK; learning the protocol, start with raw HTTP against public endpoints and add signing once you can read a response.

Then confirm three things that account for most failed first deployments. Demo and live keys are not interchangeable, so a key that works in one environment will fail in the other with a "does not exist" error. The IP whitelist must include the actual egress IP of the machine, including any NAT or egress gateway in front of it. And the instrument's lot size, tick size and minimum order size must be respected, with `sz` interpreted as contracts on derivatives and base currency on spot.

Everything else, including the rate-limit codes you will eventually see, you can calibrate from the responses once the first order lands.

👉 [Start with a demo key on OKX and keep the live key for when the strategy earns it](https://okx.com/join/CASH20)
