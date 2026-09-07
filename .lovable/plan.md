# Add live ETH price next to wallet balance

## Goal
Show the connected wallet's ETH balance together with its current USD market value inside the existing wallet dropdown.

## What will be built
1. A public server function that fetches the latest ETH/USD price from a free public API, with a short server-side cache to avoid rate limits.
2. Update the wallet dropdown (`WalletButton`) to fetch the price when a wallet is connected and display the USD value next to the raw ETH amount.
3. Keep the existing USDG, GOLD, and COINS rows untouched.

## Why a server function?
Calling the price API from the server avoids CORS issues in the browser and lets us cache the result. No API key is required for the public endpoint we will use.

## Visual result
```text
ETH    0.1234
       ≈ $300.50 USD
USDG   0.00
GOLD   1,234
COINS  5,678
```

## Verification
- Build passes without errors.
- In the preview, connect a wallet on the Robinhood chain and see the ETH row show both the balance and its USD value.
