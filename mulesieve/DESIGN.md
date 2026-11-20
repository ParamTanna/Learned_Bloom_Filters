# MuleSieve: design note

*Param Tanna and Heet Panchal, IIT Gandhinagar. Working draft, November 2026.*

This note explains how the learned Bloom filter work in this repository is being extended to a
payments problem: screening a payee against India's mule-account intelligence at the moment of
payment, without exposing the flagged list.

## The problem in one paragraph

Several central systems in India now hold mule-account data: the I4C Suspect Registry (about
32 lakh flagged accounts by June 2026), RBI's DPIP Smart Registry, DoT's Financial Fraud Risk
Indicator and RBIH's MuleHunter.AI. Their reach is uneven (MuleHunter.AI with about 26 banks,
the DPIP pilot with 8), and the public documentation we found describes raw identifiers being
pushed in and queried out. From 1 January 2027, RBI's amended customer-protection directions put
the burden of proof on banks and make beneficiary banks share compensation. That gives every bank a
reason to check payees, and to warn customers precisely.

## Why the obvious approaches don't work

**Sharing a hashed list.** Most UPI identifiers are phone numbers. There are about 10^10 ten-digit
numbers, and hashing all of them is minutes of work on a laptop. Hagen et al. (NDSS 2021) and
Signal's 2017 contact-discovery post both show hashed phone numbers being reversed this way. So a
hashed list leaked from any one of hundreds of institutions is the whole list. That includes
fraudsters who want to know whether their own mules have been flagged.

**A learned Bloom filter over account identifiers.** This was our first idea, and the results in this
repository rule it out. In the URL experiments (notebooks 03 to 05), the oracle reached AUC 0.97 on
real URLs. Rerun on random strings, it fell to AUC 0.54, and the partitioned filter became 2 to 6
per cent *worse* than a plain Bloom filter. Once identifiers are passed through a keyed function to
protect them, they look exactly like those random strings. There is nothing left in the identifier
for a model to learn from.

So the learning has to move somewhere else. In payments, the place it fits is the **cost of
mistakes**.

## Design

```
 flagged identifiers ---> keyed PRF (threshold key, t-of-n holders) ---> tiered filters + exception list
 (registry, bank models,                                                          |
  NCRP-linked complaints)                                                  distributed to every
                                                                           bank / PSP switch
 payment time:
 payee ID --blind--> OPRF holders --evaluated--> unblind --> lookup in local filters --> tier --> action
```

1. **Inputs and tiers.** Each flagged identifier (account plus IFSC, VPA, mobile) carries a
   confidence level: confirmed (linked to an NCRP complaint), suspected (flagged by the holding
   bank's own behavioural model), or registry-only. These become the tiers. The behavioural model
   (account age, fan-in, pass-through ratio, device churn) is where ordinary machine learning still
   belongs, at the bank that holds the account and can see that behaviour.
2. **Keyed function.** Identifiers pass through an oblivious PRF (RFC 9497) whose key is split
   among several holders, so no single party can evaluate it alone. Without the key, a filter is
   useless to whoever copies it.
3. **Tiered filters with cost-aware sizing.** There is one filter per tier. A legitimate payee
   checked against tier *t* is wrongly flagged with probability *f_t*, at a friction cost *c_t*.
   A false "hold" costs far more than a false "warn". We minimise expected friction, the sum of
   c_t * f_t, under a total memory budget, where a tier with n_t keys costs about 1.44 * n_t * log2(1/f_t)
   bits. Setting the derivative to zero gives **f_t proportional to n_t / c_t**: cheap, crowded tiers
   get looser filters and expensive tiers get tighter ones. This is the same water-filling idea the
   partitioned learned Bloom filter uses for its score bands, applied to tiers.
4. **Exception list for popular payees.** Payment traffic is extremely skewed. One false positive
   on a large merchant would add friction to lakhs of payments a day. The builder holds the key, so
   it can test the most-paid legitimate payees (large merchants, utilities, government accounts)
   against the filters and keep the few that collide in a small exact exception list. This follows
   the stacked-filter idea of Deeds, Hentschel and Idreos (PVLDB 2021) of learning from known
   negatives.
5. **Query path.** The paying bank's switch blinds the payee identifier, gets it evaluated by the
   key holders, unblinds it and checks the local filters. Queries are needed only for first-time or
   high-value payees. They are rate-limited and logged per institution.
6. **Response.** Tier to action: pass / a specific warning that states the reason / a cooling-off
   period / a hold for review. RBI's 2027 directions treat ignoring a "specific, directed and clear"
   bank warning as customer negligence, which is why the warning tier matters.

The closest technical precedent is Kales et al. (USENIX Security 2019), who combined an OPRF with a
client-side filter for private contact discovery. MuleSieve adapts that pattern to payments and adds
the tiers, the cost-aware sizing and the exception list.

## Threat model

| Threat | Plain or hashed list | MuleSieve |
|---|---|---|
| Filter copied or leaked from one institution | Whole list recoverable by enumeration | Useless without the key |
| Insider bulk-checks whether mules are flagged | Unlimited offline checks | Every check is an online, rate-limited, logged query |
| Central party learns who pays whom | Sees every payee queried | Key holders see blinded values only |
| Crafted inputs that inflate false alarms (a limitation we noted in the URL project) | Possible offline | Each probe costs a logged query |
| Wrongly flagged account must be cleared (courts have ordered this) | Copies cannot be recalled | Rebuild and rotate the key; old copies stop matching |

## Evaluation plan (in progress)

- **Data:** a synthetic multi-bank payment network with mule rings, starting from IBM's AMLworld
  synthetic transactions and published mule typologies. Payee popularity follows a heavy-tailed
  distribution.
- **Measurements:**
  - memory per lakh flagged identifiers;
  - expected friction on legitimate payments versus an equal-memory plain Bloom filter;
  - OPRF latency and throughput per CPU core, compared against UPI's average of about 9,500
    transactions per second (25.47 bn in October 2026);
  - time to reverse a plain hashed list of phone-number identifiers, versus ours.
- **Baseline:** single-bank lists against a pooled MuleSieve list, measured as detection lift on
  the synthetic network.

## Open questions

- Who should hold the key shares (RBIH or IDPIC, NPCI, a bank-consortium node), and whether that
  is legally workable.
- Where the lookup sits in practice: the bank's UPI switch, the PSP app's backend, or both.
- DPDP: section 17(1)(c) largely exempts processing for fraud prevention, so we frame privacy as a
  security safeguard under section 8(5), not as a compliance requirement.

## Status

| Done | In progress |
|---|---|
| Bloom filter, learned BF, sandwiched and partitioned LBF from scratch, honest memory accounting, zero-false-negative validation, random-key ablation (this repository) | Synthetic payment network, keyed filter with tiers and the exception list, the measurements above |
