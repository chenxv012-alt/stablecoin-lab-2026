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

<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:

<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:

<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:

<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:

<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts | |
| Transfers revert while paused | |
| **Redemption** reverts while paused | |
| An attacker cannot burn someone else's balance | |
| ...but the vault holding `MINTER_ROLE` can | |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
