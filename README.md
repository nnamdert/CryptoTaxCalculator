# Crypto Tax FIFO Calculator

Browser-based crypto tax tool that fetches on-chain transactions, applies FIFO cost basis matching, and exports IRS Form 8949 and TurboTax-compatible CSV files. Single HTML file, 100% client-side — no server, no accounts, no data leaves your browser.

**Live demo:** [nnamdert.page/cryptotax.html](https://nnamdert.page/cryptotax.html)

<img width="1845" height="832" alt="CryptoTaxScreenshot_01" src="https://github.com/user-attachments/assets/8fc38eb3-4378-4d9d-afa4-200a182b9e3b" />


## What It Does

1. **Fetches transactions** from Ethereum (Etherscan V2), BSC (MegaNode), and Solana (Helius) using your own free API keys
2. **Identifies internal transfers** between your wallets and excludes them from taxable events
3. **Prices every transaction** via CryptoCompare, with swap-pair derivation for memecoins/unknown tokens
4. **Runs FIFO matching** across your full transaction history to compute cost basis for each disposal
5. **Flags unresolvable rows** as "Needs Review" instead of guessing — orphan disposals, unpriced tokens, and basis gaps are surfaced clearly
6. **Exports** IRS Form 8949 CSV (filtered by tax year) and TurboTax Custom CSV (all years, for TurboTax's own FIFO carryover)

## Supported Chains & Assets

| Chain | API Provider | Native Asset | Token Support |
|-------|-------------|-------------|---------------|
| Ethereum | Etherscan V2 | ETH | ERC-20 |
| BSC | MegaNode (NodeReal) | BNB | BEP-20 |
| Solana | Helius | SOL, WSOL | SPL tokens, Jupiter swaps, pump.fun memecoins |

Solana support includes Jupiter V2 token name resolution with a reserved-symbol guard that prevents impersonator tokens (common on pump.fun) from polluting your SOL/USDC FIFO queues.

## Quick Start

1. **Get free API keys** (links are in the tool's config panel):
   - [Etherscan V2](https://etherscan.io/apis) — for ETH transactions
   - [MegaNode / NodeReal](https://dashboard.nodereal.io/) — for BSC transactions
   - [Helius](https://dashboard.helius.dev/) — for Solana transactions
   - [CryptoCompare](https://www.cryptocompare.com/cryptopian/api-keys) — **required** for pricing

2. **Add your wallet addresses**, one per line, formatted as `chain:address`:
   ```
   eth:0xYourEthAddress
   bsc:0xYourBscAddress
   sol:YourSolanaAddress
   ```
   List every wallet you own so transfers between them aren't taxed.

3. **Click "Fetch & Calculate"** — the tool fetches, prices, and runs FIFO automatically.

4. **Review the results:**
   - **Summary** shows your total realized gain/loss for the selected tax year
   - **Transactions table** shows every raw transaction with status (OK, Needs Review, Internal Transfer)
   - **Realized Gains** table shows every FIFO-matched disposal with basis, proceeds, and gain/loss
   - **Manual Cost Basis Overrides** panel lets you set basis for orphan disposals
   - **Unpriced Transactions** panel lets you set basis/proceeds for rows the tool couldn't price

5. **Export:**
   - **Form 8949 CSV** — filtered to your tax year, ready for Schedule D
   - **TurboTax Custom CSV** — all years, importable via TurboTax's "Other (CSV)" option
   - **Raw Transactions CSV** — full dataset for your records
   - **Needs Review CSV** — rows requiring manual attention

<img width="1574" height="991" alt="form8949Screenshot_01" src="https://github.com/user-attachments/assets/279e62f7-ae45-461f-a4fc-2700d39be605" />

## Manual Override System

Not every transaction can be automatically priced or matched. The tool provides two override panels:

### Phase 4: Manual Cost Basis Overrides

For disposals where FIFO ran out of acquisition lots (e.g., tokens acquired from an exchange you can't scan, a defunct service like tip.cc, or a wallet you haven't added). Options per row:

- **Ignore** — exclude from 8949 (default)
- **Set basis** — enter the USD cost basis from your records
- **Zero basis** — treat full proceeds as gain (IRS-conservative, always defensible)

Bulk action: **"Zero basis all visible"** applies zero basis to every row above the dollar threshold in one click.

### Phase 3: Unpriced Transactions

For rows that couldn't be priced at all — neither CryptoCompare nor swap-pair derivation could attach a USD value. Two flavors:

- **Orphan receipts** (incoming) — set the cost basis you paid
- **Unpriced disposals** (outgoing) — set the proceeds you received

Both support zero values (zero basis for receipts, zero proceeds for disposals).

Bulk action: **"Zero all unpriced"** applies the conservative treatment to every unpriced row.

### Override Persistence

All overrides are stored in your browser's `localStorage` and persist across sessions. Override keys are based on transaction hash + asset + direction, so they survive wallet additions and code updates.

The tool distinguishes **active** overrides (currently used by FIFO) from **stale** overrides (left over from previous runs). Use **"Clear stale only"** for safe cleanup without losing active work.

## How Pricing Works

The tool uses a three-stage pricing pipeline:

1. **CryptoCompare** — looks up the daily close price for known assets (ETH, SOL, BNB, major tokens). Covers ~95% of transactions.

2. **Swap-pair derivation** — for memecoins and unknown tokens that CryptoCompare doesn't know. If one side of a DEX swap is priced (e.g., SOL), the tool derives the other side's price from the swap ratio. Handles multi-target swaps proportionally.

3. **Manual overrides** — for anything still unpriced after stages 1-2. The user attests a USD value via the Unpriced Transactions panel.

## Solana-Specific Features

- **Jupiter V2 token resolution** — resolves SPL token mints to human-readable names via Jupiter's token list API
- **Reserved symbol guard** — prevents impersonator tokens (e.g., a pump.fun token named "SOL") from corrupting your SOL FIFO queue. Impersonators are renamed to `SYMBOL(mintPrefix…)` format.
- **WSOL aliasing** — wrapped SOL (`So111...1112`) is treated as native SOL for FIFO purposes
- **Helius enhanced transaction parsing** — extracts swap legs, transfers, and fee payments from Solana's complex transaction format

## Data Privacy

- **100% client-side** — no backend server, no database, no analytics, no tracking
- **API keys stay in your browser** — stored in localStorage, never transmitted to any server other than the respective API providers
- **Transaction data stays local** — fetched data is cached in sessionStorage for the current tab only
- **Override data stays local** — stored in localStorage on your machine
- **No cookies, no telemetry, no third-party scripts**

The HTML file can be downloaded and run from `file:///` with full functionality (CORS-permitting).

## Technical Details

- **Single file**: ~2,660 lines of vanilla HTML/CSS/JS, no build step, no dependencies, no frameworks
- **FIFO engine**: canonical key = `wallet|priceId` (not display symbol), ensuring tokens with the same name but different contracts get separate queues
- **Multi-wallet awareness**: internal transfers between user-listed wallets are detected and excluded
- **Tax year filtering**: 8949 export filters to the selected year; TurboTax export includes all years for FIFO carryover
- **Group-aware overrides**: when multiple rows share the same override key (e.g., a Solana transaction with multiple SOL outflows), the override value is distributed proportionally

## Limitations

- **No exchange imports** — only on-chain data. If you traded on Coinbase, MEXC, Binance, etc., those trades aren't captured. Use the manual override system to fill in basis from exchange CSV exports.
- **No DeFi yield/staking** — LP positions, staking rewards, and complex DeFi interactions are not parsed. They'll appear as raw transfers.
- **Price coverage** — CryptoCompare doesn't know every memecoin. Swap-pair derivation catches most, but some rows will need manual pricing.
- **FIFO only** — no LIFO, HIFO, or specific identification methods.
- **ETH, BSC, Solana only** — Bitcoin, Cardano, and other chains are not yet supported. Planned for future versions.
- **Not tax advice** — this tool generates best-effort reports. Always have a CPA review before filing.

## FAQ

**Q: How does the tool price pump.fun memecoins if no aggregator tracks them?**

Swap-pair price derivation. For every DEX swap, the tool looks at both sides. If one side is priced (SOL, USDC, etc.) and the other is an unknown memecoin, the tool assigns the memecoin's USD value equal to the priced side's total. This is IRS-correct: for crypto-to-crypto trades, fair market value is the value of what was given up.

**Q: What if FIFO can't find acquisition basis for a disposal?**

Usually caused by tokens entering your wallet from a source the tool didn't scan (exchange withdrawal, tip.cc, another wallet). The Manual Cost Basis Overrides panel surfaces these sorted by impact. Set a manual basis from your records, or use zero basis (IRS-conservative, maximum tax).

**Q: Can I use this for TY2019–2024?**

Yes. Change the tax year field. The tool fetches complete history so FIFO lots carry across years.

**Q: Staking rewards?**

Not supported in v1. Currently treated as zero-basis acquisitions.

**Q: How do I add a new token to the known-assets list?**

Edit the `KNOWN_ASSETS` object near the top of the `<script>` block. Add the contract address (lowercase for EVM, base58 for Solana), a `priceId` string matching an entry in `CC_SYMBOLS`, and the token's decimals. Save and reload.

## Contributing

Issues, bug reports, and PRs are welcome. The codebase is intentionally a single file for simplicity — keep it that way.

## License

No License with explicit attribution requirements. You may use, modify, distribute, and sell this software freely, provided you maintain visible attribution to **nnamdert** as the original author and include the full license text. See [LICENSE](LICENSE) for details.
