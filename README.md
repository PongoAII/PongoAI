```
    ____                          _    ____
   / __ \____  ____  ____ _____  / \  /  _/
  / /_/ / __ \/ __ \/ __ `/ __ \/ _ \ / /
 / ____/ /_/ / / / / /_/ / /_/ / ___// /
/_/    \____/_/ /_/\__, /\____/\/  /___/
                  /____/

       > LangGraph multi-agent orchestrator on Robinhood Chain
       > agents propose - humans approve - signer signs
```

```console
guest@pongoai:~$ whoami
PongoAI - an autonomous fleet of agents that read on-chain state,
propose transactions, wait for human approval, and settle safely.

guest@pongoai:~$ cat mission.txt
Manage RWA (real-world asset) exposure on Robinhood Chain by
orchestrating positions on long.xyz and dividend streams from
theindex.finance - with hard guardrails on every value-moving action.

guest@pongoai:~$ tree -L 1 stack/
stack/
|-- agents/      LangGraph . StateGraph . ReAct loops
|-- llm/         Anthropic Claude . OpenAI GPT
|-- chain/       web3.py 7.x . eth-account . EIP-1559
|-- storage/     SQLModel . SQLite proposal queue
`-- runtime/     Python 3.11+ . Click . Rich

guest@pongoai:~$ grep -R "focus" .
./chain      : Robinhood Chain (Arbitrum L2)
./assets     : tokenized stocks . dividend streams . long.xyz launches
./mode       : human-in-the-loop . signer isolated from agents
./guardrails : whitelist . notional cap . gas ceiling . sim before send
```

```console
guest@pongoai:~$ systemctl status pongoai.service
* pongoai.service - PongoAI orchestrator
   Loaded: loaded (config/policy.yaml; enabled)
   Active: active (scaffold) since 2026-07-22
   Status: "wiring real ABIs - testnet dress rehearsal next"

guest@pongoai:~$ cat roadmap.txt
[ ] fill contracts.yaml with real addresses
[ ] implement long.xyz position reader
[ ] implement theindex.finance dividend reader
[ ] SIGNER_MODE=safe integration (Gnosis Safe)
[ ] first testnet execution end-to-end
[ ] public v0.1.0 release

guest@pongoai:~$ echo $PRINCIPLES
1. Agents propose. Humans approve. A separate signer signs.
2. Every transaction is simulated before it reaches the human.
3. Guardrails run outside the LLM - no prompt can bypass them.
4. The private key never leaves the signer process.

guest@pongoai:~$ _
```

---

<sub>` > STATUS `  scaffold complete . integrating real contracts on testnet</sub><br>
<sub>` > STACK  `  python . langgraph . web3.py . sqlmodel . click . rich</sub><br>
<sub>` > CHAIN  `  Robinhood Chain L2 . long.xyz . theindex.finance</sub>
