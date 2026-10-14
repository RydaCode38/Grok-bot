What it does:It acts as an automated multi-agent system that continuously monitors the live cloud GPU market (especially H100, H200, A100, RTX 4090) and helps you rent cheap GPUs to run AI jobs overnight — while keeping full human control.Core Features:Live data: Pulls real-time GPU rental prices every ~75 seconds from public sources (priceofcompute.com + gpufinder.dev)
Multi-agent architecture (same as the original concept):SCOUT → scans the market
QUANT → ranks the best offers by price
RISK → filters bad deals / budget checks
EXEC → prepares the rental proposal
YOU → final human approval required (no automatic spending)

Dark, modern dashboard UI with live offer list, profit tracker, budget remaining, jobs completed, and a real-time log console
Simulation of rental + revenue when you click “Approve”
One-click buttons to open Vast.ai and RunPod
Fully portable single .exe file (no Python needed after building)

Important notes:It does not automatically spend money or rent GPUs by itself.
It currently uses public pricing APIs only.
Real automatic renting (via Vast.ai / RunPod API keys) can be added later if desired.

In short: It’s a working, live-data version of the “Grok Bot” night-shift GPU arbitrage desk from the viral 

