# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:In Lab 1, totalCollateral() == totalSupply() worked because the collateral was itself an on-chain token — the chain could verify both sides of the equation by itself. In Lab 2 that is no longer possible, because the asset is off-chain and only a reporter can tell the chain what exists.The closest we can write is something like custodian.realHoldings() >= vault.totalClaimValue(), i.e. the real asset should be at least as large as the on-chain claim. The direction matters: shares should never exceed the real holdings, otherwise the extra claims have nothing behind them. But even this weaker inequality cannot be proven on-chain. realHoldings() is just a number a permissioned address can move — it is a report, not a balance.Ex2.1 makes this concrete: Alice subscribes, tBill.balanceOf(alice) = 1000e18, yet custodian.realHoldings() = 0. The inequality is broken the moment it is written, and nothing on-chain reacts. So the equation does not assert that the asset exists — it only asserts that the reporter says it does.

<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:An SPV is basically an empty shell company set up for one purpose — in this case, to hold the T-Bills. It is like not putting all your eggs in one basket: the issuer could have held the T-Bills itself, but if it goes bankrupt, the court would seize everything under its name to pay creditors, including the T-Bills, and the token holders would be left with nothing. By putting the T-Bills inside a separate SPV, the assets are legally split from the issuer's own assets, so the court can only touch the issuer's things and the T-Bills stay safe — this is what bankruptcy-remote means. The token holder does not literally own the T-Bills; what they hold is a claim against the SPV, enforced through legal documents, not through the token itself. That is why even perfectly written on-chain code still needs this legal structure: the code lives in the virtual world, but the asset lives in the real one, and property separation is a legal job, not a technical one. If the SPV's custodian goes bankrupt, the token holder still has a legal claim and would have to go through the courts to recover the T-Bills or some compensation.

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:A single trusted reporter can fail in several ways. It can be greedy or negligent and silently move the NAV to benefit itself, for example by re-pricing the book before a large redemption. Its software or algorithm can have a bug and report a wrong number. Its private key can leak, letting an attacker forge an attestation. Or it can simply be slow and not update the NAV at all, which distorts subscription pricing, redemption payouts, and the two invariants. To mitigate these, I would not rely on one reporter. I would use several independent reporters and cross-check their submissions, similar to a Byzantine generals setup — if one number disagrees with the rest, it is rejected and the honest majority prevails. To keep them honest, I would add a staking mechanism like proof-of-stake: reporters post a bond, and if they are caught lying or going offline too long, the bond is slashed and they are replaced. For pure software bugs, technical staff should be able to patch the contract or the feed, and there should be a staleness check so a frozen NAV pauses the system instead of being trusted forever. Finally, legal contracts and liability should sit behind the role, so a malicious reporter faces consequences off-chain as well.


<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:A stale NAV is different from a wrong one: the reporter is not lying, it just has not updated the number yet. Suppose the true value has moved to 1.05 but the on-chain NAV is still 1.00. Anyone who knows the real value has changed can buy shares at the old price, because the shares are being priced below what they are actually worth. When the reporter finally updates the NAV to 1.05, those shares are suddenly worth more and the buyer profits at the expense of the holders who did not act. If the true value falls to 0.95 instead, the same person does the opposite — they redeem at the stale 1.00 before the NAV is corrected, leaving the remaining holders with the loss. This is not really a hacking attack; it is an information advantage. Insiders, market makers, or anyone watching the off-chain T-Bill market closely will see the move first. Adding a staleness check on the feed would help — if the NAV has not been updated for too long, the system should pause rather than keep trading on an old number — but it does not fully remove the risk, because the attacker can still act in the window before the check triggers.



<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:If the NAV rises after Alice enqueues, she is locked into the lower price and only receives 1000 USDC even though her shares are now worth 1250 — the extra 250 stays with the holders who did not redeem. If the NAV falls instead, the opposite happens: Alice still gets 1000 while the remaining holders absorb the loss. So the locked price shifts risk onto whoever is on the other side of the trade, and it is not obvious that this is fair. The redeemer gets certainty but gives up upside; the stayers keep the upside but also carry the downside. I do not think one side should automatically bear all of it. A cleaner design would be to settle at the average of the NAV at enqueue and the NAV at settle, so both sides share the price move over the T+1 window. Even better, give the redeemer a choice at enqueue time: lock the price now for certainty, or accept the settlement-time NAV for a potentially better payout. Each option has its own risk, and the redeemer picks which one to take, so the risk is allocated by consent rather than by default.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:I think USDC handled it better in 2023 than the money-market fund did in 2008. In 2008 the fund froze redemptions entirely, which protected the fund itself but hurt investors and damaged trust — people could not get their money out. USDC slowed redemptions but did not close them, which gave Circle time to recover the SVB deposits and eventually restored the peg. If Circle had failed to get the money back, USDC would probably have collapsed completely. For a tokenized T-Bill fund, "closing the redemption channel" means you can still enqueue but settle stops working, so the queue just keeps growing and the people at the back wait indefinitely. If new users cannot enqueue and old tickets cannot be paid, that is effectively a default, because the promise to redeem is broken. The pause() lever in this lab freezes transfers, minting, and redemption all at once, which I think is the wrong tool. During a liquidity squeeze you want to slow the exit, not lock everyone in. If only redemption were frozen but transfers still worked, users could at least trade their shares in a secondary market. If I had to pick one, I would freeze transfers before redemption, because fully freezing redemption looks like a default and destroys confidence. The better design is modular pausing: separate switches for transfers, minting, and redemption, so the issuer can respond to the actual problem instead of shutting everything down.

