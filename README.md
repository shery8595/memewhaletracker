# 🐳 MemeWhaleTrackerAgent – ROMA Integration Guide

The **MemeWhaleTrackerAgent** tracks predefined Solana meme-wallets and reports the most-traded meme coins in the last 24 hours.  
This guide explains how to integrate and run it inside the **ROMA** framework.

---

## 📖 Table of Contents
1. [Step 1. Configure Environment](#-step-1-configure-environment)  
2. [Step 2. Create Wallets File](#-step-2-create-wallets-file)  
3. [Step 3. Add the Adapter](#-step-3-add-the-adapter)  
4. [Step 4. Register the Agent](#-step-4-register-the-agent)  
5. [Step 5. Update Deep Research Agent Profiles](#-step-5-update-deep-research-agent-profiles)
6. [Step 6. Compose Docker](#-step-6-Compose-Docker)
7. [Step 7. Notes](#-step-7-Notes)  

---

## 📌 Step 1. Configure Environment

Add the following to your `.env` file:

```env
HELIUS_API_KEY=your_helius_api_key_here
RPC_URL=https://api.mainnet-beta.solana.com
LOG_LEVEL=INFO
```

## 📌 Step 2. Create Wallets File

In ROMA/data/wallets.json, list the wallets you want to track:

``
[
  "WalletAddress1",
  "WalletAddress2",
  "WalletAddress3"
]
``
## 📌 Step 3. Add the Adapter

Create the file:
ROMA/src/adapters/meme_whale_tracker_adapter.py

``
import os
import json
import requests
import time
from datetime import datetime, timedelta

from core.schema import ExecutionOutput   # ✅ adjust path if needed

class MemeWhaleTrackerAdapter:
    """
    Adapter that loads a predefined wallet list, queries Helius for recent
    transactions, extracts token transfer events, and aggregates token counts
    for the last `since_hours` (default 24).
    """

    def __init__(self, api_key=None, wallets_file=None):
        self.api_key = api_key or os.getenv("HELIUS_API_KEY")
        if not self.api_key:
            raise ValueError("Helius API key required. Set HELIUS_API_KEY env var or pass api_key param.")

        self.base_url = "https://api.helius.xyz/v0"
        wallets_file = wallets_file or os.path.join(os.path.dirname(__file__), "wallets.json")

        if os.path.exists(wallets_file):
            with open(wallets_file, "r") as f:
                self.wallets = json.load(f)
        else:
            self.wallets = []

        self._delay_between_calls = 0.3   # throttle to avoid rate limits

    def _get_transactions_for_wallet(self, wallet):
        url = f"{self.base_url}/addresses/{wallet}/transactions"
        params = {"api-key": self.api_key}
        try:
            resp = requests.get(url, params=params, timeout=15)
        except Exception as e:
            print(f"[ERROR] request failed for {wallet}: {e}")
            return []

        if resp.status_code != 200:
            print(f"[ERROR] Helius returned {resp.status_code} for {wallet}: {resp.text}")
            return []

        try:
            return resp.json()
        except Exception as e:
            print(f"[ERROR] decoding JSON for {wallet}: {e}")
            return []

    def _tx_timestamp(self, tx):
        for key in ("timestamp", "blockTime", "block_time", "block_time_unix", "blockTimeUnix"):
            val = tx.get(key)
            if val:
                try:
                    ts = int(val)
                    if ts > 1_000_000_000_000:   # convert ms → seconds
                        ts //= 1000
                    return datetime.utcfromtimestamp(ts)
                except Exception:
                    pass
        return None

    def _extract_token_symbols_from_tx(self, tx):
        tokens = []
        for key in ("tokenTransfers", "token_transfers", "tokenTransfer", "tokenTransfersWithMeta"):
            if key in tx and isinstance(tx[key], list):
                for t in tx[key]:
                    sym = t.get("tokenSymbol") or t.get("symbol") or t.get("token_name") or t.get("tokenName")
                    if sym:
                        tokens.append(sym.upper())
                    else:
                        mint = t.get("mint") or t.get("tokenMint")
                        if mint:
                            tokens.append(mint)
        return tokens

    def execute(self, task_input=None, since_hours: int = 24, top_n: int = 10):
        cutoff = datetime.utcnow() - timedelta(hours=since_hours)
        trades_summary, wallets_involved = {}, {}

        for wallet in self.wallets:
            txs = self._get_transactions_for_wallet(wallet)
            time.sleep(self._delay_between_calls)

            if not isinstance(txs, list):
                continue

            for tx in txs:
                ts = self._tx_timestamp(tx)
                if ts is None or ts < cutoff:
                    continue

                token_symbols = self._extract_token_symbols_from_tx(tx)
                for sym in token_symbols:
                    trades_summary[sym] = trades_summary.get(sym, 0) + 1
                    wallets_involved.setdefault(sym, set()).add(wallet)

        sorted_trades = sorted(trades_summary.items(), key=lambda x: x[1], reverse=True)
        top_trades = sorted_trades[:top_n]

        data = {
            "summary": f"Top tokens traded by {len(self.wallets)} tracked wallets in last {since_hours} hours",
            "top_trades": [
                {"token": token, "trades": count, "wallets": list(wallets_involved.get(token, []))[:5]}
                for token, count in top_trades
            ]
        }

        return ExecutionOutput(success=True, data=data, metadata={"agent": "MemeWhaleTracker"})


if __name__ == "__main__":
    api = os.getenv("HELIUS_API_KEY")
    if not api:
        print("Please set HELIUS_API_KEY env var and ensure wallets.json is present.")
        exit(1)

    adapter = MemeWhaleTrackerAdapter(api_key=api)
    out = adapter.execute(since_hours=24, top_n=10)
    print(json.dumps(out.dict(), indent=2))
``
## 📌 Step 4. Register the Agent

In ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agents.yaml, add:

```
name: "MemeWhaleTrackerAgent"
type: "executor"
adapter_class: "adapters.meme_whale_tracker_adapter.MemeWhaleTrackerAdapter"
description: "Tracks predefined Solana meme-wallets and reports the most-traded meme coins in the last 24h"

model:
  provider: "litellm"
  model_id: "openrouter/x-ai/grok-4-fast:free"
  temperature: 0.35

params:
  wallets_file: "data/wallets.json"
  since_hours: 24

enabled: true
```

## 📌 Step 5. Update Deep Research Agent Profiles

Edit:
ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/profiles/deep_research_agent.yaml

```
Replace:

executor_adapter_names:
  SEARCH: "OpenAICustomSearcher"


with:

executor_adapter_names:
  SEARCH: "MemeWhaleTracker"

```

## 📌 Step 6. Compose Docker
```
docker compose down
docker compose up -d --build
```

## 📌 Step 7. Notes

If you get empty results, check:

Your wallets are active

since_hours covers the correct timeframe

Helius free tier has API rate limits → add retries or upgrade if needed.
