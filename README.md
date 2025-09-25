This is a professional guide for integrating the MemeWhaleTrackerAgent into ROMA
.
The agent tracks predefined Solana meme-wallets and reports the most-traded meme coins in the last 24 hours.

Step One. Add API Keys to .env

Open your .env file and add:

HELIUS_API_KEY=your_helius_api_key_here
RPC_URL=https://api.mainnet-beta.solana.com
LOG_LEVEL=INFO


This enables on-chain transaction queries through the Helius API
.

Step Two. Create the tracked wallets file

In ROMA/data/wallets.json, add the wallets you want to track:

[
  "WalletAddress1",
  "WalletAddress2",
  "WalletAddress3"
]

Step Three. Create the adapter

In ROMA/src/adapters/meme_whale_tracker_adapter.py:

import os
import json
import requests
from datetime import datetime, timedelta
import time

# ✅ import ExecutionOutput from your repo's schema
# adjust path if needed (e.g. `from core.schema import ExecutionOutput`)
from core.schema import ExecutionOutput


class MemeWhaleTrackerAdapter:
    """
    Adapter that loads a predefined wallet list, queries Helius for each wallet's recent transactions,
    extracts token transfer events, aggregates token counts for the last `since_hours` (default 24).
    """

    def __init__(self, api_key=None, wallets_file=None):
        self.api_key = api_key or os.getenv("HELIUS_API_KEY")
        if not self.api_key:
            raise ValueError("Helius API key required. Set HELIUS_API_KEY env var or pass api_key param.")

        self.base_url = "https://api.helius.xyz/v0"
        # wallets_file default: same folder
        if not wallets_file:
            wallets_file = os.path.join(os.path.dirname(__file__), "wallets.json")

        if os.path.exists(wallets_file):
            with open(wallets_file, "r") as f:
                self.wallets = json.load(f)
        else:
            self.wallets = []

        # small throttle to avoid hitting rate limits
        self._delay_between_calls = 0.3

    def _get_transactions_for_wallet(self, wallet):
        """
        Uses Helius address transactions endpoint.
        Returns list of transaction dicts (best effort).
        """
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
        """
        Try common timestamp fields and return a datetime (UTC).
        Helius may use 'timestamp' or 'blockTime' (seconds) or ms — this function handles both.
        """
        for key in ("timestamp", "blockTime", "block_time", "block_time_unix", "blockTimeUnix"):
            val = tx.get(key)
            if val:
                try:
                    ts = int(val)
                    # if it looks like milliseconds, convert
                    if ts > 1_000_000_000_000:
                        ts = ts // 1000
                    return datetime.utcfromtimestamp(ts)
                except Exception:
                    pass
        # fallback: tx may have 'slot' only -> return None
        return None

    def _extract_token_symbols_from_tx(self, tx):
        """
        Attempt to find token transfers inside a tx dict. Different Helius outputs vary.
        Returns a list of token symbol strings (uppercase).
        """
        tokens = []
        for key in ("tokenTransfers", "token_transfers", "tokenTransfer", "tokenTransfersWithMeta"):
            if key in tx and isinstance(tx[key], list):
                for t in tx[key]:
                    sym = (
                        t.get("tokenSymbol")
                        or t.get("symbol")
                        or t.get("token_name")
                        or t.get("tokenName")
                    )
                    if sym:
                        tokens.append(sym.upper())
                    else:
                        mint = t.get("mint") or t.get("tokenMint")
                        if mint:
                            tokens.append(mint)
        return tokens

    def execute(self, task_input=None, since_hours: int = 24, top_n: int = 10):
        """
        Main entry used by ROMA. Returns an ExecutionOutput object.
        task_input is unused because wallets are preconfigured.
        """
        cutoff = datetime.utcnow() - timedelta(hours=since_hours)
        trades_summary = {}   # token_symbol -> count
        wallets_involved = {}  # token -> set(wallets)

        for i, wallet in enumerate(self.wallets):
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
                {
                    "token": token,
                    "trades": count,
                    "wallets": list(wallets_involved.get(token, []))[:5]
                }
                for token, count in top_trades
            ]
        }

        # ✅ Wrap result in ExecutionOutput so ROMA won’t break
        return ExecutionOutput(
            success=True,
            data=data,
            metadata={"agent": "MemeWhaleTracker"}
        )


# Quick command-line test: run this file directly (from repo root)
if __name__ == "__main__":
    api = os.getenv("HELIUS_API_KEY")
    if not api:
        print("Please set HELIUS_API_KEY env var and ensure wallets.json is present in the adapter folder.")
        exit(1)

    adapter = MemeWhaleTrackerAdapter(api_key=api)
    out = adapter.execute(since_hours=24, top_n=10)
    print(json.dumps(out.dict(), indent=2))  # ✅ use .dict() since it's ExecutionOutput



This file defines the adapter class that powers the MemeWhaleTrackerAgent.

Step Four. Register the agent in agents.yaml

In ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agents.yaml, add:

- name: "MemeWhaleTrackerAgent"
  type: "executor"
  adapter_class: "adapters.meme_whale_tracker_adapter.MemeWhaleTrackerAdapter"
  description: "Tracks predefined Solana meme-wallets and reports the most-traded meme coins in the last 24h"
  model:
    provider: "litellm"
    model_id: "openrouter/google/gemini-2.0-flash-exp:free"
    temperature: 0.35
  params:
    wallets_file: "data/wallets.json"
    since_hours: 24
  enabled: true


This integrates your new agent into the ROMA runtime.

Step Five. Create the executor script (optional)

If you want a direct entrypoint, create ROMA/src/meme_whale_tracker.py:

from adapters.meme_whale_tracker_adapter import MemeWhaleTrackerAdapter

if __name__ == "__main__":
    adapter = MemeWhaleTrackerAdapter(wallets_file="data/wallets.json", since_hours=24)
    results = adapter.run()
    print(results)


This allows you to run the agent directly without going through the full ROMA runtime.

Step Six. Update Deep Research Agent Profiles

The DeepResearchAgent profile loads default searchers (OpenAI, Gemini, Exa) that may conflict with your environment.
To avoid errors, disable them and point the research profile away from incompatible agents.

In:

ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/profiles/deep_research_agent.yaml


Find:

executor_adapter_names:
  SEARCH: "OpenAICustomSearcher"


Replace with:

executor_adapter_names:
  SEARCH: "SmartWebSearcher"


Also, disable conflicting agents in agents.yaml by setting:

- name: "OpenAICustomSearcher"
  enabled: false

- name: "GeminiCustomSearcher"
  enabled: false

- name: "ExaComprehensiveSearcher"
  enabled: false


This ensures that only the MemeWhaleTrackerAgent and compatible executors run.

Step Seven. Run the agent
Run directly with ROMA runtime:
python -m roma_runtime.run_agent --agent MemeWhaleTrackerAgent


Expected output (example):

{
  "top_tokens": [
    {"symbol": "BONK", "count": 120},
    {"symbol": "WIF", "count": 95},
    {"symbol": "POPCAT", "count": 42}
  ],
  "time_window": "last 24h"
}

Run in Docker:
docker compose down
docker compose up -d --build

Notes

If you get empty results, check that your wallets are active and since_hours covers the right timeframe.

Helius free tier has API rate limits. Use retries or upgrade if needed.

Disabling default searchers in deep_research_agent.yaml is important, otherwise some profiles will break.
