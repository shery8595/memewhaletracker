# 🐳 MemeWhaleTrackerAgent – ROMA Integration Guide

The **MemeWhaleTrackerAgent** tracks predefined Solana meme-wallets and reports the most-traded meme coins in the last 24 hours.  
This guide explains how to integrate and run it inside the **ROMA** framework.

---

## 📖 Table of Contents
1. [Step 1. Configure Environment](#-step-1-configure-environment)  
2. [Step 2. Create Wallets File](#-step-2-create-wallets-file)  
3. [Step 3. Add the Adapter](#-step-3-add-the-adapter)  
4. [Step 4. Register the Agent](#-step-4-register-the-agent)  
5. [Step 5. (Optional) Create Executor Script](#-step-5-optional-create-executor-script)  
6. [Step 6. Update Deep Research Agent Profiles](#-step-6-update-deep-research-agent-profiles)  
7. [Step 7. Run the Agent](#-step-7-run-the-agent)  
8. [Notes](#-notes)  

---

## 📌 Step 1. Configure Environment

Add the following to your `.env` file:

```env
HELIUS_API_KEY=your_helius_api_key_here
RPC_URL=https://api.mainnet-beta.solana.com
LOG_LEVEL=INFO

## 📌 Step 2. Create Wallets File

In ROMA/data/wallets.json, list the wallets you want to track:

