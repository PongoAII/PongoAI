<div align="center">

<pre>
 _____                                _____ 
|  __ \                         /\   |_   _|
| |__) |__  _ __   __ _  ___  /  \    | |  
|  ___/ _ \| '_ \ / _` |/ _ \/ /\ \   | |  
| |  | (_) | | | | (_| | (_) / ____ \ _| |_ 
|_|   \___/|_| |_|\__, |\___/_/    \_\_____|
                   __/ |                    
                  |___/                     
</pre>

**A LangGraph multi-agent orchestrator for managing on-chain assets on Robinhood Chain.**

Monitors positions on [long.xyz](https://app.long.xyz/), tracks dividend streams on [theindex.finance](https://theindex.finance/), and coordinates rebalancing — with human-in-the-loop approval for every transaction.

[![Python](https://img.shields.io/badge/python-3.11%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![LangGraph](https://img.shields.io/badge/agents-LangGraph-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![web3.py](https://img.shields.io/badge/web3.py-7.x-F16822)](https://web3py.readthedocs.io/)
[![Robinhood Chain](https://img.shields.io/badge/chain-Robinhood%20L2-00C805)](https://robinhood.com/us/en/chain/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](#license)
[![Status](https://img.shields.io/badge/status-scaffold-orange)](#roadmap--status)

[Architecture](#architecture) · [Stack](#stack) · [LLM Providers](#llm-providers) · [Install](#project-installation) · [Usage](#cli-usage) · [Security](#security)

</div>

---

## Overview

Agents propose. You approve. A separate executor signs.

The orchestrator runs four specialist agents wired together as a LangGraph `StateGraph`. Each agent is a ReAct loop with its own tools, and each is a node in the graph — order is explicit, state is captured by the checkpointer, and every step is inspectable. Agents read on-chain state, apply a policy you define in YAML, and enqueue concrete transaction proposals into a local SQLite queue. Nothing reaches the chain without your explicit approval — and even then, transactions pass through a set of hard guardrails enforced outside any LLM context.

**Bring your own LLM** — run with Anthropic Claude, OpenAI GPT, any model on OpenRouter, or fully local via Ollama. No vendor lock-in; swap providers with one env var.

> **Note** — This is a scaffold. The LangGraph wiring, proposal queue, executor, guardrails, and CLI are functional end-to-end. The tool implementations that read specific contract state are stubs pending real ABIs. See [Roadmap](#roadmap--status).

## Table of contents

<table>
<tr>
<td valign="top" width="33%">

**Getting started**
- [Architecture](#architecture)
- [Stack](#stack)
- [LLM Providers](#llm-providers)
- [Prerequisites](#prerequisites)
- [Installation](#project-installation)

</td>
<td valign="top" width="33%">

**Using it**
- [Configuration](#configuration)
- [CLI usage](#cli-usage)
- [End-to-end flow](#end-to-end-flow)
- [Tests](#tests)

</td>
<td valign="top" width="33%">

**Operations**
- [Security](#security)
- [Contracts](#contracts)
- [Roadmap / status](#roadmap--status)
- [Contributing](#contributing)
- [License](#license)

</td>
</tr>
</table>

---

## Architecture

```mermaid
flowchart TD
    User([You · CLI])
    Orchestrator[LangGraph StateGraph]

    Long[Long Agent<br/>positions on long.xyz]
    Index[Index Agent<br/>dividends on theindex.finance]
    Rebal[Rebalancer Agent<br/>allocation drift]
    Reports[Reports Agent<br/>read-only P&amp;L]

    Queue[(Proposal Queue<br/>SQLite)]
    Approval{Human approval}
    Guardrails{{Hard guardrails}}
    Signer[Signer · isolated key]
    Chain([Robinhood Chain])

    User --> Orchestrator
    Orchestrator --> Long & Index & Rebal & Reports
    Long --> Queue
    Index --> Queue
    Rebal --> Queue
    Queue --> Approval
    Approval -->|approved| Guardrails
    Guardrails -->|pass| Signer
    Signer --> Chain

    classDef agent fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef control fill:#FAEEDA,stroke:#BA7517,color:#412402
    classDef exec fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    class Long,Index,Rebal,Reports agent
    class Queue,Approval control
    class Guardrails,Signer exec
```

**Design principles**

<table>
<tr>
<td width="50%" valign="top">

**Isolated signing**<br/>
Agents never see the private key. Only the executor has access — ideally through a Safe, KMS, or hardware wallet, not a plaintext env var.

</td>
<td width="50%" valign="top">

**Simulate before enqueue**<br/>
Every proposal runs through `eth_call` at submit time. If the transaction would revert, it never reaches the queue — the human never sees a broken proposal.

</td>
</tr>
<tr>
<td width="50%" valign="top">

**Guardrails outside the LLM**<br/>
Contract whitelist verified by address (not just symbol), mandatory notional/slippage for value-moving proposals, gas units ceiling, daily tx limit. All enforced in `executor/guardrails.py`.

</td>
<td width="50%" valign="top">

**Read-only is read-only**<br/>
The Reports agent has no access to `submit_proposal`. Its capability set makes it structurally incapable of moving funds.

</td>
</tr>
</table>

---

## Stack

<table>
<thead>
<tr><th>Layer</th><th>Technology</th><th>Why</th></tr>
</thead>
<tbody>
<tr><td>Language</td><td><a href="https://www.python.org/"><b>Python 3.11+</b></a></td><td>Mature ecosystem for Web3 and agent frameworks</td></tr>
<tr><td>Agents</td><td><a href="https://langchain-ai.github.io/langgraph/"><b>LangGraph</b></a> · <a href="https://python.langchain.com/"><b>LangChain Core</b></a></td><td>Explicit StateGraph, ReAct loops, checkpoints, replayable steps</td></tr>
<tr><td>LLM</td><td>Anthropic · OpenAI · OpenRouter · Ollama · Local</td><td>Via <code>LLM_PROVIDER</code> env var; swap with zero code changes</td></tr>
<tr><td>Blockchain</td><td><a href="https://web3py.readthedocs.io/"><b>web3.py 7.x</b></a> · <b>eth-account</b></td><td>Standard EVM client, EIP-1559 support, compatible with Robinhood Chain (Arbitrum L2)</td></tr>
<tr><td>CLI</td><td><a href="https://click.palletsprojects.com/"><b>Click</b></a> · <a href="https://rich.readthedocs.io/"><b>Rich</b></a></td><td>Animated boot sequence, colored output, tabular proposal display</td></tr>
<tr><td>Persistence</td><td><a href="https://sqlmodel.tiangolo.com/"><b>SQLModel</b></a> (SQLite)</td><td>Proposal queue and local audit log</td></tr>
<tr><td>Config</td><td><b>Pydantic v2</b> · <b>PyYAML</b> · <b>python-dotenv</b></td><td>Strong validation, YAML config, secrets in <code>.env</code></td></tr>
<tr><td>Scheduling</td><td><a href="https://apscheduler.readthedocs.io/"><b>APScheduler</b></a></td><td>Daemon mode for periodic scans</td></tr>
<tr><td>Concurrency</td><td><b>filelock</b></td><td>Cross-process nonce lock prevents parallel-execution corruption</td></tr>
<tr><td>Tests</td><td><b>pytest</b></td><td>11 security-focused tests covering guardrails and preflight checks</td></tr>
<tr><td>Quality</td><td><b>ruff</b> · <b>mypy</b></td><td>Optional lint and type-check</td></tr>
</tbody>
</table>

---

## LLM Providers

PongoAI supports five LLM backends. Set `LLM_PROVIDER` and the matching config in `.env`. No code changes needed.

<table>
<thead>
<tr><th>Provider</th><th>Config</th><th>Cost</th><th>Best for</th></tr>
</thead>
<tbody>
<tr>
<td><b>Anthropic</b></td>
<td><code>LLM_PROVIDER=anthropic</code><br/><code>LLM_MODEL=claude-sonnet-4-6</code><br/><code>ANTHROPIC_API_KEY=sk-ant-...</code></td>
<td>Paid</td>
<td>Best agent reasoning, recommended default</td>
</tr>
<tr>
<td><b>OpenAI</b></td>
<td><code>LLM_PROVIDER=openai</code><br/><code>LLM_MODEL=gpt-4o</code><br/><code>OPENAI_API_KEY=sk-...</code></td>
<td>Paid</td>
<td>Strong alternative, widely used tooling</td>
</tr>
<tr>
<td><b>OpenRouter</b></td>
<td><code>LLM_PROVIDER=openrouter</code><br/><code>LLM_MODEL=anthropic/claude-sonnet-4-20250514</code><br/><code>OPENROUTER_API_KEY=sk-or-...</code></td>
<td>Free tier available</td>
<td>Access to any model (Claude, Gemini, Llama, etc.) via one API</td>
</tr>
<tr>
<td><b>Ollama</b></td>
<td><code>LLM_PROVIDER=ollama</code><br/><code>LLM_MODEL=llama3.1</code></td>
<td>Free (local)</td>
<td>Privacy, no API dependency, offline operation</td>
</tr>
<tr>
<td><b>Local</b></td>
<td><code>LLM_PROVIDER=local</code><br/><code>LLM_MODEL=your-model</code><br/><code>LOCAL_LLM_BASE_URL=http://localhost:8080/v1</code></td>
<td>Free (local)</td>
<td>vLLM, llama.cpp, LM Studio, text-generation-webui, etc.</td>
</tr>
</tbody>
</table>

<details>
<summary><b>Setting up Ollama (local, free)</b></summary>

1. Install Ollama from [ollama.com](https://ollama.com)
2. Pull a model:
   ```bash
   ollama pull llama3.1        # general purpose, 8B
   ollama pull qwen2.5-coder   # code-focused, 7B
   ```
3. Set in `.env`:
   ```
   LLM_PROVIDER=ollama
   LLM_MODEL=llama3.1
   ```
4. Run `pongo scan` — agents will use your local model.

Note: smaller models (7B–8B) may struggle with complex tool-calling. For production use, 70B+ or a cloud provider is recommended.

</details>

---

## Prerequisites

Every OS needs the same trio: **Python 3.11+**, **Git**, **pip**. Pick your platform below.

<details>
<summary><b>Linux</b> — Ubuntu · Debian · Fedora · Arch</summary>

**Ubuntu / Debian**
```bash
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip git build-essential
```

**Fedora**
```bash
sudo dnf install -y python3.11 python3-pip git gcc
```

**Arch**
```bash
sudo pacman -S python python-pip git base-devel
```

**Verify**
```bash
python3.11 --version   # Python 3.11.x or higher
git --version
```

</details>

<details>
<summary><b>macOS</b> — Intel · Apple Silicon</summary>

Requires [Homebrew](https://brew.sh/). If you don't have it yet:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install:
```bash
brew install python@3.11 git
```

Verify:
```bash
python3.11 --version
git --version
```

On Apple Silicon (M1/M2/M3), everything runs native on ARM — no Rosetta required.

</details>

<details>
<summary><b>Windows 10 / 11</b> — WSL2 (recommended) · Native</summary>

Two paths. WSL2 is strongly recommended for the Web3 stack.

**Option A — WSL2 (recommended)**

1. Open PowerShell as Administrator:
   ```powershell
   wsl --install -d Ubuntu
   ```
2. Reboot when prompted.
3. In the Ubuntu terminal that launches, follow the **Linux** section above.

**Option B — Native Windows**

1. Install Python 3.11+ from [python.org/downloads](https://www.python.org/downloads/). During install, **check "Add Python to PATH"**.
2. Install [Git for Windows](https://git-scm.com/download/win).
3. Install [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) — some dependencies compile native code.
4. Verify in PowerShell:
   ```powershell
   python --version
   git --version
   ```

> **Tip** — If PowerShell blocks venv activation later, run once:
> `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`

</details>

---

## Project installation

<details open>
<summary><b>Linux · macOS · WSL</b></summary>

```bash
git clone https://github.com/PongoAII/PongoAI.git
cd PongoAI
python3.11 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -e .
```

</details>

<details>
<summary><b>Windows PowerShell</b></summary>

```powershell
git clone https://github.com/PongoAII/PongoAI.git
cd PongoAI
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install -e .
```

</details>

Verify the CLI:

```console
$ pongo
 _____
|  __ \                         /\   |_   _|
| |__) |__  _ __   __ _  ___  /  \    | |
  ...
  [ OK ] Loading policy (config/policy.yaml)
  [ OK ] Loading contracts (config/contracts.yaml)
  [ OK ] Connecting LLM (anthropic / claude-sonnet-4-6)
  [ OK ] Wiring agents (LangGraph StateGraph)
  [ OK ] Arming guardrails (whitelist + notional + gas + slippage)
  [ OK ] Initializing signer (isolated key module)

               ~ ~
               |=|
           ___/   \___
     .----' _________ '----.
   | |  | O   | | {(@)} |  | |    ← cyber eye lights up
   | |  |     | |   *   |  | |

  PongoAI is ready.
```

The boot sequence is animated: spinner on each step, mascot eye powers on from off → dim → lit, chimney smoke rises.

For development (tests + lint):

```bash
pip install -e ".[dev]"
```

---

## Configuration

Three files to fill in: `.env` (secrets + LLM provider), `config/policy.yaml` (strategy), `config/contracts.yaml` (addresses).

### 1. Environment variables

```bash
cp .env.example .env       # Linux / macOS / WSL
```
```powershell
Copy-Item .env.example .env   # Windows
```

Minimum viable `.env`:

| Variable | Purpose | Example |
|---|---|---|
| `LLM_PROVIDER` | Which LLM backend | `anthropic` / `openai` / `openrouter` / `ollama` / `local` |
| `LLM_MODEL` | Model name | `claude-sonnet-4-6` / `gpt-4o` / `llama3.1` |
| `ANTHROPIC_API_KEY` | (if provider = anthropic) | `sk-ant-...` |
| `OPENROUTER_API_KEY` | (if provider = openrouter) | `sk-or-...` |
| `RPC_URL` | Chain endpoint | `https://rpc.testnet.chain.robinhood.com` |
| `CHAIN_ID` | From block explorer | `46630` (testnet) / `4663` (mainnet) |
| `SIGNER_MODE` | `local` · `safe` · `kms` | `local` |
| `PRIVATE_KEY` | **Testnet only** | `0x...` |

> **Warning** — For company treasury on mainnet, use `SIGNER_MODE=safe` with a Gnosis Safe. Never store a mainnet private key in a plaintext env file. The executor will refuse to run `SIGNER_MODE=local` against a non-testnet RPC unless explicitly overridden.

### 2. Policy — `config/policy.yaml`

Defines target allocation, thresholds, and hard guardrails.

| Field | Meaning |
|---|---|
| `allocation.target` | Desired split (%) between long.xyz and theindex.finance |
| `allocation.rebalance_threshold_pct` | Minimum drift before the Rebalancer proposes a swap |
| `long_xyz.watch` | Tokens to monitor (default: NVDA, TSLA, AAPL, SPY, SPCX, GOOGL) |
| `long_xyz.max_position_usd` | Cap per individual position |
| `theindex_finance.claim_min_usd` | Minimum accrued value to trigger a claim proposal |
| `guardrails.max_notional_usd_per_tx` | Hard ceiling per transaction |
| `guardrails.max_slippage_bps` | Maximum slippage (100 = 1%) |
| `guardrails.max_gas_units_per_tx` | Caps unusually expensive calldata |
| `guardrails.allow_mainnet_local_signer` | Safety override (default: false) |

### 3. Contracts — `config/contracts.yaml`

Ships with **real mainnet addresses** for theindex.finance contracts and long.xyz stock tokens. Populated from official docs.

<details>
<summary><b>Included addresses</b></summary>

**theindex.finance core:**
- `$INDEX` token: `0x56910D4409F3a0C78C64DD8D0545FF0705389870`
- `StockDistributor`: `0x2459dedb3012d1e929edd17df26620120bdf11bf`
- `StockTreasury`: `0x1604Ff11dFeAaC437077aEDA2FA492ac9EC804dF`
- `IndexFeeHook`: `0x2cD91bD228ff4c537031d6b8204782090c84c0cC`

**Uniswap V4 on Robinhood Chain:**
- `PoolManager`: `0x8366a39CC670B4001A1121B8F6A443A643e40951`
- `Universal Router`: `0xE28c0e44F4016b073db20cF28971CAc6ce3664D3`

**15 stock tokens (long.xyz):** NVDA, SPCX, GOOGL, MU, MSFT, AAPL, TSLA, SPY, PLTR, INTC, COIN, AMD, META, USAR, SNDK

**Native tokens:** WETH, USDG

</details>

**ABIs:** `abis/erc20.json` (covers all stock tokens), `abis/chainlink_aggregator_v3.json` (price feeds), and `abis/stock_distributor.json` (verified from Blockscout) are included. Run `scripts/fetch_abis.sh` to pull remaining ABIs from Blockscout.

---

## CLI usage

<table>
<tr>
<th align="left" width="30%">Command</th>
<th align="left">What it does</th>
</tr>
<tr>
<td><code>pongo</code></td>
<td>Animated boot sequence with mascot. Shows config status, LLM provider, and readiness.</td>
</tr>
<tr>
<td><code>pongo scan</code></td>
<td>Boot + run one pass of all agents. Reads on-chain state, enqueues proposals. <b>No transactions sent.</b></td>
</tr>
<tr>
<td><code>pongo proposals list</code></td>
<td>List pending proposals. Add <code>--all</code> to include approved/rejected/executed/failed.</td>
</tr>
<tr>
<td><code>pongo proposals show &lt;id&gt;</code></td>
<td>Full JSON of a proposal — calldata, rationale, simulation result, notional.</td>
</tr>
<tr>
<td><code>pongo proposals approve &lt;id&gt;</code></td>
<td>Mark a proposal ready for execution. Add <code>--by "name"</code> for audit trail.</td>
</tr>
<tr>
<td><code>pongo proposals reject &lt;id&gt; --reason "..."</code></td>
<td>Reject with a required reason.</td>
</tr>
<tr>
<td><code>pongo execute</code></td>
<td>For each approved proposal: re-check guardrails, prompt for final confirmation, sign, send.</td>
</tr>
<tr>
<td><code>pongo execute --yes-all</code></td>
<td>Skip interactive confirmation. <b>Testnet automation only.</b></td>
</tr>
<tr>
<td><code>pongo report pnl [--since DATE]</code></td>
<td>P&amp;L report. Defaults to last 30 days.</td>
</tr>
<tr>
<td><code>pongo watch --interval 15m</code></td>
<td>Daemon mode. Runs <code>scan</code> on a schedule (accepts <code>s</code>/<code>m</code>/<code>h</code> suffixes). Approval still required.</td>
</tr>
<tr>
<td><code>pongo help [command]</code></td>
<td>Show help. Fast — no boot animation.</td>
</tr>
</table>

---

## End-to-end flow

From zero to first signed transaction on testnet:

```
 ┌────┐    ┌────────┐    ┌────────┐    ┌─────────┐    ┌─────────┐    ┌────────┐    ┌────────┐
 │ 1  │ →  │   2    │ →  │   3    │ →  │    4    │ →  │    5    │ →  │   6    │ →  │   7    │
 └────┘    └────────┘    └────────┘    └─────────┘    └─────────┘    └────────┘    └────────┘
 Install   Configure     Fund from    Fetch ABIs     Tune policy    pongo scan   pongo execute
            .env          faucet
```

<details>
<summary><b>Step-by-step</b></summary>

1. **Install** the project (see [Installation](#project-installation)).
2. **Configure `.env`** — pick your LLM provider, set testnet RPC.
3. **Fund** your test wallet from the testnet faucet.
4. **Fetch ABIs** — run `scripts/fetch_abis.sh` to pull verified ABIs from Blockscout.
5. **Tune `policy.yaml`** — start with a small target allocation and tight guardrails.
6. Run `pongo scan` — watch proposals appear in `pongo proposals list`.
7. `pongo proposals show <id>` — inspect calldata, simulation, notional.
8. `pongo proposals approve <id>` — approve one.
9. `pongo execute` — confirm the prompt; keep the returned tx hash.
10. Verify on the block explorer that the tx was mined.
11. `pongo report pnl` — see the result.

Only after this cycle passes cleanly on testnet, migrate to mainnet: swap RPC, swap chain_id, set `SIGNER_MODE=safe`, and point at your treasury Safe.

</details>

---

## Tests

```bash
pip install -e ".[dev]"
pytest
```

The test suite includes 11 security-focused tests:

- **Whitelist bypass prevention** — rejects whitelisted symbol with mismatched address
- **Mandatory fields** — rejects proposals without notional_usd or slippage_bps
- **Negative value rejection** — catches negative notional or slippage
- **Gas units ceiling** — blocks unusually expensive calldata
- **Mainnet safety preflight** — refuses SIGNER_MODE=local against non-testnet RPC
- **Queue lifecycle** — enqueue, list, approve flow

Lint and type-check:

```bash
ruff check .
mypy src
```

---

## Security

> **Warning** — Read this section before running against mainnet.

- **Never commit `.env`.** `.gitignore` covers it, but confirm `git status` shows only `.env.example` before your first commit.
- **Testnet first.** The full flow must pass cleanly on testnet before touching mainnet.
- **Mainnet preflight.** The executor refuses to run `SIGNER_MODE=local` against a non-testnet RPC. Override only if you explicitly set `guardrails.allow_mainnet_local_signer: true` in policy.yaml (not recommended).
- **Use a Safe for company funds.** Setting `SIGNER_MODE=safe` and pointing at a Gnosis Safe means transactions require N-of-M signatures from Safe owners. Sensible default for treasury: **2-of-3**.
- **Address verification.** The submit tool resolves contract addresses from `contracts.yaml` — agents cannot supply arbitrary addresses. Guardrails verify the resolved address matches the whitelist entry.
- **Simulation at submit time.** Every proposal runs through `eth_call` before entering the queue. Reverting transactions never reach the human for approval.
- **Nonce lock.** A cross-process file lock prevents parallel `pongo execute` runs from grabbing the same nonce.
- **EIP-1559.** The signer uses `maxFeePerGas`/`maxPriorityFeePerGas` when the chain supports it, with legacy fallback.
- **Guardrails are a floor, not a ceiling.** LLMs can hallucinate. Always review calldata and simulation output at approval time.
- **The key never leaves the signer.** No agent code reads `PRIVATE_KEY`. If a lint check ever shows an agent module importing it, treat that as a critical bug.

---

## Contracts

### theindex.finance — how dividends work

The StockDistributor is a **push system**, not a claim system. A keeper runs cycles every ~15 minutes:

```
canStart() == true → startCycle() → snapshotHolders() → buyStocks() → distributeBatch()
```

INDEX holders **receive stock tokens directly in their wallets** — no manual claim needed. PongoAI's Index Agent monitors distribution cycles and tracks received stock tokens via ERC-20 balances.

### long.xyz — stock tokens

Long.xyz hosts tokenized stock positions on Robinhood Chain. All stock tokens are standard ERC-20 with 18 decimals. PongoAI tracks 15 tokens (NVDA, TSLA, AAPL, SPY, SPCX, GOOGL, MU, MSFT, PLTR, INTC, COIN, AMD, META, USAR, SNDK) with addresses configured in `contracts.yaml`.

---

## Roadmap / status

**Working today**

- [x] LangGraph `StateGraph` wiring four specialist ReAct agents
- [x] Multi-provider LLM support (Anthropic, OpenAI, OpenRouter, Ollama, local)
- [x] Animated CLI with mascot boot sequence
- [x] SQLite proposal queue with full lifecycle (enqueue → approve → execute)
- [x] Hard guardrails (address-verified whitelist, mandatory notional/slippage, gas ceiling, daily count)
- [x] Simulation at submit time (rejects reverting transactions)
- [x] EIP-1559 signer with cross-process nonce lock
- [x] Mainnet safety preflight (blocks local signer on non-testnet)
- [x] Real contract addresses for theindex.finance and long.xyz
- [x] StockDistributor ABI (verified from Blockscout)
- [x] 11 security-focused tests
- [x] Full CLI (`pongo`, `scan`, `proposals`, `execute`, `report`, `watch`, `help`)

**Stubs pending real implementation**

- [ ] `src/tools/long_tools.py` — read positions, encode swaps on long.xyz
- [ ] `src/tools/index_tools.py` — read distribution cycles, track received stock tokens
- [ ] `src/tools/portfolio.py` — aggregate snapshot and historical P&L
- [ ] `src/executor/signer.py` — Safe Transaction Service integration, KMS mode
- [ ] Historical snapshot persistence for accurate P&L over time
- [ ] Remaining ABIs (StockTreasury, IndexFeeHook, Universal Router)

**Suggested build order**

1. `index_tools` — read `StockDistributor.canStart()`, `nextDistribution()`, `cycleActive()`
2. `portfolio_snapshot` — sum stock token balances across both protocols
3. `long_tools.get_position_snapshot` — ERC-20 balanceOf + price from Chainlink
4. Calldata encoders for swaps via Universal Router
5. Safe mode of the signer, then mainnet migration

---

## Contributing

Contributions welcome. Please open an issue before large changes so we can align on approach.

1. Fork and create a feature branch.
2. `pip install -e ".[dev]"` and make sure `pytest`, `ruff check`, and `mypy src` all pass.
3. Add a test for anything security-relevant (guardrails, queue transitions).
4. Open a PR with a clear description of what changed and why.

---

## License

Released under the [MIT License](LICENSE).
