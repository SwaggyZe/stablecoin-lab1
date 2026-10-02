# Lab 1 — How to Create a Stablecoin

In this lab you build a **fiat-collateralized stablecoin** from scratch: deposit collateral to mint, burn to redeem, fully backed at every moment.

The core claim, up front:

> A stablecoin's "stability" does not come from the ERC-20 code — it comes from the **mint-redeem loop**.
> ERC-20 is only a bookkeeping format; anyone can write one in ten minutes. The real work is designing the loop and the permissions around it.

---

## 1. Setup

Pick the track that matches your machine. **Everything after this section is identical for everyone.**

| You are on | Track |
|---|---|
| macOS / Linux | **A** — install Foundry locally |
| Windows | **B** — Codespaces, nothing installed on your machine |

> `lib/` (`forge-std`, `openzeppelin-contracts`) **ships inside this repository**.
> Neither track needs `git clone --recursive`, and neither needs `make setup` —
> a plain clone compiles as-is.

### Track A — macOS / Linux

**Step 1. Install Foundry**

First check whether you already have it:

```bash
forge --version
```

If that prints a version, skip to Step 2. Otherwise install it with the script in this repo (**do not use `foundryup`**):

```bash
bash scripts/install-foundry-cn.sh
```

The script detects your platform (macOS ARM / Intel, Linux x86_64 / arm64), tries GitHub directly, and falls back to the `gh-proxy.com` mirror. Add the line it prints to `~/.zshrc` or `~/.bashrc`, then reopen your terminal:

```bash
export PATH="$PATH:$HOME/.foundry/bin"
```

> ⚠️ **Why not `foundryup`**: its 108 MB release download goes through GitHub's CDN,
> which measured ~33 KB/s from mainland China and times out reliably
> (`Operation timed out (os error 60)`). The mirror measured ~3 MB/s — 37 seconds.

**Step 2. Get the code**

```bash
git clone https://github.com/hgwoops/stablecoin-lab-2026 && cd stablecoin-lab-2026
```

Then go to **Verify** below.

### Track B — Windows (Codespaces)

Foundry has **no native Windows binary** — the official releases ship only macOS and Linux
builds, so `scripts/install-foundry-cn.sh` will refuse to run for you, and Git Bash /
PowerShell are not workarounds. The straightforward path is to run the lab on GitHub's
servers from your browser. Nothing is installed locally, which also makes it the only
option on locked-down corporate machines.

1. Open <https://github.com/hgwoops/stablecoin-lab-2026>
2. Green **`Code`** button → **`Codespaces`** tab → **`Create codespace on main`**
3. **The first launch takes 2–5 minutes**: it builds the environment from `.devcontainer/`,
   then automatically runs `make doctor` — you should see a row of `✓` in the terminal
4. In the terminal at the bottom, run:

```bash
make test
```

`7 passed; 0 failed` means you are set.

The free tier is 120 core-hours/month (≈ 60 real hours on a 2-core machine), far more than
this lab needs. **Stop it when you are done** at <https://github.com/codespaces> — a running
Codespace keeps burning quota.

> If you would rather have a local environment on Windows, install **WSL2 + Ubuntu** and then
> follow Track A inside it. That is a one-time configuration (enable virtualization in BIOS,
> enable "Virtual Machine Platform", one admin elevation), after which it is as fast as Linux.

### Verify (both tracks)

```bash
make doctor
```

In about 30 seconds this tells you whether the machine is ready for class: it checks the
toolchain, the dependencies, and actually runs the core tests. Anything marked `✗` comes with
the exact command to fix it.

All green means you can start. **If you cannot fix it, paste the entire `make doctor` output to the TA.**

---

## 2. Quick start

```bash
make doctor        # environment check — all green before you go on
make test          # Ex0 checkpoint: core tests, should be all green (7 passed)
make exercise      # the hands-on tasks; red right now — the red ones ARE your task list
make anvil         # terminal A: start a local chain
make deploy-anvil  # terminal B: deploy to it
```

Deployment prints three addresses — write them down (they are referred to below as `$USDC`,
`$SUSD`, `$VAULT`). For the full set of minting, balance-checking and on-chain commands, see
Ex1 and Ex3 in `EXERCISES.md`.

