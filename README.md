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

## Private reviews

| protocol | scope | found | report |
| ---- | ---- | ----- |---------|
| [Grappa Finance](https://www.grappa.finance/) | Full Collateral Engine | 1C, 2M |[report](https://github.com/Delvir0/Audits/blob/main/Grappa%20Finance/report.md)
| [Wassie-Racing](https://wassieracing.com/) | Game Contracts | 1C, 2H, 1M |[report](https://github.com/Delvir0/Audits/blob/main/Wassie-Racing/report.md)

# SpearBit reviews
adding...

## Contests

**Scope:** This is the scope **I** audited, not the whole codebase

| protocol | scope | H/M found | place | report |
| ---- | ---- | ---------| ---- | ---- | 
| Hubble Exchange | InsuranceFund.sol, OrderBook.sol | 1H, 1 M | 14/148 | [report](https://audits.sherlock.xyz/contests/72/report)
| Bond Options | OTLM.sol | 1 M | 15/153 | [report](https://audits.sherlock.xyz/contests/99/report) 
| Dinari | full contest | 1 M | 16/176 | [report](https://audits.sherlock.xyz/contests/98/report) 
| Rubicon V2 | RubiconMarket.sol | 2 H | 101/179 | [report](https://github.com/Delvir0/Grappa-Finance/blob/main/GrappaFinance-FullCollateralEngine-report.md.pdf)
