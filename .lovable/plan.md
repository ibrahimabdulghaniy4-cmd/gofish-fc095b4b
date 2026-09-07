# Wallet panel: live ETH price + USDG renamed to FISH

## Goal
1. Show the connected wallet's ETH balance together with its current USD market value.
2. Replace the USDG row with a FISH token row that uses the uploaded round FISH logo. The FISH contract address is not available yet, so the balance will show 0 until the contract address is provided and wired in.

## What will be built

### A. Live ETH price
- A small public server function that fetches the latest ETH/USD price from a free public price API, with a short server-side cache to avoid rate limits and no API key needed.
- `WalletButton` fetches the price once a wallet is connected on the Robinhood chain and shows the USD value under the ETH amount.

### B. USDG -> FISH
- Upload the provided logo as a CDN asset (round blue badge with the angler character) and use it for the token row.
- In `WalletButton`, rename the "USDG" row to "FISH", swap the logo, and keep the balance at 0 as a placeholder until the contract address is shared. A named constant (e.g. `FISH_TOKEN_ADDRESS` in the chains config) will hold the contract address so it is one-line to fill in later.
- The only USDG reference in the app is that single wallet row, so nothing else changes.

## Visual result
```text
ETH    0.1234
       ≈ $300.50 USD
FISH   0          (new round logo)
GOLD   1,234
COINS  5,678
```

## Out of scope / waiting on you
- Real FISH balance from the smart contract: needs the token contract address. Send it when ready and I will wire live balances.

## Verification
- Build passes without errors.
- In the preview, connect a wallet: ETH row shows balance + USD value, and the FISH row shows the new logo.