**Your task list lives in `EXERCISES.md`** — every exercise's goal, acceptance command, and
where to look when you get stuck.

---

## 3. What the three contracts do

| File | Role |
|---|---|
| `src/MockUSDC.sol` | **The collateral.** A mock USDC: 6 decimals, with a test faucet |
| `src/SimpleStablecoin.sol` | **The stablecoin itself.** ERC-20 + `MINTER_ROLE` (who may mint) + `PAUSER_ROLE` (whether transfers are frozen) |
| `src/Vault.sol` | **The loop.** Deposit collateral to mint 1:1; burn to take collateral back |

The key invariant: **`vault.totalCollateral() == stable.totalSupply()`**

As long as that equation holds, every coin is fully backed. The moment someone can mint out of
thin air without collateral appearing, the equation breaks and the coin stops being stable.

`script/Deploy.s.sol` wires all three together in a single command, and finishes by granting
`MINTER_ROLE` to the vault — **skip that step and every deposit reverts**.

`src/exercises/` is a second system to practise on: `OverCollateralizedVault.sol`
(over-collateralization + liquidation) together with `MockPriceFeed.sol` (a price feed you can
move by hand). It contains four TODOs, which are the subject of Ex5.

---

## 4. Homework

### Tier 1 (required) — Ex0–Ex6 in `EXERCISES.md`

1. **Ex0 Environment**: `make doctor` all green → `make test` all green
2. **Ex1 The loop**: run it once from the command line — faucet mUSDC → `approve` → `deposit`
   to mint → `redeem` to get collateral back
3. **Ex2 Decimals trap**: write the two `test_Ex2_*` tests in `test/exercises/01_LoopTasks.t.sol`
4. **Ex3 Break the peg yourself**: use `cast` to grant `MINTER_ROLE` to an attacker and mint.
   Submit a screenshot of `totalSupply()` far exceeding `totalCollateral()`
5. **Ex4 Permissions and pausing**: write the five `test_Ex4_*` tests in the same file
6. **Ex5 Over-collateralization + liquidation**: fill in the four TODOs in
   `src/exercises/OverCollateralizedVault.sol` until `test/exercises/03_OverCollateralTasks.t.sol`
   is green
7. **Ex6 Invariant testing**: implement the handler's `redeem` and write two `invariant_*` tests
8. Answer the discussion questions in `STUDENT-QUESTIONS.md`

> Tier 1 does **not** require deploying to a testnet — a local Anvil is enough.
> The acceptance command for every exercise is `make exercise`; when you are done it should be all green.

### Tier 2 (bonus)

Deploy to the Sepolia testnet and verify the source on Etherscan; submit the contract links.

```bash
cp .env.example .env    # fill in private key, RPC URL, Etherscan API key
source .env
forge script script/Deploy.s.sol:Deploy \
  --rpc-url $SEPOLIA_RPC_URL --broadcast --verify --private-key $PRIVATE_KEY
```

**`--verify` needs `ETHERSCAN_API_KEY`** — free and takes seconds at
<https://etherscan.io/myapikey>. Leaving it empty still deploys fine: the contracts go on-chain
and only the verification step errors. **Do not redeploy because of it** — verify afterwards:

```bash
# one run per contract, with its own address and source file
forge verify-contract <address> src/SimpleStablecoin.sol:SimpleStablecoin \
  --chain sepolia --etherscan-api-key $ETHERSCAN_API_KEY
```

⚠️ Use a **fresh throwaway wallet** holding test ETH only. The private key must stay in `.env`,
which is already listed in `.gitignore` — **committing a private key scores zero**.

Sepolia test ETH: <https://cloud.google.com/application/web3/faucet/ethereum/sepolia>

### Tier 3 (challenge, optional)

Pick one:

- **Ex7 · Take the vault down**: `make challenge` (`test/challenges/Unstoppable.t.sol`, ported
  from Damn Vulnerable DeFi v4). You hold 10 DVT; the goal is to stop the vault from offering
  flash loans.
