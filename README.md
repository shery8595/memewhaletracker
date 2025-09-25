sentient-roma-memewhaletracker

This is an experimental guide for integrating the MemeWhaleTrackerAgent into ROMA
.
The agent tracks predefined Solana wallets and reports the most-traded meme coins in the last 24 hours.

Step One. Add the Helius API key to .env:
HELIUS_API_KEY=your_helius_api_key_here


Optionally, you can also add:

RPC_URL=https://api.mainnet-beta.solana.com
LOG_LEVEL=INFO

Step Two. Create or update your wallets.json file in data/:
[
  "WalletAddress1",
  "WalletAddress2",
  "WalletAddress3"
]


This is the list of meme-wallets the agent will track.

Step Three. Add the MemeWhaleTrackerAgent to agents.yaml:
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

Step Four. Create the adapter in adapters/meme_whale_tracker_adapter.py:
import os
import json
import requests
from datetime import datetime, timedelta

class MemeWhaleTrackerAdapter:
    """
    Adapter that loads a predefined wallet list, queries Helius for each wallet's recent transactions,
    extracts token transfer events, aggregates token counts for the last `since_hours` (default 24).
    """

    def __init__(self, api_key=None, wallets_file=None, since_hours=24):
        self.api_key = api_key or os.getenv("HELIUS_API_KEY")
        self.wallets_file = wallets_file
        self.since_hours = since_hours

    def run(self):
        # load wallets
        with open(self.wallets_file, "r") as f:
            wallets = json.load(f)

        # TODO: implement Helius API calls, aggregate token transfers
        results = {"top_tokens": [], "time_window": f"{self.since_hours}h"}
        return results

Step Five. Disable conflicting or unused agents

Some core ROMA agents may conflict or be unnecessary. In agents.yaml, set them to enabled: false if you don’t want them running. For example:

- name: "OpenAICustomSearcher"
  type: "custom_search"
  adapter_class: "OpenAICustomSearchAdapter"
  description: "Direct API-based search using OpenAI"
  enabled: false

Step Six. Run the agent locally:
python -m roma_runtime.run_agent --agent MemeWhaleTrackerAgent


Expected output (example):

{
  "top_tokens": [
    {"symbol": "BONK", "count": 120},
    {"symbol": "WIF", "count": 95},
    {"symbol": "POPCAT", "count": 42}
  ],
  "time_window": "24h"
}

Step Seven. Run with Docker:
docker compose down
docker compose up -d --build