<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:I think KYC is only the first filter. Passing it should let a normal user hold shares, but for a regulated fund I would add a second layer: sanctions screening, politically exposed persons, and some kind of abnormal-behaviour check. For example, if two accounts keep trading with each other at a price far from everyone else's, that looks like wash trading or layering and should be flagged. The line should be clear enough that an ordinary user who has finished KYC is not blocked, but flexible enough that clearly suspicious activity can be stopped. On who controls the list, I do not think one person should hold the switch, because that just recreates a centralised authority on-chain. I would split it across three parties — the custodian, an auditor, and the regulator — with one or two representatives each, so decisions need joint agreement. That is slower than a single admin but much harder to abuse. If the whitelist is misused to freeze someone unfairly, the regulator, an auditor, and ultimately a court should be able to review it, and the affected user should have a legal path to appeal. The list should be strict enough to satisfy compliance, but not so broad that it quietly becomes a censorship tool.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:A regulated fund needs both powers for legitimate reasons: it may have to freeze an address during an investigation, or seize shares after a court order or a sanctions ruling, and without those tools it cannot comply with the law. But the same powers can easily become a one-man switch, where the issuer freezes or burns whoever it dislikes, and then nobody holding the token feels safe. That is the tension Ex4.3 exposes: once an address is off the whitelist, burn() reverts too, so a freeze accidentally blocks a seizure — the two tools collide instead of working together. My safeguards would be multi-signature approval, a time limit on freezes, and a requirement that a seizure needs a court order or the custodian's consent. A frozen user should also have a legal appeal path through the regulator, an auditor, or the courts. On who holds each key, I would let the issuer initiate a freeze but only after notifying the regulator and the custodian and getting their agreement; in an emergency it could act first and report afterwards. Seizure should sit with the regulator or a court, again with notification to the other parties. The two powers should use different keys, so a leak of one does not hand over both, and the contract should separate the freeze check from the burn check so a freeze does not silently disable a lawful seizure.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | |test_Ex2_SharesExistWhileTheCustodianHoldsNothing
| a | `realHoldings()` moves with no cash moving | |test_Ex2_RealHoldingsIsJustANumber
| b | The claim value floats while the share count stays | |test_Ex3_TheClaimFloats_TheShareCountDoesNot
| b | One `attest` re-prices every holder at once | |test_Ex3_OneCallMovesTheWholeBook
| d | A non-whitelisted address cannot receive shares, even by paying | |test_Ex4_NoKyc_NoShares_EvenIfYouPay
| d | A whitelisted holder cannot send shares to a non-whitelisted address | |test_Ex4_TransferChecksBothEndpoints
| d | Removing an address freezes that holder — and blocks a burn | |test_Ex4_FreezeBeatsConfiscation

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:The scenario I think is most likely to be exploited is a custodian reporting fake holdings — realHoldings() writes a large number while nothing is actually there off-chain. This breaks plank (a) custody, and I think it is the most dangerous because it works like a Ponzi scheme: the fund claims it has enough assets to back every share, attracts new investors on that promise, and uses the new money to pay earlier redeemers. As long as new money keeps coming in, everyone looks fine, but the moment subscriptions slow down the whole thing collapses, and the last holders are left with nothing. What makes it worse is that no hacking is required. The custodian just types a number into a permissioned function, exactly like Ex2.1 shows — Alice gets 1000e18 shares while custodian.realHoldings() is still 0. The chain cannot check the number against anything real, so the fake report is accepted as truth. Unlike a wrong NAV, which can at least be cross-checked against multiple reporters, a fake custody report has no on-chain counterweight. The only real defence is off-chain: independent audits, a regulated custodian with legal liability, and multiple parties verifying the same holdings. On-chain alone, the bridge has no way to see the box.
