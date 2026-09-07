# GoFish ("Pulau Pancing") — Economy & Game Systems Documentation

**Codebase analyzed:** `gofish-main` (TanStack Start v1 + React + Three.js/React Three Fiber, Tailwind v4, shadcn/ui, Supabase/Postgres backend, Drizzle migrations, wagmi/viem wallet auth on the Robinhood chain, id `4663`)

**Document scope:** A deep, code-level breakdown of the project's economic system, gameplay systems, and every player-facing feature, reconstructed directly from the server functions (`src/lib/*.functions.ts`), the RNG/economy logic (`src/lib/fishRules.ts`, `src/lib/xp.ts`), and the authoritative SQL business logic in `drizzle/migrations/0000`–`0013`.

---

## 1. High-Level Architecture

| Layer | Technology | Responsibility |
|---|---|---|
| Client (game) | React Three Fiber / Three.js, Zustand-style stores, TanStack Router | Rendering, input, casting animation, UI panels (shop, quests, gold, leaderboard, chat) |
| Server functions | TanStack Start `createServerFn` (`src/lib/*.functions.ts`) | Wallet-signature verification, thin request validation (Zod), delegates all game-state mutation to Postgres RPCs |
| Database | Supabase Postgres, `SECURITY DEFINER` PL/pgSQL functions | **Sole source of truth.** All RNG rolls, currency balances, gear ownership, quest progress, and withdrawal logic run server-side in SQL — the client cannot supply a roll result |
| On-chain read | viem `createPublicClient` against the Robinhood chain (chain id `4663`) | Reads an ERC-20 balance for a placeholder "hold token," resolves USD value via GeckoTerminal, and periodically snapshots it |
| Auth | Wallet signature ("Sign-In with Ethereum"-style message), 30-minute signature validity | Every privileged server function requires a re-verifiable `{address, issuedAt, signature}` proof |

Key architectural principle repeated throughout the codebase: **Postgres never talks to the blockchain**, and **the client never supplies gameplay RNG outputs**. Every catch, sale, purchase, quest advance, and gold claim is rolled/computed inside a `SECURITY DEFINER` SQL function that only `service_role` may execute.

---

## 2. Dual-Currency Economic System

GoFish runs **two separate currencies** with fundamentally different purposes:

| Currency | Column | Backed by | Earned via | Spent on | Withdrawable to real value? |
|---|---|---|---|---|---|
| **Coins** | `profiles.coins` | Nothing (pure in-game soft currency) | Selling caught fish, claiming quest rewards | Rods, baits, boats (shop) | No |
| **Gold** | `profiles.gold` | Treasury (assumed always sufficient) | Daily NPC reward event (fish turn-ins), gated by level + on-chain token holding | Withdrawal requests → paid out off-chain by an admin, `tx_hash` recorded | **Yes** — gold is explicitly convertible/redeemable |

### 2.1 Coins Economy (Soft Currency)

**Earning coins — selling fish**
`sell_fish(_wallet, _item_id, _species_id, _sell_all)` deletes the matching inventory rows and credits:

```
earned = SUM( weight_kg × fish_species.base_price_per_kg × mutation.multiplier )  [rounded]
```

It supports three modes in one call: sell one specific fish (`_item_id`), sell every fish of one species (`_species_id`), or sell the entire bucket (`_sell_all = true`).

**Spending coins — the Shop**
Coins buy permanent gear ownership rows in `player_rods`, `player_baits`, `player_boats`. Every purchase function (`buy_rod`, `buy_bait`, `buy_boat`) enforces, atomically in SQL:
1. The tier exists and isn't already owned.
2. `profile.level >= tier.min_level` (level gate — see §4.4).
3. `profile.coins >= tier.price_coins` (deducted in the same `UPDATE ... RETURNING`).
4. Advances any active quest requirement of type `buy_rod` / `buy_bait` / `buy_boat` keyed on that tier id.

### 2.2 Gold Economy (Hard Currency / Real-Value-Backed)

Gold is the "real money bridge" of the game. It is earned from a **daily NPC reward event** and is the only balance that can be **withdrawn**. Access to gold generation is double-gated:

1. **Level gate** — `profile.level >= npc_reward_config.min_level` (seeded to `5`).
2. **On-chain hold-value gate** — the wallet must be holding (and have *provably held, continuously*) at least the tracked token's USD value threshold for a tier (see §2.3).