- **Peg Stability Module (PSM)**: add a 1:1 swap channel against another stablecoin, so that
  arbitrageurs pull the price back to peg for you.
- **Tighten the burn permission**: remove the centralization risk that the vault can burn
  anyone's balance — the code version of question A1 in `STUDENT-QUESTIONS.md`.
- **Wire in real Chainlink**: replace `MockPriceFeed` with Sepolia's `AggregatorV3Interface`.
  Read `decimals()` first; do not hardcode 8.

---

## 5. Common problems

**`Ownable` / role errors**
OpenZeppelin v5 has breaking changes, so v4 tutorials from the web will fail. This repo pins
`v5.0.2` — do not upgrade it.

**`ds-test/test.sol` not found, or `lib/` is empty**
`lib/` is bundled with the repo, so a normal `git clone` never hits this. You probably modified
`lib/`, or unzipped an archive without the directory. Restore it:

```bash
git checkout -- lib
```

**`deposit` keeps reverting after deployment**
Nine times out of ten you skipped `stable.grantRole(stable.MINTER_ROLE(), address(vault))`.
The vault has no minting right.

**Amounts are off by an order of magnitude**
`1000e6 = 1_000_000_000` in smallest units. Both tokens have 6 decimals — do not compute with 18.

**Addresses change after restarting Anvil**
Anvil starts from a clean state every time, so you must redeploy. To keep state, use
`anvil --load-state demo-state.json`.

**`make exercise` is all red**
That is expected — the tests under `test/exercises/` are **deliberately red**; they are your task
list. `make test` is the "everything is fine" checkpoint. Once you finish, `make exercise` goes green.

**`make exercise` hangs, or an invariant failure looks strange**
Foundry stores invariant counterexamples in `cache/invariant/` and replays them on the next run.
To search for a fresh counterexample, `rm -rf cache/invariant` first.

---

## 6. Submission

| Item | Weight |
|---|---|
| Contract correctness + test coverage | 40% |
| Deployment and Etherscan verification (Tier 2) | 20% |
| Threat-model analysis (discussion questions) | 25% |
| Code style and documentation | 15% |

Submit:

1. A link to your repo, with a screenshot of `forge test` passing
2. Tier 2: Sepolia contract addresses + Etherscan links
3. Your answers to the discussion questions in `STUDENT-QUESTIONS.md`
4. An architecture diagram in your README — a photo of a hand drawing is fine

---

## 7. Questions left to you

These have no standard answers. They are the real point of this lab:

1. The vault holds `MINTER_ROLE`, which means it can `burn` any user's balance. Is that
   acceptable? How would you change it?
2. Pausing freezes transfers, minting and **redemption** together. What happens when you freeze
   redemption in a crisis? How should a real system be designed?
3. Under what conditions does this coin depeg? Are "insufficient collateral" and "a blocked
   redemption channel" the same class of problem?
4. If the collateral were Treasuries or real estate instead of cash, how would you write the
   `totalCollateral()` invariant?

Question 4 is the door into next week's RWA lab.

---

## Completed lab: architecture and evidence

See [discussion answers](STUDENT-QUESTIONS.md).

```mermaid
flowchart LR
    U[User] -->|1. approve collateral spending| C[MockUSDC: 6 decimals]
    U -->|2. deposit amount| V[Vault]
    C -->|transferFrom user to vault| V
    V -->|mint to user: MINTER_ROLE| S[SimpleStablecoin: 6 decimals]
    U -->|3. redeem amount| V
    V -->|burn caller balance| S
    V -->|return collateral to caller| U
    A[Admin] -->|grant roles| S
    P[Pauser] -->|pause all balance updates| S
```

The normal loop preserves equal collateral and supply. Direct collateral donations can produce a surplus, so the backing invariant uses `>=`. Authorized unbacked minting breaks backing. The no-stablecoin-in-vault property is scoped to the tested deposit/redeem operations; arbitrary transfers can invalidate it.

```mermaid
flowchart LR
    B[Borrower] -->|deposit 18-decimal mWETH| O[OverCollateralizedVault]
    F[MockPriceFeed: 8 decimals] -->|collateral valuation| O
    O -->|mint 6-decimal sUSD at ratio >=150%| B
    L[Liquidator] -->|burn own sUSD for borrower debt| O
    O -->|seize collateral plus 10% bonus, capped at balance| L
```

