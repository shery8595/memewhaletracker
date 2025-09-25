MemeWhaleTrackerAgent Integration Guide
A professional guide for integrating the MemeWhaleTrackerAgent into the ROMA framework. This agent tracks predefined Solana meme wallets and reports the most-traded meme coins within a specified time window.

📋 Prerequisites
ROMA framework installed

Helius API account (sign up here)

Basic understanding of Solana wallet addresses

🚀 Installation & Configuration
Step 1: Add API Keys to Environment
Open your .env file and add the following configuration:

env
HELIUS_API_KEY=your_helius_api_key_here
RPC_URL=https://api.mainnet-beta.solana.com
LOG_LEVEL=INFO
Note: The RPC URL above uses Solana mainnet. Replace with devnet if testing.

Step 2: Configure Tracked Wallets
Create the wallet configuration file at ROMA/data/wallets.json:

json
[
  "WalletAddress1",
  "WalletAddress2", 
  "WalletAddress3"
]
Tip: Replace with actual Solana wallet addresses you wish to monitor.

🔧 Adapter Implementation
Step 3: Create the MemeWhaleTracker Adapter
Create ROMA/src/adapters/meme_whale_tracker_adapter.py:

python
import os
import json
import requests
from datetime import datetime, timedelta
import time

# Adjust import path according to your project structure
from core.schema import ExecutionOutput

class MemeWhaleTrackerAdapter:
    """
    Adapter that loads a predefined wallet list, queries Helius for each wallet's 
    recent transactions, extracts token transfer events, and aggregates token counts.
    """
    
    def __init__(self, api_key=None, wallets_file=None):
        self.api_key = api_key or os.getenv("HELIUS_API_KEY")
        if not self.api_key:
            raise ValueError("Helius API key required. Set HELIUS_API_KEY env var or pass api_key param.")

        self.base_url = "https://api.helius.xyz/v0"
        
        # Set default wallets file path
        if not wallets_file:
            wallets_file = os.path.join(os.path.dirname(__file__), "wallets.json")

        if os.path.exists(wallets_file):
            with open(wallets_file, "r") as f:
                self.wallets = json.load(f)
        else:
            self.wallets = []

        # Rate limiting delay between API calls
        self._delay_between_calls = 0.3

    def _get_transactions_for_wallet(self, wallet):
        """Fetch transactions for a specific wallet using Helius API."""
        url = f"{self.base_url}/addresses/{wallet}/transactions"
        params = {"api-key": self.api_key}
        
        try:
            resp = requests.get(url, params=params, timeout=15)
        except Exception as e:
            print(f"[ERROR] Request failed for {wallet}: {e}")
            return []

        if resp.status_code != 200:
            print(f"[ERROR] Helius returned {resp.status_code} for {wallet}: {resp.text}")
            return []

        try:
            return resp.json()
        except Exception as e:
            print(f"[ERROR] JSON decoding failed for {wallet}: {e}")
            return []

    def _tx_timestamp(self, tx):
        """Extract timestamp from transaction data."""
        for key in ("timestamp", "blockTime", "block_time", "block_time_unix", "blockTimeUnix"):
            val = tx.get(key)
            if val:
                try:
                    ts = int(val)
                    # Convert milliseconds to seconds if necessary
                    if ts > 1_000_000_000_000:
                        ts = ts // 1000
                    return datetime.utcfromtimestamp(ts)
                except Exception:
                    pass
        return None

    def _extract_token_symbols_from_tx(self, tx):
        """Extract token symbols from transaction data."""
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
        """Main execution method called by ROMA framework."""
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

        # Sort and get top results
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

        return ExecutionOutput(
            success=True,
            data=data,
            metadata={"agent": "MemeWhaleTracker"}
        )


if __name__ == "__main__":
    # Command-line test functionality
    api = os.getenv("HELIUS_API_KEY")
    if not api:
        print("Please set HELIUS_API_KEY env var and ensure wallets.json is present.")
        exit(1)

    adapter = MemeWhaleTrackerAdapter(api_key=api)
    out = adapter.execute(since_hours=24, top_n=10)
    print(json.dumps(out.dict(), indent=2))
⚙️ Agent Registration
Step 4: Register Agent in Configuration
Add the following entry to ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agents.yaml:

yaml
name: "MemeWhaleTrackerAgent"
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
🔄 Framework Integration
Step 5: Update Deep Research Agent Profiles
Modify ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/profiles/deep_research_agent.yaml:

Replace:

yaml
executor_adapter_names:
  SEARCH: "OpenAICustomSearcher"
With:

yaml
executor_adapter_names:
  SEARCH: "SmartWebSearcher"
Step 6: Disable Conflicting Agents
In agents.yaml, disable incompatible searchers:

yaml
name: "OpenAICustomSearcher"
enabled: false

name: "GeminiCustomSearcher" 
enabled: false

name: "ExaComprehensiveSearcher"
enabled: false
🧪 Testing & Execution
Step 7: Run the Agent
Option A: Direct ROMA Execution

bash
python -m roma_runtime.run_agent --agent MemeWhaleTrackerAgent
Option B: Docker Deployment

bash
docker compose down
docker compose up -d --build
Expected Output Example:
json
{
  "top_tokens": [
    {"symbol": "BONK", "count": 120},
    {"symbol": "WIF", "count": 95}, 
    {"symbol": "POPCAT", "count": 42}
  ],
  "time_window": "last 24h"
}
🛠️ Troubleshooting
Common Issues & Solutions
Issue	Solution
Empty results	Verify wallet addresses are active and since_hours covers recent activity
API rate limits	Upgrade Helius plan or implement retry logic with exponential backoff
Import errors	Check Python path and ensure ExecutionOutput import is correct
Transaction parsing fails	Verify Helius API response format hasn't changed
Debug Mode
Enable debug logging by setting LOG_LEVEL=DEBUG in your .env file.

📝 Notes
Helius Free Tier: Be mindful of API rate limits (typically 100k requests/month)

Wallet Activity: Ensure tracked wallets have recent transaction activity

Time Zones: All timestamps are processed in UTC

Error Handling: The adapter includes comprehensive error handling for network issues
