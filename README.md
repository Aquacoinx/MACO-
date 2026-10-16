# $MACO Volume Arena

A mobile-friendly wallet-registration site and server-side indexer for the $MACO trading-volume contest on Pons / Robinhood Chain (chain ID 4663).

**Token:** `0x6ffb075F8030C6eA7914c1BA90A8CF87c9A8686d`  
**What is counted:** qualifying buys + sells made by registered wallets during the configured UTC contest window.  
**Supported Pons launch generations:** Pons V2 bonding-curve events and its graduated Uniswap v4 pool; legacy Pons V1 Uniswap v3 pool events.  
**No keys or seed phrases are requested from participants.**

## Files

- `index.html` — public website, wallet registration and leaderboard UI.
- `schema.sql` — Supabase database, access policies and contest-window leaderboard view.
- `indexer/index.mjs` — server-side chain indexer. It uses verified on-chain logs and writes volume with a server-only Supabase key.
- `supabase/functions/register-wallet/index.ts` — verifies a wallet signature before registration, so people cannot simply type another person's wallet.
- `.github/workflows/indexer.yml` — scheduled GitHub Actions runner (every five minutes).
- `.env.example` — indexer configuration example.

## Important status

This is a complete deployable project package, **not a website already published on the internet**. Publishing it requires a Supabase project and a GitHub repository/hosting account controlled by you. This chat cannot sign into your accounts or publish into them. The code has not been live-tested against the public RPC from this environment, so complete the validation checklist below before announcing the contest or awarding prizes.

## 1. Create the free database

1. Create a project at [Supabase](https://supabase.com/).
2. Open the project's SQL Editor.
3. Run the entire `schema.sql` file.
4. In Project Settings → API, copy the Project URL and the **service-role/secret key** for the server indexer. Keep the service key private. Copy the **anon/publishable key** for the browser site.

### Set contest dates

Do this before the contest starts. Contest timestamps must include a UTC offset; use `Z` for UTC. This example is only a template — replace the dates with your actual dates:

```sql
update public.maco_contest_config
set contest_name = '$MACO Volume Arena — Week 1',
    start_at = '2026-11-01T00:00:00Z',
    end_at   = '2026-11-08T00:00:00Z',
    registration_deadline = '2026-10-31T23:59:59Z',
    updated_at = now()
where id = true;
```

**Important:** eligible wallets must register before `start_at`. The public leaderboard only counts registered wallets created on or before the start time and trades with timestamps in `[start_at, end_at)`. If you set a start date in the past after the indexer has already passed it, those missed trades will not automatically be backfilled. Set dates first, then start the indexer.

## 2. Configure the public website

Open `index.html` and replace these two placeholders near the bottom:

```js
const SUPABASE_URL = "YOUR_SUPABASE_PROJECT_URL";
const SUPABASE_ANON_KEY = "YOUR_SUPABASE_ANON_KEY";
```

Use the project URL and anon/publishable key. **Never use the service-role key in this file.** The website can be hosted as a static site with [Cloudflare Pages](https://pages.cloudflare.com/), [GitHub Pages](https://pages.github.com/), or [Netlify](https://www.netlify.com/).

## 3. Deploy the secure wallet-registration function

The website requires a wallet signature. Do not enable public direct inserts into `maco_registrations`.

1. Install the [Supabase CLI](https://supabase.com/docs/guides/cli) and sign in.
2. Link your project: `supabase link --project-ref YOUR_PROJECT_REF`.
3. From the project root, deploy the function: `supabase functions deploy register-wallet`.
4. Supabase provides `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` to deployed Edge Functions. Confirm the function is deployed before testing registration.

The function validates the signature, a fresh timestamp and a one-time nonce, then checks the contest registration deadline and start time before inserting the wallet.

## 4. Configure automatic indexing with GitHub Actions

1. Create a private or public GitHub repository and upload the contents of this package to its root.
2. In repository Settings → Secrets and variables → Actions, add these **Secrets**:
   - `SUPABASE_URL` — Supabase project URL.
   - `SUPABASE_SERVICE_ROLE_KEY` — server-only service role/secret key.
   - `ROBINHOOD_RPC_URL` — optional; default public RPC is `https://rpc.mainnet.chain.robinhood.com`.
3. Add these **Variables** if you need to override defaults:
   - `MACO_TOKEN_ADDRESS` = `0x6ffb075F8030C6eA7914c1BA90A8CF87c9A8686d`
   - `LAUNCH_SEARCH_BLOCKS` = `1000000`
   - `INDEX_CHUNK_SIZE` = `1000`
   - `MAX_CHUNKS_PER_RUN` = `100`
   - `ETH_USD_URL` = `https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd`
   - `USD_QUOTE_TOKENS` = `{}` unless the launch uses a non-ETH quote token; then set a JSON object mapping each quote-token address to its USD price.
   - `QUOTE_TOKEN_DECIMALS` = `18` unless a non-ETH quote token requires another decimal count.
4. Go to Actions → **Index MACO trading volume** → Run workflow once. Check the logs for “Found MACO launch” and successful indexing.
5. The workflow then runs every five minutes. GitHub scheduled workflows can be delayed or paused by GitHub; manually run the workflow if needed.

The indexer uses a bounded launch-event search and saves its cursor/pool metadata in `maco_indexer_state`. It is idempotent: `(tx_hash, log_index)` prevents duplicate trade rows. It does not ask for or use any wallet private keys.

## 5. Validate before announcing winners

- Confirm the launch version, curve/pool, and token address against [Pons](https://ponsfamily.com/) and the [Robinhood Chain explorer](https://robinhoodchain.blockscout.com/).
- Run one small known buy and one sell during a test window; compare the wallet, side, quote amount and transaction/log index against the explorer.
- Confirm `maco_trade_volume` rows have the expected timestamps and source labels.
- Confirm the public `maco_leaderboard` view ranks only registered wallets and only trades inside the contest dates.
- Reconcile results at the end of the contest and review suspicious wallet clusters before paying prizes.

## Volume and fairness caveats

- Buys and sells both count, so the same capital can generate volume repeatedly. State this in the public rules.
- USD values for ETH-quoted swaps use an ETH/USD price API when the indexer processes the log. During normal five-minute indexing this is an approximation of the trade-time value; a delayed backfill can differ from historical prices. For a prize contest requiring exact historical USD pricing, add a historical price oracle/source and reconcile before final results.
- Pons V2 non-ETH quote assets must be configured in `USD_QUOTE_TOKENS` and `QUOTE_TOKEN_DECIMALS`; unknown quote assets are deliberately rejected instead of assigned a made-up USD value.
- The indexer cannot prove that separate wallets are not controlled by the same person. Wash trading and linked wallets need manual review; no automatic filter catches every manipulation.
- Public RPC rate limits and occasional outages can delay indexing. The cursor is only advanced after a chunk succeeds, so failed chunks are retried.

## Technical references

- [Pons integration docs](https://docs.ponsfamily.com/)
- [Pons event-indexing reference](https://docs.bitquery.io/docs/blockchain/robinhood/pons-api/)
- [Robinhood Chain connection details](https://docs.robinhood.com/chain/connecting/)
