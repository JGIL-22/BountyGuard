# 🛡 BountyGuard

> Trustless bug bounty escrow on Stellar Soroban — built at White Cloak Technologies Stellar Bootcamp 2026.

Freelance security researchers in Southeast Asia submit vulnerability reports. Companies lock USDC in a Soroban smart contract. When the company calls `approve_bounty`, funds release automatically — no middlemen, no delayed payouts.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Next.js 15 Frontend (React + TypeScript)               │
│  ├─ Freighter wallet integration                        │
│  ├─ Soroban contract client (simulate + sign+submit)    │
│  └─ Cyber-noir UI (Space Mono + Syne fonts)             │
└───────────────────────┬─────────────────────────────────┘
                        │ Stellar SDK / XDR
┌───────────────────────▼─────────────────────────────────┐
│  Soroban Smart Contract (Rust)                          │
│  ├─ create_bounty   → locks USDC into escrow            │
│  ├─ submit_report   → researcher submits hash on-chain  │
│  ├─ approve_bounty  → auto-releases USDC to researcher  │
│  ├─ dispute_bounty  → halts payout for review           │
│  ├─ cancel_bounty   → refunds company if Open           │
│  ├─ get_bounty      → read a single bounty              │
│  └─ get_bounty_count→ total bounties created            │
└───────────────────────┬─────────────────────────────────┘
                        │ Soroban RPC
              Stellar Testnet / Mainnet
```

---

## Prerequisites

- **Rust** 1.95+ with `wasm32-unknown-unknown` target (you already have this ✓)
- **Node.js** 20+
- **Stellar CLI** (`cargo install --locked stellar-cli --features opt`)
- **Freighter** browser extension installed + funded testnet account ✓

---

## Step 1 — Install Stellar CLI

```bash
cargo install --locked stellar-cli --features opt
```

Verify:
```bash
stellar --version
```

---

## Step 2 — Configure Stellar CLI for Testnet

```bash
stellar network add testnet \
  --rpc-url https://soroban-testnet.stellar.org \
  --network-passphrase "Test SDF Network ; September 2015"
```

Generate (or import) a deployer identity:
```bash
# Generate new keypair
stellar keys generate deployer --network testnet

# OR import your existing secret key
stellar keys add deployer --secret-key
```

Fund it via Friendbot:
```bash
stellar keys fund deployer --network testnet
```

---

## Step 3 — Build the Smart Contract

From the project root:
```bash
cd contracts/bountyguard
stellar contract build
```

The compiled `.wasm` will be at:
```
contracts/bountyguard/target/wasm32-unknown-unknown/release/bountyguard.wasm
```

---

## Step 4 — Deploy to Testnet

```bash
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/bountyguard.wasm \
  --source deployer \
  --network testnet
```

Copy the **Contract ID** from the output — it looks like:
```
CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

---

## Step 5 — Set Up the Frontend

```bash
cd frontend
npm install
cp .env.example .env.local
```

Edit `.env.local`:
```env
NEXT_PUBLIC_SOROBAN_CONTRACT_ID=<YOUR_CONTRACT_ID_HERE>
NEXT_PUBLIC_STELLAR_READ_ADDRESS=<ANY_FUNDED_TESTNET_ADDRESS>
```

For `NEXT_PUBLIC_STELLAR_READ_ADDRESS`, use your Freighter wallet address (it must have some XLM). Fund it at https://friendbot.stellar.org if needed.

---

## Step 6 — Run the Dev Server

```bash
npm run dev
```

Open http://localhost:3000 — you'll see the BountyGuard dashboard.

---

## Smart Contract Functions

| Function | Who Calls It | Description |
|---|---|---|
| `create_bounty(company, title, amount, token)` | Company | Locks USDC in escrow, returns bounty ID |
| `submit_report(researcher, bounty_id, report_hash)` | Researcher | Submits SHA-256/IPFS hash on-chain |
| `approve_bounty(company, bounty_id)` | Company | Auto-releases escrowed USDC to researcher |
| `dispute_bounty(company, bounty_id)` | Company | Halts payout, marks as Disputed |
| `cancel_bounty(company, bounty_id)` | Company | Refunds USDC if bounty is still Open |
| `get_bounty(bounty_id)` | Anyone (read) | Returns full Bounty struct |
| `get_bounty_count()` | Anyone (read) | Returns total bounties created |

---

## Bounty Status Flow

```
Open ──────────────────────────────────► Cancelled (company cancels + refund)
  │
  └─► [Researcher submits] ──► Submitted
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                       Approved            Disputed
                    (USDC released)     (manual review)
```

---

## Testing with Stellar CLI

After deploying, you can test directly via CLI:

```bash
# Create a bounty (replace addresses/amounts)
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet \
  -- create_bounty \
  --company <YOUR_ADDRESS> \
  --title "Test SQL Injection Bounty" \
  --amount 5000000000 \
  --token <USDC_CONTRACT_ID>

# Read bounty count
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet \
  -- get_bounty_count

# Get bounty details
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet \
  -- get_bounty \
  --bounty_id 1
```

---

## Testnet USDC

For testnet USDC, use the Circle testnet USDC contract:
```
GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5
```

Or use **native XLM** for quick testing by updating `.env.local`:
```env
NEXT_PUBLIC_SOROBAN_ASSET_ADDRESS=native
NEXT_PUBLIC_SOROBAN_ASSET_CODE=XLM
```

And update the `create_bounty` call in `contract-client.ts` to use `native` as the token.

---

## Production Build

```bash
cd frontend
npm run build
npm start
```

---

## Project Structure

```
bountyguard/
├── Cargo.toml                          # Rust workspace
├── contracts/
│   └── bountyguard/
│       ├── Cargo.toml
│       └── src/lib.rs                  # ← Soroban smart contract
└── frontend/
    ├── .env.example
    ├── next.config.js
    ├── package.json
    ├── tsconfig.json
    └── src/
        ├── app/
        │   ├── globals.css             # Cyber-noir design system
        │   ├── layout.tsx
        │   └── page.tsx                # ← Main dashboard UI
        ├── hooks/
        │   └── use-freighter-wallet.ts # Freighter React hook
        └── lib/
            ├── config.ts               # Env var centralization
            ├── contract-client.ts      # Soroban contract bridge
            ├── format.ts               # Amount + address utils
            ├── freighter.ts            # Freighter wallet layer
            └── types.ts                # Shared TypeScript types
```

---

## Common Issues

| Issue | Fix |
|---|---|
| `simulateTransaction` fails | Fund `NEXT_PUBLIC_STELLAR_READ_ADDRESS` at friendbot.stellar.org |
| "Wrong network" badge | Switch Freighter to Testnet |
| `#1 Unauthorized` error | The connected wallet doesn't match the required signer |
| WASM build fails | Run `rustup target add wasm32-unknown-unknown` |
| Freighter not detected | Ensure extension is installed and this is a Chromium-based browser |

---

Built for **Stellar Bootcamp 2026** @ White Cloak Technologies, Ortigas, Pasig City 🇵🇭