#### 2.2.1 The Daily NPC Reward Event

A `npc_reward_events` row is created once per UTC day (idempotently, by cron or on-demand):

| Field | Description |
|---|---|
| `spawn_at` | `00:00 UTC` |
| `expire_at` | `spawn_at + 2 hours` — a **2-hour daily claim window** |
| `base_requirement` | 3 randomized fish-turn-in tiers: `common`, `rare`, `epic`, each with a random qty drawn from configured min/max |
| `bonus_requirement` | 2 fixed bonus tiers: `legendary` and `mythic`, each worth extra gold, claimable **once per event/visit** |

**`npc_reward_config` defaults:**

| Key | Default Value | Meaning |
|---|---|---|
| `common_min` / `common_max` | 100 / 200 | Random qty range for the common turn-in package |
| `rare_min` / `rare_max` | 40 / 100 | Random qty range for the rare turn-in package |
| `epic_min` / `epic_max` | 20 / 60 | Random qty range for the epic turn-in package |
| `base_reward_gold` | 1 | Gold paid per complete base package (all 3 rarities turned in together) |
| `bonus_legendary_qty` / `bonus_legendary_gold` | 5 / 2 | Turn in 5 legendary fish once per visit → +2 gold |
| `bonus_mythic_qty` / `bonus_mythic_gold` | 1 / 3 | Turn in 1 mythic fish once per visit → +3 gold |
| `min_level` | 5 | Minimum player level to see/claim the NPC at all |

**Claim resolution logic (`claim_npc_reward`, SQL, atomic):**
- **Base packages** are all-or-nothing bundles: the number of packages claimable = `floor(stock_of_rarity / required_qty)`, taken as the **minimum across all three base rarities** (the scarcest resource caps the claim).
- Fish are consumed **oldest-first** (`ORDER BY caught_at ASC`) from `fish_inventory_items`, joined to `fish_species` for rarity (inventory itself has no rarity column).
- **Bonus** is a single yes/no claim per visit — only pays out if the wallet has *enough* legendary/mythic stock for the full bonus bundle.
- **Every visit is hard-capped** by the wallet's hold-tier's `generation_cap_gold` — base gold is truncated by remaining "room," and if there isn't enough room left for the bonus bundle, the bonus is skipped entirely for that visit.
- Every gold change is written to `gold_ledger` (see §2.4) with a `reason` and the resulting `balance_after`.

#### 2.2.2 On-Chain Hold Tiers (`fish_hold_tiers`)

The USD value gate is resolved from a real on-chain ERC-20 balance × a live GeckoTerminal price (cached 60s) against a placeholder token address on the Robinhood chain (chain id `4663`; the code explicitly notes this is a placeholder for the not-yet-launched real `$FISH` token).

| Tier ID | Min USD Held | Gold Generation Cap / Visit | Withdraw Min | Withdraw Max | Withdrawals / Day | Min Continuous Hold Time |
|---|---:|---:|---:|---:|---:|---:|
| `tier_10` | $10 | 5 gold | 5 | 5 | 1 | 24 hours |
| `tier_100` | $100 | 15 gold | 5 | 15 | 1 | 24 hours |
| `tier_1000` | $1,000 | 30 gold | 5 | 30 | 1 | 48 hours |
| `tier_2000` | $2,000 | Unlimited (`NULL`) | Unlimited (`NULL`) | Unlimited (`NULL`) | 1 | 72 hours |

**Anti-abuse fix (migration `0010`):** the original design gated everything on an *instant* balance snapshot, which allowed flash-loaning tokens or rotating one balance across many wallets to fake eligibility. The fix introduces `fish_hold_snapshots` (periodic USD-value samples per wallet) and `resolve_windowed_tier()`, which only grants a tier if **every snapshot across the tier's `min_hold_hours` window is at or above that tier's threshold**, and there is enough snapshot history to prove the window is actually covered (1-hour slack for sampling-cadence gaps). A brand-new wallet — no matter how large its instant balance — cannot claim a tier immediately. The system distinguishes:
- **`tier` (windowed/gating)** — the only value used to gate NPC claims and withdrawals.
- **`instantTier` (display-only)** — "you'd be at tier X right now if it stays there for N more hours," never used for access control.

