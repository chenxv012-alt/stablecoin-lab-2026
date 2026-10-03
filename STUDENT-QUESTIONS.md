# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:

The risk is that MINTER_ROLE combines two powers that should be separate. The vault needs permission to create sUSD after receiving collateral, but its current permission also lets it call burn(user, amount) against an arbitrary account. A bug, compromised vault, or malicious upgrade could erase a holder’s tokens without that holder asking to redeem. A normal attacker cannot do this, as the Ex4 test shows, but that protection does not help if the powerful role is held by the wrong contract.

I would change SimpleStablecoin so MINTER_ROLE authorizes minting only. Burning someone else’s balance would require that person’s explicit allowance or a narrowly scoped signed authorization. In Vault.redeem, the user would approve an exact sUSD amount, and the vault would call burnFrom(msg.sender, amount) before returning the corresponding USDC. The vault could no longer choose an unrelated victim as the burn target. I would test both that authorized redemption succeeds and that the vault cannot burn a user’s balance without authorization. Minting would still need collateral checks and limits, because separating burn permission does not eliminate every risk of a compromised minter.

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:

I would split these roles according to how quickly each decision must be made and how much damage it can cause. DEFAULT_ADMIN_ROLE can grant or revoke the other roles, so it should belong to a carefully controlled, multi-signature governance wallet, preferably with a timelock for routine changes. It should not be a developer’s everyday wallet or the vault itself. Changes to this role should be logged, reviewed, and subject to a recovery process.

MINTER_ROLE should belong only to a narrowly designed vault or issuance contract that mints after it has received and accounted for collateral. No ordinary externally owned account should be able to mint directly. I would also add issuance limits and monitoring, because a contract holding this role can still be exploited.

PAUSER_ROLE should belong to a separate emergency-response multi-signature wallet. It may need to pause dangerous activity quickly, but it should not automatically gain minting or administrative powers. Reopening the system could require a slower governance decision and a public explanation. This separation reduces the damage from one compromised key, while making the authority and accountability of each role clearer.

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:

I would stop applying one blanket pause check to every call of _update. In an ERC-20 token, a normal transfer has a nonzero from and a nonzero to; minting has a zero from, and burning has a zero to. The token could reject _update only when it is paused and both addresses are nonzero. That would freeze transfers between holders while allowing the burn required for redemption. If minting should also stop during an incident, I would control minting with a separate flag or check rather than accidentally tying it to the redemption switch.

The vault’s redemption path must also avoid an ordinary sUSD transfer while transfers are paused. For example, it could burn the user’s authorized tokens directly with burnFrom and then return USDC. I would test a paused user-to-user transfer that reverts, a paused redemption that succeeds, and the expected decrease in both total supply and collateral after that redemption. The design should still permit a separate, exceptional halt to redemptions if the collateral itself cannot safely be paid out, but that decision should be explicit and more tightly governed.

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:

The two cases show that a stable asset needs both credible reserves and a usable exit. The Reserve Primary Fund broke its $1 share value in 2008 and suspended redemptions. A freeze can slow a run and give managers time to value or liquidate assets, but it also prevents holders from obtaining cash when they most need it. A token might still move between wallets, yet its promised $1 redemption would be unavailable; buyers would likely demand a discount for that uncertainty. The fund’s problems also spread concern to other money-market funds. SEC account

During the 2023 USDC depeg, Circle maintained its 1:1 redemption commitment. A functioning redemption route gives eligible holders an incentive to buy discounted USDC and redeem it for dollars, which can help the market price return toward $1. However, “not permanently shut” does not mean every redemption settled instantly: banking disruptions caused backlogs, which Circle said it worked through after banking services resumed. Keeping redemption open supports confidence only if reserves and payment channels can actually meet demand; otherwise a rush to redeem could expose a real shortfall. Circle update

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:

One class of depeg is an accounting or authorization failure. If a privileged account mints sUSD without receiving collateral, totalSupply() rises while totalCollateral() does not. When the system previously had exactly one unit of collateral per token, totalCollateral() >= totalSupply() immediately becomes false. A faulty deposit calculation, such as confusing six and eighteen decimals, can produce the same warning. These are failures that the numerical invariant is designed to catch.

