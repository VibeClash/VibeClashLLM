VibeClash — LIVE, Open Source, All‑in‑One Trading Agent
> Solana (Jupiter/Raydium) • Pump.fun (post‑migration) • BNB Chain (UniV2/Pancake) • Astar (UniV2) • Optional LLM policy

VibeClash is a modular trading agent designed for **real trading** (no paper mode). It provides a clean CLI, small but expressive strategy layer, exchange adapters for **Solana** and **EVM chains**, and an optional **LLM‑assisted policy**.

---




## TL;DR (LIVE)

```bash
npm i
cp .env.example .env
# Fill RPCs + keys (burners). MODE must be LIVE.

# BSC with LLM Strategy
export LLM_ENABLED=true && export OPENAI_API_KEY=sk-...
npm run dev -- run --network bsc --strategy llm --base WBNB --quote BUSD --amount 0.05

# Validate your LLM key quickly
npm run dev -- llm-test
```

---

## Highlights
- **Live‑only** execution with explicit key requirements
- **Pluggable LLM** decision policy (OpenAI / Anthropic / Deepseek)
- **Strategy engine** with tiny but realistic examples (SMA momentum; LLM policy)
- **DEX routing**: Jupiter on Solana, UniswapV2 routers on BSC/Astar (+ ERC20 approve path)
- **Clean TypeScript** repo, MIT license, Dockerfile

---

## Supported Networks

| Chain | Router/Aggregator | Swap Signing | Notes |
|------:|-------------------|--------------|-------||
| BNB Chain (BSC) | UniV2 / Pancake | EVM Private Key | Includes ERC20 allowance auto‑approve |
| Aster | UniV2 compatible | EVM Private Key | Router address configurable in `.env` |

> **Always verify** router addresses and keep slippage tight for new pools.

---

## Architecture (High‑Level)

```mermaid
flowchart LR
  subgraph CLI
    C1[VibeClash run] --> C2[params]
  end

  subgraph Core
    E[Engine] -->|invoke| S[Strategy]
    S -->|needs quote| X
    S -->|may trade| X
  end

  subgraph Exchanges
    X{Exchange Adapter} -->|solana| SO[SolanaExchange]
    X -->|evm| EV[EvmExchange]
  end

  subgraph Infra
    J[Jupiter REST]:::ext
    U[UniV2 Routers]:::ext
    RPCS[(RPC Providers)]:::ext
  end

  SO -->|quote/swap| J
  EV -->|getAmountsOut/swap| U
  SO & EV --> RPCS

  classDef ext fill:#222,stroke:#555,color:#eee;
```

---

## Trading Loop (Sequence)

```mermaid
sequenceDiagram
  participant CLI as CLI
  participant Eng as Engine
  participant Strat as Strategy
  participant Ex as Exchange
  participant DEX as DEX/Agg

  CLI->>Eng: run(...)
  Eng->>Strat: runOnce()
  Strat->>Ex: getQuote(pair, amount, slippageBps)
  Ex->>DEX: fetch quote/route
  DEX-->>Ex: route + price
  Ex-->>Strat: Quote(price, route)
  alt Buy Signal
    Strat->>Ex: swap(params, route)
    Ex->>DEX: submit tx
    DEX-->>Ex: txid
    Ex-->>Strat: TradeResult(txid, filledPrice)
  else Hold
    Strat-->>Eng: no‑op
  end
  Eng-->>CLI: iteration complete
```

---

## EVM Swap Path (Allowance + Swap)

```mermaid
sequenceDiagram
  participant Bot as VibeClash(EVM)
  participant ERC20 as Base Token
  participant Router as UniV2 Router

  Bot->>ERC20: allowance(owner, router)
  alt allowance < amountIn
    Bot->>ERC20: approve(router, amountIn)
    ERC20-->>Bot: Approval Tx Receipt
  end
  Bot->>Router: swapExactTokensForTokens(amountIn, minOut, path, to, deadline)
  Router-->>Bot: Swap Tx Receipt
```

---

## LLM Policy – Decision Graph