#### 2.2.3 Withdrawals (Gold → Real Payout)

| Function | Actor | Effect |
|---|---|---|
| `getHoldStatus` | Player | Read-only preview of current USD hold value + resolved tier, for client-side UX before submitting |
| `requestWithdrawal` | Player | Re-resolves the tier **server-side** (never trusts a client-supplied tier), validates `amount` against `wd_min`/`wd_max`, checks the wallet hasn't hit `wd_per_day` for pending/paid requests today, then atomically debits `gold` and inserts a `pending` `withdrawal_requests` row + a `withdrawal_locked` ledger entry |
| `getMyWithdrawals` | Player | Own withdrawal history, most recent first (max 50) |
| `cancelWithdrawal` | Player | Self-cancel a still-`pending` request — gold is refunded immediately (no need to wait for admin or expiry) |
| `admin_mark_withdrawal` | Admin (server-checked wallet) | Approve (`status → paid`, records `tx_hash`) or reject (`status → rejected`, refunds gold + `withdrawal_rejected_refund` ledger entry) |
| `expire_stale_withdrawals` (cron) | System | Auto-refunds and marks `expired` any `pending` request older than `game_config.withdrawal_expire_hours` (default **168 hours / 7 days**) |

`withdrawal_requests.status` lifecycle: `pending → {paid | rejected | expired | cancelled}`.

#### 2.2.4 Gold Ledger (Audit Trail)

Every gold-balance mutation is append-only logged to `gold_ledger` with an enumerated `reason`:

| Reason | Trigger | Sign |
|---|---|---|
| `npc_base_claim` | Base package(s) claimed from the daily NPC | + |
| `npc_bonus_claim` | Bonus-only claim (base already exhausted this visit, bonus still available) | + |
| `withdrawal_locked` | Player submits a withdrawal request | − |
| `withdrawal_rejected_refund` | Admin rejects a pending withdrawal | + |
| `withdrawal_expired_refund` | Auto-expiry cron refunds a stale pending withdrawal | + |
| `admin_adjustment` | Manual admin correction (reserved) | ± |

Each row stores `amount`, `balance_after` (post-mutation balance, for auditability without recomputation), and a `metadata` JSON blob (e.g. which fish were consumed, which withdrawal id).

---

## 3. Core Gameplay Loop — Fishing

### 3.1 Casting & Catch Roll (`record_catch`, fully server-authoritative)

The client sends only an **advisory** `weatherKind` hint (validated against real rows, silently falls back to `"cerah"`/clear if invalid) — species, rarity, weight, and mutation are **100% rolled server-side**. This closed a prior client-authoritative RNG exploit (migration `0005`).

**Step-by-step resolution inside `record_catch`:**

1. **Cooldown check** — rejects if `now() < last_cast_at + cast_cooldown_seconds` (default **1.5s**, tunable via `game_config`).
2. **Weather resolution** — looks up `weather_effects` for the resolved kind; pulls its `rarity_multiplier` JSON map.
3. **Equipped gear lookup** — reads the wallet's `equipped = true` row in `player_rods` / `player_baits` (falling back to `starter` rod / `basic_bait` if none equipped).
4. **Combined luck factor:**
   ```
   luck = (1 + rod.luck_percent / 100) × (1 + bait.luck_percent / 100)
   ```
5. **Weighted species pool** — every non-monster species whose `min_weight_kg <= rod.max_catch_weight_kg` (i.e., **too-large-for-your-rod species are excluded from the roll entirely**) enters a weighted pool:
   ```
   species_weight = rarity_base_weights[rarity]
                     × (luck, if rarity ≠ "common"; else ×1)
                     × bait.rarity_multiplier[rarity]  (default 1)
                     × weather.rarity_multiplier[rarity]  (default 1)
   ```
   A uniform roll against the cumulative weight picks the species.
