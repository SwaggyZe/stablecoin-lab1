# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

MINTER_ROLE combines two powers that need different authorization: creating liabilities and destroying another account's property. A compromised authorized contract could burn a holder's balance without paying collateral back. However, possession of a role is not the same as a publicly reachable exploit: the current immutable Vault only calls burn for msg.sender inside redeem. The impersonation test demonstrates what the token would permit if the vault address could make an arbitrary call; it does not prove that an external attacker can currently instruct this Vault to burn Alice's tokens.

I would remove the role-authorized burn(address,uint256) function. The token would instead expose burn(uint256), which burns only msg.sender's balance. Vault.redeem would first transferFrom the caller's sUSD into the vault, then burn its own balance, then return collateral to that same caller. The holder must approve the vault explicitly. Alternatively, burnFrom could consume the holder's allowance. Failed collateral transfers must revert the entire transaction, and redemption should have a reentrancy guard. Tests should cover missing and insufficient allowance, successful redemption, rollback, and attempts to burn another holder's tokens. Separating mint authorization remains necessary; fixing burn alone does not stop unbacked issuance.

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

The constructor gives the administrator three roles, while deployment additionally authorizes the vault to mint. This is convenient for a lab but creates a single point of failure: a stolen administrator key can grant an attacker minting rights or freeze all holders. Splitting visible roles is insufficient if a single hot wallet still retains the administrator authority to recreate them.

For a production design, DEFAULT_ADMIN_ROLE should belong to a threshold multisignature controlled by independent governance participants, with a delay for routine privilege changes and a two-step administrator transfer. MINTER_ROLE should belong only to reviewed issuance contracts whose state transitions verify incoming collateral and enforce issuance limits. After the initial configuration is verified, the deployment account should lose its direct minting privilege. PAUSER_ROLE could belong to a separate emergency security multisignature with a narrowly scoped ability to stop risky operations. Resuming service and changing emergency policy should require stronger governance review. Monitoring should alert on role grants, unexpected supply changes, and reserve discrepancies. Key rotation and recovery need rehearsed procedures. Multisignatures reduce individual-key risk but do not eliminate coordinated abuse, contract defects, or an incorrectly designed redemption policy.

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

I would separate transfer, issuance, and redemption controls instead of attaching one whenNotPaused modifier to every balance update. In the token's _update hook, an ordinary transfer has both from and to nonzero; minting has from equal to zero, and burning has to equal to zero. A transfer pause should reject only ordinary transfers. Issuance should have its own stop switch so that a suspected minting exploit can be contained without blocking solvent holders from exiting.

In the current lab, redemption burns the caller's balance directly, so allowing authorized burns while transfers are paused lets Vault.redeem continue. If implementing the safer transfer-then-burn design described in A1, ordinary transfer pausing would also block the transfer into the vault. That requires an explicit, narrowly authorized redemption path or an allowance-consuming burnFrom path rather than a broad exemption for every vault-related transfer. A separate redemption emergency control may still be needed if the redemption function itself is vulnerable, but its scope and recovery process should be documented. Tests should verify transfers and minting fail under their respective flags, valid redemptions succeed, allowances remain enforced, and failed payouts roll back burns.

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

The question's contrast needs qualification: keeping a redemption commitment is different from processing every request continuously. The Reserve Primary Fund delayed and suspended redemption payments after its net asset value fell below one dollar in September 2008. Circle's March 2023 experience also involved banking-hour restrictions and processing backlogs; describing USDC redemptions as uninterrupted throughout the weekend would be misleading. Circle reported that substantially all backlogs had cleared by March 15.

Mechanically, closing redemption removes the direct route from a discounted token to its underlying assets. A holder may then accept a larger secondary-market discount because both recovery value and timing are uncertain. A suspension can reduce immediate forced sales or unequal treatment of investors, but it transfers liquidity costs to holders. Keeping a credible redemption route available gives arbitrageurs an incentive to buy below par and redeem, provided reserves are accessible and transaction costs and eligibility restrictions permit it. It does not repair an actual asset shortfall. The relevant distinction is therefore solvency, access to liquid reserves, and settlement timing, rather than merely whether a pause flag is set. These are different failure modes and need different remedies.