```mermaid
flowchart TD
  A[Fetch Quote & Context] --> B[Prompt LLM (BUY or HOLD)]
  B -->|BUY| C[Execute Swap (minOut w/ slippage)]
  B -->|HOLD| D[Skip Trade]
  C --> E[Log txid + price]
  D --> E[Log snapshot]
```

### Example System/User Prompt (built‑in)
```
System: "You are a cautious trading micro‑policy. Only answer with BUY or HOLD."
User: 
  Pair: SOL/USDC
  Price: 123.45678901
  Instruction: Reply with BUY if a small buy is reasonable now; otherwise reply with HOLD.
```

Enable with:
```env
LLM_ENABLED=true
LLM_PROVIDER=openai  # or anthropic | deepseek
OPENAI_API_KEY=...
```

---

## Strategy Interface (Build Your Own)

```ts
export interface IExchange {
  getQuote(p: TradeParams): Promise<Quote>;
  swap(p: TradeParams, route?: any): Promise<TradeResult>;
}

export interface TradeParams {
  base: TokenInfo;  // symbol/address/decimals
  quote: TokenInfo;
  amount: number;   // base amount
  slippageBps: number;
}
```

**Add a new strategy**
1. Create `src/strategies/myStrat.ts` that exposes `runOnce()`
2. Import & route it in `src/core/engine.ts`
3. Invoke via `--strategy myStrat`

---

## Config Graph (Where settings come from)

```mermaid
graph LR
  A[.env] --> B[Zod Parser]
  B --> C[env object]
  C --> D[SolanaExchange]
  C --> E[EvmExchange]
  C --> F[Engine/Strategy]
```

Key variables (`.env.example` includes all):
- `MODE=LIVE` (required)
- Solana: `SOLANA_RPC`, `SOLANA_KEYPAIR`, `JUPITER_BASE_URL`
- EVM: `BSC_RPC`, `ASTAR_RPC`, `EVM_PRIVATE_KEY`, `*_UNISWAPV2_ROUTER`
- LLM: `LLM_ENABLED`, `LLM_PROVIDER`, `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `DEEPSEEK_API_KEY`

---

## Risk Controls (Simple Defaults)

```mermaid
pie title Slippage Budget
  "Expected price" : 9900
  "Slippage (bps)" : 100
```
- Default `SLIPPAGE_BPS=100` (1%); tune per pair liquidity.
- Prefer **tiny sizes** for new pools; scale only with observed depth.
- Use **burner wallets** and separate machines for ops.

---

## Example PnL Snapshot (Concept)
> Hook this to a candles API & your trade logs.

```mermaid
gantt
dateFormat  YYYY-MM-DD
title  Weekly PnL Milestones (Example)
section PnL
+1%        :done,    des1, 2025-10-01, 1d
-0.3%      :active,  des2, 2025-10-02, 1d
+0.7%      :         des3, 2025-10-03, 1d
+0.2%      :         des4, 2025-10-04, 1d
-0.1%      :         des5, 2025-10-05, 1d
```

---

## Docker

```bash
docker build -t VibeClash .
docker run --rm -it --env-file .env VibeClash --help
```

---

## Development Checklist
- [ ] Fill `.env` with **LIVE** RPCs + keys (burners)
- [ ] Verify router addresses
- [ ] Start with small `--amount`
- [ ] Set `SLIPPAGE_BPS` conservatively
- [ ] (Optional) Turn on LLM policy and validate with `llm-test`

---

## FAQ

**Q: Does VibeClash support Pump.fun tokens?**  
A: Yes, **after migration** to Raydium (normal SPL pools). The watcher shows demo logs; wire a real feed to auto‑trigger strategies.

**Q: Why LIVE‑only?**  
A: This repo targets production‑like flows with explicit key handling and real tx paths. If you want paper trading, fork the earlier commit and re‑enable the simulators.

**Q: Can I add new chains?**  
A: Yes—add an adapter under `src/exchanges/`, implement `getQuote/swap`, extend token registries, and register in the CLI.

---

## Legal
MIT licensed. No warranties. You are responsible for your own keys, compliance, and risk management.