6. **Weight roll** — `weight_kg = round(min_weight_kg + random() × (max_weight_kg − min_weight_kg), 2)`.
7. **Mutation roll** — an independent weighted roll over the `mutations` table (drop-weight based), applied on top of the catch.
8. **Monster fish** *(current behavior, migration `0006`, superseding an earlier separate 2% roll)* — monster species (e.g. **Ancient Leviathan**) are **not** a special-cased roll anymore. They compete inside the exact same weighted pool as everything else, gated by the same `min_weight_kg <= rod.max_catch_weight_kg` filter. Since the Leviathan's `min_weight_kg = 1200`, only a wallet with a **Mythic Rod** (cap 1500) can ever roll it — this was a deliberate fix so gear progression actually matters for the single largest potential payout in the game, instead of every rod having an equal flat shot at it.
9. **Persist** — inserts the caught fish into `fish_inventory_items`, increments the matching `fish_<rarity>` counter on the profile, adds XP, recomputes level, and stamps `last_cast_at = now()`.
10. **Quest hook** — best-effort call to `advance_quest_progress` for the `catch_count` requirement type (failure here never fails the catch itself).

### 3.2 XP & Leveling

XP curve (mirrors the SQL `level_for_xp` function):

```
xp_for_level(N) = 100 × (N − 1)²        // total XP required to REACH level N
level_for_xp(xp) = floor( sqrt(xp / 100) ) + 1
```

| Level | Total XP to reach | XP span for that level |
|---:|---:|---:|
| 1 | 0 | 100 |
| 2 | 100 | 200 |
| 3 | 400 | 500 |
| 4 | 900 | 700 |
| 5 | 1,600 | 900 |
| 10 | 8,100 | 1,900 |
| 20 | 36,100 | 3,900 |

XP awarded per catch (base, before mutation multiplier):

| Rarity | Base XP |
|---|---:|
| common | 10 |
| rare | 25 |
| epic | 60 |
| legendary | 150 |
| mythic | 400 |
| *(unknown/fallback)* | 5 |

`xp_gained = max(1, round(base_xp_for_rarity × mutation.multiplier))`.

### 3.3 Weather System

| Weather | Bite Window (s) | Rarity Multipliers |
|---|---:|---|
| Cerah (Clear) | 1.6 | none |
| Berawan (Cloudy) | 1.6 | none |
| Berkabut (Foggy) | 1.3 | epic ×1.3, legendary ×1.3, mythic ×1.3 |
| Hujan (Rain) | 1.1 | epic ×1.3, legendary ×1.5, mythic ×1.5 |
| Badai (Storm) | 0.9 | legendary ×1.8, mythic ×2.5 |

**Bite window** = the reaction time the player has to hook a fish once it bites — harsher weather shortens the window but boosts rare-fish odds, a risk/reward trade-off.

**Weather cycle:** changes every `240` seconds by default, drawn from a weighted pool:

| Weather | Cycle Weight |
|---|---:|
| Cerah | 40 |
| Berawan | 25 |
| Berkabut | 15 |
| Hujan | 12 |
| Badai | 8 |

### 3.4 Mutations

Applied multiplicatively to both **sell price** and **XP gained** for a catch:

| Mutation | Multiplier | Drop Weight (relative odds) |
|---|---:|---:|
| Normal (none) | ×1.0 | 55 |
| Big | ×1.2 | 15 |
| Dark | ×1.3 | 10 |
| Albino | ×1.4 | 7 |
| Sparkling | ×1.5 | 5 |

### 3.5 Fish Species Catalog (seeded/fallback data)

| Species | Rarity | Weight Range (kg) | Monster? | Base Price / kg (coins) | Rarity Pool Weight |
|---|---|---:|:---:|---:|---:|
| Clownfish | Common | 5 – 40 | No | 4 | 100 |
| Mackerel | Rare | 35 – 120 | No | 6 | 45 |
| Scad | Epic | 100 – 300 | No | 9 | 18 |
| Red Snapper | Legendary | 280 – 650 | No | 14 | 6 |
| Baby Tuna | Mythic | 600 – 1,300 | No | 22 | 2 |
| Ancient Leviathan | Mythic | 1,200 – 3,000 | **Yes** | 40 | 2 (shares "mythic" bucket) |

Example illustrative sell value: a 1,300 kg Baby Tuna at Normal mutation ≈ `1300 × 22 = 28,600` coins; an Ancient Leviathan at max weight ≈ `3000 × 40 = 120,000` coins before any mutation bonus.

---

## 4. Gear Progression (Shop Systems)

Three parallel gear systems — **Rods**, **Baits**, **Boats** — each with 6 tiers, each individually ownable (`player_rods`/`player_baits`/`player_boats`) and equippable (only one equipped per category at a time). Buying **requires both** enough coins **and** meeting the tier's level gate; equipping only requires ownership (the level gate was already proven at purchase time).