A different class is an economic or liquidity failure. The vault can still hold the same number of MockUSDC units as there are sUSD units while the backing asset loses value, becomes inaccessible, or cannot be redeemed for real dollars. In this lab, MockUSDC even has an unrestricted test faucet, so a token balance is not proof of valuable reserves. Likewise, pausing redemption can remove the mechanism that pulls a discounted sUSD back toward $1. In those cases the numerical invariant may remain true while the market price falls below $1. A production system therefore needs reserve-quality, valuation, liquidity, and redemption checks in addition to balance arithmetic.

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:

First, the attacker obtains MINTER_ROLE, perhaps through a compromised administrator or an unauthorized role grant. They call mint(attacker, 1,000,000e6) without depositing USDC. The attacker’s sUSD balance and totalSupply() increase by one million tokens, but vault collateral does not increase. Assuming the system was exactly backed beforehand, the collateral-backs-supply invariant fails at this mint.

Next, if the vault already contains at least one million USDC belonging to legitimate depositors, the attacker calls redeem(1,000,000e6). The vault burns the attacker’s newly created sUSD and sends one million USDC from the shared collateral pool to the attacker. The fake tokens disappear, but the legitimate users’ backing has been drained. If the vault lacks enough collateral, that particular redemption should revert; the unauthorized mint remains a serious problem regardless.

The best stopping point is before the role grant or unauthorized mint: protect DEFAULT_ADMIN_ROLE, restrict minting to a vault that verifies incoming collateral, and enforce issuance limits. A collateral check during redemption is useful too, but it cannot identify whether the attacker’s sUSD was legitimately backed when minted.

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:

Not in the same simple way. MockUSDC.balanceOf(address(vault)) reports units held by an on-chain address. A U.S. Treasury bill is a security held through custodians and legal accounts; the EVM cannot directly inspect who legally owns it, whether it has been pledged elsewhere, or whether a sale will settle in time for redemptions. A token representing a Treasury bill could be held by the vault, but that token would only be evidence of a claim under an off-chain arrangement, not proof by itself that the underlying security is available.

The invariant would have to compare verified, usable dollar reserve value with outstanding sUSD liabilities. That introduces independent custody records, reconciliations, attestations or audits, pricing data, interest and maturity calculations, settlement delays, and possible valuation haircuts. The reporting feed could be stale or wrong even when its on-chain number looks precise. Treasury bills are generally liquid, but “valuable” and “immediately spendable as redemption cash” are different properties. I would therefore test solvency and redemption liquidity separately and make the limits of any on-chain reserve figure explicit.

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:

A building cannot be transferred into a Solidity vault the way an ERC-20 token can. The physical property and its legal title remain off-chain. One possible structure is a legally established special-purpose vehicle (SPV) that owns the building and issues a token representing defined rights in that vehicle. The vault could hold that token or a documented claim against the SPV, but the smart contract would not itself become the registered property owner. The rights attached to the token would need to be enforceable under the relevant jurisdiction’s law.

This arrangement needs more than code: a title and legal team, an SPV manager, a property manager, an independent appraiser, a custodian or trustee for the ownership records, auditors, and an entity responsible for reporting rent, costs, liens, and changes in value. Transfer restrictions and investor eligibility might also matter. Most importantly, a building cannot normally be sold in seconds to meet sUSD redemptions. I would not treat its appraised value as immediately available cash; the system would need a liquid reserve, conservative valuation, and a clearly disclosed redemption policy.
---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts |test_Ex4_Mint_RevertsForNonMinter |
| Transfers revert while paused |test_Ex4_Pause_BlocksTransfers |
| **Redemption** reverts while paused |test_Ex4_Pause_BlocksRedeem |
| An attacker cannot burn someone else's balance |test_Ex4_AttackerCannotBurnOthersBalance |
| ...but the vault holding `MINTER_ROLE` can |test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance|

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
One additional attack scenario: I would test whether an attacker can grant themselves MINTER_ROLE. The existing non-minter test proves that a direct mint call is blocked, but an attacker may first try to obtain the role and then mint. My proposed test, test_Ex4_AttackerCannotGrantSelfMinterRole, would call grantRole(MINTER_ROLE, attacker) while acting as attacker and expect the precise AccessControlUnauthorizedAccount error for DEFAULT_ADMIN_ROLE. It would then confirm that the attacker still lacks MINTER_ROLE. I chose this scenario because role escalation is a route around the protection demonstrated by the direct-mint test: if the attacker gains the role, they can create unbacked sUSD and may also gain the ability to burn other users’ balances. This test cannot protect against a genuinely compromised administrator, so production would additionally need multi-signature control, monitoring, and limits on minting.
