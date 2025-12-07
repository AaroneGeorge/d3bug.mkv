# One Bug a Day

A curated collection of smart contract vulnerabilities, exploits, and security findings from various platforms and audits. This repository documents real-world bugs with detailed analysis, mathematical proofs, and remediation strategies.

## Introduction

This repository serves as a learning resource for smart contract security auditing. Each bug is thoroughly analyzed with:

- **Vulnerability Description**: Clear explanation of the issue
- **Attack Vector**: How the vulnerability can be exploited
- **Mathematical Analysis**: Step-by-step calculations and profit formulas
- **Code Examples**: Vulnerable code and safe replacements
- **Remediation**: Recommended fixes and best practices

## Naming Convention

Files follow the format: `[AttackVector]-[Severity]-[Platform].md`

- **AttackVector**: The type of attack (e.g., MEV, Reentrancy, AccessControl)
- **Severity**: Bug severity level (e.g., MEDIUM, HIGH, CRITICAL)
- **Platform**: Source platform (e.g., CODE_ARENA, CODE4RENA, AUDIT)

---

## Table of Contents

- [Attack Vectors](#attack-vectors)
  - [MEV (Maximal Extractable Value)](#mev-maximal-extractable-value)
- [Severity](#severity)
  - [Critical](#critical)
  - [High](#high)
  - [Medium](#medium)
  - [Low](#low)
- [Platforms](#platforms)
  - [Code Arena](#code-arena)
  - [Sherlock](#sherlock)

---

## Attack Vectors

### MEV (Maximal Extractable Value)

> MEV vulnerabilities involve opportunities for extracting value through transaction ordering, front-running, or arbitrage.

- **[MagicLpAggregator Vulnerability](./Reports/MEV-MED-CODEARENA.md)** - Incorrect pricing leading to arbitrage loss

<!-- Add more attack vectors below:
### Reentrancy
- [Vulnerability Name](./FILE-NAME.md) - Description

### Access Control
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

---

## Severity

### Critical

> Can lead to complete loss of funds or protocol shutdown.

<!-- Add Critical severity vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

### High

> Significant financial impact or protocol disruption.

<!-- Add High severity vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

### Medium

> Moderate impact, exploitable under certain conditions.

- **[MagicLpAggregator Vulnerability](./Reports/MEV-MED-CODEARENA.md)** - Incorrect pricing leading to arbitrage loss

<!-- Add more Medium severity vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

### Low

> Minor impact or requires unlikely conditions.

<!-- Add Low severity vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

---

## Platforms

### Code Arena

- **[MagicLpAggregator Vulnerability](./Reports/MEV-MED-CODEARENA.md)** - Incorrect pricing leading to arbitrage loss

<!-- Add more Code Arena vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

### Sherlock

<!-- Add Sherlock vulnerabilities here:
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

<!-- Add more platforms below:
### Code4rena
- [Vulnerability Name](./FILE-NAME.md) - Description

### Hats Finance
- [Vulnerability Name](./FILE-NAME.md) - Description
-->