### 4.1 Rod Tiers

| Tier | Max Catch Weight (kg) | Luck Bonus | Speed Bonus (reel time reduction) | Price (coins) | Min. Level |
|---|---:|---:|---:|---:|---:|
| Starter Rod | 10 | +0% | +0% | 0 | 1 |
| Uncommon Rod | 40 | +10% | +5% | 1,000 | 5 |
| Rare Rod | 100 | +25% | +12% | 10,000 | 10 |
| Epic Rod | 250 | +50% | +22% | 60,000 | 20 |
| Legendary Rod | 600 | +80% | +35% | 250,000 | 35 |
| Mythic Rod | 1,500 | +130% | +50% | 1,000,000 | 50 |

*"Max Catch Weight" is a hard filter on the species-roll pool, not just a display stat — see §3.1 step 5. It is the single lever that unlocks access to heavier/rarer fish, including the game's only monster species.*

### 4.2 Bait Tiers

| Tier | Luck Bonus | Price (coins) | Min. Level |
|---|---:|---:|---:|
| Basic Bait | +0% | 0 | 1 |
| Uncommon Bait | +20% | 1,000 | 5 |
| Rare Bait | +50% | 15,000 | 10 |
| Epic Bait | +95% | 120,000 | 20 |
| Legendary Bait | +160% | 600,000 | 35 |
| Mythic Bait | +250% | 2,000,000 | 50 |

