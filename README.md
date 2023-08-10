# Audit methodology
In the dynamic world of Smart Contract auditing, it is essential to be equally precise and dynamic.\
I've created an audit methodology consisting of four stages, ensuring a comprehensive codebase examination.

### Stage 1: Codebase Understanding
Thoroughly analyse the codebase, constructing a comprehensive mental framework. Essentially laying the groundwork for a clear understanding of the protocol's architecture.

### Stage 2: Insight Amplification
Following up on Stage 1, refining the insights gained during the analysis. By revisiting and refining initial observations, aspects that may seem off, strike as not logical or possible attack entries are diligently pursued.

### Stage 3: Unveiling Vulnerabilities
Proactively seeking out weaknesses, basically poking at area's where "it would be really bad if this would (intentionally) break". This is done manually and by using tools (e.g. fuzzing). Usually, this is the part where the most severe issues arise.

### Stage 4: Harnessing Collective Creativity
Gain insights from the wider community by reading recent audits of similar protocols (such as on solodit.xyz). This infuses fresh perspectives and innovative insights into the audit, boosting its effectiveness.

# Audit reports

Provides reports of all performed audits.

## Private audits

| protocol | scope | report |
| ---- | ---- | ---------|
| [Grappa Finance](https://www.grappa.finance/) | Full Collateral Engine | [report](https://github.com/Delvir0/Grappa-Finance/blob/main/GrappaFinance-FullCollateralEngine-report.md.pdf)


## Contests

**Scope:** This is the scope **I** audited, not the whole codebase\
**AASI:** Average Audit Score Index (see page end for explanation)

| protocol | scope | H/M found | place | AASI | report |
| ---- | ---- | ---------| ---- | ---- | ---------|
| Hubble Exchange | InsuranceFund.sol, OrderBook.sol | 1H, 1 M | 14/148 | 57 | [report](https://audits.sherlock.xyz/contests/72/report)
| Bond Options | OTLM.sol | 1 M | 15/153 | 20 | [report](https://audits.sherlock.xyz/contests/99/report) 
| Dinari | full contest | 1 M | 16/176 | 20 | [report](https://audits.sherlock.xyz/contests/98/report) 
| Rubicon V2 | RubiconMarket.sol | 2 H | 101/179 | TBD | [report](https://github.com/Delvir0/Grappa-Finance/blob/main/GrappaFinance-FullCollateralEngine-report.md.pdf)

### AASI
A score from 0-100 which indicates how one performed (efficiency) based on the amount of contracts (sloc) audited vs the amount of issues found and sloc of the whole contest codebase.\
**NOTE:** a score of 100 means you also found all the issues everyone else found! Hence scoring 100 is HARD.

**Why use AASI?**
1. Some, like me, have limited time to audit in contests since we can't decide when it takes place. This results in not being able to audit the whole codebase.
While being potentially effective, this is not reflected in the ranking.
2. Using the AASI gives insight in how well you performed (efficiency rate)

**Example**
- Codebase to audit is 4 contracts of 500 sloc each. In total, 5 high (2 points per high) and 10 medium (1 point per medium) issues were found.
- I audited only 1 contract (500 nsloc) and found 1 high and 1 medium (3 points)
- `3 / 500 = 0.006`, `((5 * 2) + (5 * 10)) / 2000 = 0.01`
- `(0.006 / 0.01) * 100 = 60`
- This means you have an efficiency of 60 (60%)