Sources: [SEC Reserve Fund FAQ](https://www.sec.gov/divisions/investment/guidance/reservefundmmffaq.htm); [Federal Reserve analysis](https://www.federalreserve.gov/econres/notes/feds-notes/in-the-shadow-of-bank-run-lessons-from-the-silicon-valley-bank-failure-and-its-impact-on-stablecoins-20251217.html); [Circle operations update](https://www.circle.com/es/blog-all).

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

One class is a balance-sheet shortfall measured in collateral units. Unauthorized minting raises totalSupply without increasing the vault's collateral balance, so totalCollateral >= totalSupply fails immediately. An exploit that transfers collateral out without burning the corresponding liabilities produces the same result. The added compromised-minter test shows that an attacker can redeem newly minted coins and leave honest holders with unbacked balances.

A second class is redemption unavailability. Pausing, a broken payout path, or loss of required permissions can prevent conversion even while the numerical inequality remains true. Markets can discount a token because its holders cannot realize the promised asset value. A third class is collateral impairment: one USDC token need not always be worth one US dollar. Equal token-unit balances therefore do not establish dollar solvency. A realistic check must use conservative collateral valuation and also assess liquidity, timing, and enforceability of claims.

Finally, exact equality is narrower than sufficient backing: someone may donate collateral directly to Vault, producing excess assets without a defect. Likewise, the no-sUSD-in-vault invariant holds for our restricted deposit/redeem handler, but arbitrary token transfers can violate it. Neither invariant alone proves a market price of exactly one dollar.

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

The attacker first obtains MINTER_ROLE through an administrator's malicious or compromised grant. They call mint(attacker, 1_000_000e6), which creates one million sUSD without moving any collateral into the vault. Supply increases while assets stay constant. They then call Vault.redeem with those coins. The vault burns the attacker's sUSD and transfers the corresponding quantity of collateral to the attacker. Honest holders retain their claims, but the assets backing those claims have been removed.

There is an essential liquidity condition: the attacker can redeem the entire million in one call only if the vault holds at least one million collateral tokens. Otherwise that call reverts atomically with InsufficientCollateral. The attacker may still redeem a smaller amount up to the available reserve, draining assets belonging economically to other holders. This is why a check that the requested withdrawal fits the current balance does not establish that the caller's coins were legitimately backed when issued.

The decisive preventive step is refusing unauthorized privilege assignment and eliminating unrestricted issuance paths. Governance separation, delayed role grants, issuance limits, and monitoring provide additional layers. Detection after minting is useful only if an effective intervention can happen before collateral leaves; revoking minting privileges alone does not invalidate coins already issued.

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

An on-chain token balance would represent a claim on Treasuries, not direct proof that the vault owns accessible, unencumbered securities. A useful backing condition would compare total stablecoin liabilities with conservatively valued net reserve assets, expressed in the same dollar units. The reserve value should account for market prices, accrued interest where appropriate, fees, senior claims, settlement costs, and a liquidity haircut. This is a model-dependent solvency check, not the simple equality between two ERC-20 counters used in the lab.

New dependencies include a custodian's records, reconciliation between issued claims and actual holdings, reliable valuation updates, and enforceable rights if an intermediary becomes insolvent. Interest-rate changes can lower the sale value of securities even when their maturity payments are unchanged. Assets may also be solvent on paper yet unavailable during a banking closure or settlement delay. A reserve attestation is evidence about a particular scope and date; it is not continuous proof of unrestricted access.

The design therefore needs both a solvency measurement and a separate short-horizon liquidity policy for redemptions. Stale valuations, missing reports, and reconciliation failures should restrict new issuance. This conceptual design cannot establish legal ownership by itself.

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

A building cannot literally be transferred into an EVM contract. A possible design places legal ownership in a dedicated entity and issues digital claims representing defined rights against that entity. The vault could hold those claims as collateral, but the legal documentation must explain what holders can enforce, which creditors rank ahead of them, and how a default or liquidation is handled. Transferring a token must not be assumed to transfer registered property title automatically.

Relevant off-chain roles could include the property-owning entity, a title or claims administrator, independent valuation providers, a property manager, insurers, and parties responsible for enforcing security rights. The exact legal structure depends on jurisdiction and requires specialist review. Appraisals arrive infrequently, can be disputed, and may differ materially from a forced-sale price. Mortgages, taxes, maintenance, vacancies, and transaction expenses reduce value available to token holders.

Consequently, issuing immediately redeemable one-dollar liabilities against an illiquid building introduces a major maturity mismatch. A design might need conservative borrowing limits, liquid reserves, disclosed redemption notice periods, or auctions. Its invariant should use conservative net recoverable value and separate near-term payout capacity. Neither an NFT nor an oracle eliminates physical, legal, or liquidity risk.

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts |`test_Ex4_Mint_RevertsForNonMinter` |
| Transfers revert while paused |`test_Ex4_Pause_BlocksTransfers` |
| **Redemption** reverts while paused |`test_Ex4_Pause_BlocksRedeem` |
| An attacker cannot burn someone else's balance |`test_Ex4_AttackerCannotBurnOthersBalance` |
| ...but the vault holding `MINTER_ROLE` can |`test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance` |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

Implemented `test_Extra_CompromisedMinterDrainsCollateral`: after a privileged role grant, the attacker mints 100 sUSD and redeems the entire 100 mUSDC reserve, leaving Alice with 100 sUSD and no collateral. This targets the permission boundary that protects every holder, and proves that the reserve-balance check alone does not prevent theft.