*(`rarity_multiplier` per bait is present as a JSONB column for future per-rarity tuning but currently seeded empty — bait's effect today is purely the flat luck bonus.)*

### 4.3 Boat Tiers

| Tier | Speed Bonus | Price (coins) | Min. Level |
|---|---:|---:|---:|
| Wooden Dinghy | 100% (baseline) | 0 | 1 |
| SS Minnow | 130% | 5,000 | 5 |
| Reef Runner | 160% | 40,000 | 10 |
| Bow Raider | 200% | 200,000 | 20 |
| Sea Marshal | 250% | 800,000 | 35 |
| Vex Yacht | 320% | 3,000,000 | 50 |

### 4.4 Unified Level-Gate Ladder

All three gear systems share one progression ladder (mapped by `sort_order`, not by id, so it's consistent across all three tables):

| Sort Order | Tier Name Pattern | Min. Level |
|---:|---|---:|
| 1 | Starter / Basic / Wooden | 1 |
| 2 | Uncommon | 5 |
| 3 | Rare | 10 |
| 4 | Epic | 20 |
| 5 | Legendary | 35 |
| 6 | Mythic | 50 |

**Total cost to fully max out one full loadout (rod + bait + boat, all 6 tiers each):** ≈ 1,000,000 + 2,000,000 + 3,000,000 (top tiers alone) plus every lower tier ≈ **8.1 million coins**, which is exactly why the (dev-only, env-flag-gated) test-coin grant is set to 10,000,000.

---

## 5. Quest System

A **single active quest slot per wallet** (`player_quest_progress`), advancing linearly through a fixed **10-quest chain** (`quest_definitions`, `order_index` 1–10). Quests are explicitly **decoupled from level gating** — they are a pure "natural progress" bonus track that rewards **both coins and XP**.

### 5.1 Requirement Model (current, migration `0012`)

Each quest's `requirement` is a JSON **array** of requirement objects (not a single requirement) — a quest can demand catching, selling, *and* buying gear all at once:

```json
[{ "type": "catch_count", "key": "common", "qty": 250 },
 { "type": "sell_count",  "key": "common", "qty": 100 }]
```

| Requirement Type | `key` meaning | Advanced by |
|---|---|---|
| `catch_count` | fish rarity | every server-confirmed catch (`record_catch`) |
| `sell_count` | fish rarity | every sale (`sell_fish`) |
| `buy_rod` | rod tier id | `buy_rod` |
| `buy_bait` | bait tier id | `buy_bait` |
| `buy_boat` | boat tier id | `buy_boat` |

`player_quest_progress.progress_value` is a JSON array of counters, index-aligned with the active quest's `requirement` array. A quest is claimable once **every** counter meets its `qty`.

### 5.2 The 10-Quest "Extreme" Chain

| # | Title | Requirements (summary) | Reward Coins | Reward XP |
|---:|---|---|---:|---:|
| 1 | A Challenging Start | Catch 250 common → Sell 100 common | 500 | 150 |
| 2 | The Angler's Gear | Catch 150 common, Sell 50 common, Buy Uncommon rod, Buy Uncommon bait | 1,200 | 400 |
| 3 | Rare Pursuit | Catch 100 common + 50 rare, Sell 50 common + 20 rare | 2,500 | 800 |
| 4 | Breaking Into Epic | Catch 50 common + 25 epic, Sell 30 rare, Buy Rare rod | 5,000 | 1,500 |
| 5 | Seasoned Angler | Catch 80 rare + 15 epic, Sell 40 epic, Buy Rare bait | 9,000 | 2,500 |
| 6 | New Captain | Catch 40 epic + 5 legendary, Buy Epic rod, Buy Reef Runner boat | 16,000 | 4,000 |
| 7 | A Legend in the Making | Catch 20 legendary, Sell 60 epic, Buy Epic bait, Buy Bow Raider boat | 30,000 | 7,000 |
| 8 | Conqueror of the Depths | Catch 60 epic + 10 legendary + 2 mythic, Buy Legendary rod | 55,000 | 12,000 |
| 9 | On the Verge of Myth | Catch 15 legendary + 5 mythic, Buy Legendary bait, Buy Sea Marshal boat | 90,000 | 18,000 |
| 10 | The True Mythic Angler | Catch 10 mythic, Sell 5 legendary, Buy Mythic rod + bait + Vex Yacht boat | 150,000 | 30,000 |

**Design intent (from the migration comment):** the chain is engineered so completing it means the player has "tried everything gofish has to offer" — every rarity, every sale action, and every non-starter gear tier across all three shop systems.

**End-of-chain behavior:** once quest 10 is claimed, `current_quest_order` stays pinned at 10 and `status = 'claimed'` forever; the client must check `status`, not the truthiness of the response, to detect "no more quests."

---

## 6. Social & Meta Systems

### 6.1 Leaderboard

`getLeaderboard(sortBy)` returns the **top 50** players sorted by one of three metrics, plus the caller's own rank even if outside the visible top 50:

| Sort Mode | Metric |
|---|---|
| `xp` | Total XP / level |
| `coins` | Coin balance |
| `fish` | Total fish caught |

All reads go through `get_leaderboard` / `get_my_leaderboard_rank` SQL functions — the client never queries `profiles` directly.

### 6.2 Global Chat

| Property | Value |
|---|---|
| Message length | 1 – 240 characters |
| Cooldown | 2 seconds between messages per wallet (`chat_cooldown_seconds`) |
| Retention | Only the most recent **200 messages globally** are kept (older ones pruned on every send) |
| Delivery | Supabase Realtime subscription on `chat_messages` (public-read policy) |
| Identity | Denormalized snapshot of `username` / `display_name` / `avatar_url` at send time, so historical messages remain stable even if a player later renames |

### 6.3 Profile System

- Wallet-address-keyed profile (`profiles.wallet_address` is the primary key — no separate user id).
- Auto-created on first `ensureProfile` call with a generated `angler_<addr-fragment>` username; collision-retried up to 5 times.
- `updateProfile` — editable username (3–20 chars, alnum + underscore, case-insensitively unique) and display name (≤40 chars).
- `uploadAvatar` — base64 image upload (png/jpeg/webp/gif, ≤5 MB) to Supabase Storage under a per-wallet path.
- `getInventory` — up to 500 most recent unsold fish.
- **Dev-only test-coin faucet** (`grantTestCoins`) — grants 10,000,000 coins, but hard-gated behind a server-only `ENABLE_TEST_COINS=true` environment flag (never client-controlled); explicitly flagged in code comments as **must be deleted before public release**.

### 6.4 Admin Panel (`/admin`)

A wallet-gated route for processing gold withdrawals:
- Connect + sign-in with the admin wallet (same wallet-proof mechanism as the game).
- Lists pending withdrawal requests.
- Approve (requires entering a `tx_hash` first) or reject each request via `admin_mark_withdrawal`, which re-validates status server-side (`pending` only, no double-processing).

---

## 7. Authentication & Security Model

| Mechanism | Detail |
|---|---|
| Wallet proof | `{ address, issuedAt, signature }`, verified with `viem.verifyMessage` against a constructed auth message |
| Signature validity window | **30 minutes** (`SIGNATURE_MAX_AGE_MS`) — shortened from an original 24 hours specifically to shrink replay-attack exposure |
| Session model | The wallet signs **once per session**; the same proof is reused for every subsequent action (cast, sell, buy, equip). A full single-use-nonce-per-call model was considered and explicitly deferred as a larger architectural change |
| Authorization boundary | Every privileged SQL function is `SECURITY DEFINER`, with `REVOKE ALL ... FROM PUBLIC, anon, authenticated` and `GRANT EXECUTE ... TO service_role` only — **no direct client access to any mutation**, even with a valid Supabase session |
| Row-level security | Enabled on every table; gameplay/economy tables intentionally have **no** anon/authenticated policies (all writes route through verified server code) |
| RNG integrity | Historical fix: catch results were briefly client-suppliable (dupe/cheat vector) — closed by making `record_catch` fully server-rolled with no roll-result parameters accepted from the caller |

---

## 8. Data Model Reference (Selected Tables)

| Table | Purpose |
|---|---|
| `profiles` | Wallet-keyed player record: `coins`, `gold`, `xp`, `level`, per-rarity catch counters, `last_cast_at` |
| `fish_inventory_items` | Unsold caught fish (species, weight, mutation, timestamp) |
| `fish_species`, `rarity_base_weights` | Species catalog and rarity roll weights |
| `rod_tiers`, `bait_tiers`, `boat_tiers` | Gear catalogs (stats, price, level gate, sort order) |
| `player_rods`, `player_baits`, `player_boats` | Per-wallet ownership + equipped flag |
| `mutations` | Mutation catalog (multiplier + drop weight) |
| `weather_effects`, `weather_cycle_config` | Weather catalog + cycling weights |
| `game_config` | Global tunables (`cast_cooldown_seconds`, `monster_catch_chance` *(legacy, unused since 0006)*, `withdrawal_expire_hours`, `chat_cooldown_seconds`) |
| `gold_ledger` | Append-only audit trail of every gold balance change |
| `npc_reward_config`, `npc_reward_events`, `npc_reward_claims` | Daily gold-NPC event configuration, instance, and per-wallet claim progress |
| `fish_hold_tiers`, `fish_hold_snapshots` | On-chain hold-value tier definitions + periodic balance snapshots for windowed eligibility |
| `withdrawal_requests` | Gold withdrawal lifecycle records |
| `quest_definitions`, `player_quest_progress` | The 10-quest chain and each wallet's single active-slot progress |
| `chat_messages` | Global chat feed (200-message rolling retention) |

---

## 9. Summary of Notable Design Decisions & Fixes (from migration history)

| Migration | Fix / Feature |
|---|---|
| `0005` | Made `record_catch` fully server-authoritative (closed a client-RNG dupe exploit); added cast cooldown; shortened signature validity 24h → 30min |
| `0006` | Removed the separate flat 2% monster-catch roll; folded monster species into the normal rod-cap-gated weighted pool so gear progression actually matters for the best payout in the game |
| `0007` | Added `min_level` gates (1/5/10/20/35/50) uniformly across rods, baits, and boats |
| `0008` | Introduced the entire gold economy: `gold` balance, ledger, daily NPC reward event, on-chain hold tiers, withdrawal requests, admin approval flow, and the original (later replaced) simple quest chain |
| `0009` | Fixed a fish-inventory race condition (not detailed above; concurrency hardening) |
| `0010` | Replaced instant-balance tier gating with a continuous-holding time window (`fish_hold_snapshots` + `resolve_windowed_tier`) to prevent flash-loan / balance-rotation abuse |
| `0011` | Added withdrawal auto-expiry (168h default) and player self-cancel with immediate refund |
| `0012` | Rebuilt the quest chain from single-requirement/coins-only to 10 multi-requirement quests paying both coins and XP, with hooks from catching, selling, and every gear-purchase action |
| `0013` | Added the leaderboard (3 sort modes + "my rank") and global chat (240-char, 2s cooldown, 200-message retention, Realtime-backed) |

---

*Document generated from static analysis of the uploaded `gofish-main` source tree — server functions, Postgres migrations, and RNG/economy constants — reflecting the system's current (post-`0013`) authoritative behavior.*