Liquidation requires a ratio below 120%. A liquidator must pay the whole debt even when collateral is insufficient; voluntary liquidation may be uneconomic. The mock oracle, permission model, and fixed decimal assumptions are educational limitations.

Verified locally with Foundry v1.8.4 / Solidity 0.8.24: **29 tests passed, 0 failed**, including Ex7. The invariant run used 256 runs and 128,000 calls with zero reverts. See [test output](evidence/forge-test.txt), [environment checks](evidence/doctor.txt), and [actual local-chain execution](evidence/local-demo.txt). The completed Sepolia deployment and source verification are recorded below.

Ex3 actual result: `totalSupply = 1000600000000`, `totalCollateral = 600000000` (both 6-decimal raw units). This demonstrates undercollateralization; there is no exchange price feed in this lab that measures a market depeg.

The upstream Windows installation statement is outdated: [official Foundry v1.8.4 releases](https://github.com/foundry-rs/foundry/releases/tag/v1.8.4) include Windows binaries. The local tool archive was checked against its published SHA256.

### Codespaces execution screenshots

These original screenshots document the student's Codespaces execution:

- Ex1: after redeeming 400 sUSD, supply, collateral and user sUSD balance are each 600 tokens; user mUSDC balance is 400 tokens.
- Ex3: after authorized unbacked minting, supply is 1,000,600 sUSD while collateral remains 600 mUSDC.

![Ex1 redemption balances](evidence/ex1-redeem.png)

![Ex3 unbacked issuance](evidence/ex3-depeg.png)

### Codespaces final test verification

After restoring the original Ex5 acceptance-test file, the Codespaces run passed **28 tests, 0 failed, 0 skipped**, including the ten original Ex5 tests, Ex7, and both invariant properties. The invariant run made 128,000 calls with zero reverts. The local 29-test run also includes an additional handler smoke test; this accounts for the one-test difference.

![Ex5: ten original acceptance tests passed](evidence/ex5-tests.png)

![Codespaces: all 28 tests passed](evidence/forge-test.png)


## Sepolia deployment and source verification (Tier 2)

All three contracts were deployed to **Ethereum Sepolia (chain ID 11155111)** and verified on Etherscan. The original Codespaces verification output reports **“All (3) contracts were verified!”**.

Deployer: `0x8dd34bBe1096BA2bf8107fD68E5374c177Ca459A`.

| Contract | Sepolia address | Etherscan |
|---|---|---|
| MockUSDC | `0xc50a8D5Fb485B85E2Fc06ADEf1D6eD394F267FD0` | [Verified source](https://sepolia.etherscan.io/address/0xc50a8D5Fb485B85E2Fc06ADEf1D6eD394F267FD0#code) |
| SimpleStablecoin | `0x0541A56b2BfFBdAbBa4e690656089D65b17DdFF6` | [Verified source](https://sepolia.etherscan.io/address/0x0541A56b2BfFBdAbBa4e690656089D65b17DdFF6#code) |
| Vault | `0x4DD54bd095374E08D8D31967fBB919514D685242` | [Verified source](https://sepolia.etherscan.io/address/0x4DD54bd095374E08D8D31967fBB919514D685242#code) |

Read-only checks confirmed that Vault's `collateral()` and `stable()` reference the two deployed tokens, both tokens use 6 decimals, and Vault has `MINTER_ROLE`. The stablecoin was not paused. At the initial post-deployment check, supply and collateral were both zero. See [the checks](evidence/sepolia-checks.txt) and [deployment record](evidence/sepolia-deployment.json).

The Ex1 and Ex3 demonstrations above were executed on local Anvil. The addresses in this section are the public Sepolia deployments.

The first broadcast encountered the delegated-account transaction-pool limit after deploying the two tokens. Deployment completed by resuming the saved transaction sequence with `--resume --slow`, confirming each transaction before sending the next.

![Sepolia: all three contracts verified](evidence/sepolia-verified.png)
