# Tokenization Lego on Stellar + Trustless Work

**Open-source template** to build a tokenization stack with **Stellar/Soroban**, **Trustless Work** (escrows), participation tokens, token sales, ROI vaults, and **Next.js** apps for issuers, investors, and transparency.

---

## Monorepo structure

| App | Description |
|-----|-------------|
| `apps/backoffice-tokenization` | Issuer console: escrows, tokenization, milestones, disputes |
| `apps/investor-tokenization` | Investor portal: buy tokens, holdings, claim ROI |
| `apps/project-updates-tokenization` | Transparency portal: milestones and project status |
| `apps/evidence-service` | Off-chain evidence microservice (IPFS) |
| `apps/smart-contracts` | Soroban contracts: escrow, Token Factory, Token Sale, Vault |

---

## What this template demonstrates

1. **Escrow** — Multi-release, milestones, approvals, disputes, and fund release (Trustless Work + Soroban).
2. **Tokenization** — Token deploy → primary sale → funds to escrow → milestones → returns to vault → investors claim ROI by balance.
3. **Frontends** — Backoffice (manage), Investor (invest & claim), Project Updates (read-only).

---

## Contracts (in `apps/smart-contracts`)

- **Escrow** — Multi-release, roles, disputes, approvals.
- **Token Factory** — Mint/burn participation tokens.
- **Token Sale** — Sell tokens for USDC and route to escrow.
- **Vault** — Returns and ROI claim by token holdings.

Includes tests and JSON snapshots.

---

## How to run

From the repo root (all apps):

```bash
npm install
npm run dev
```

Or a single app:

```bash
cd apps/<backoffice-tokenization|investor-tokenization|project-updates-tokenization|evidence-service>
npm install
npm run dev
```

Each app has a `.env.example`; copy to `.env` or `.env.local` and fill in the variables (Soroban RPC, network, contract IDs, Trustless Work API, etc.).

---

## Tech stack

- **Next.js** (App Router), **Tailwind**, **ShadCN**
- **Trustless Work**: Wallet Kit, Blocks (escrow UI), API
- **Soroban** for contract calls

---

## Intended use

For teams experimenting with tokenization, hackathons, RWA, or integrating Trustless Work. Fork, modify, and build your product.

**License:** MIT.
