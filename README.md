# Cumulo: Limonata Infrastructure

This repository documents the infrastructure, configuration, and operational practices used by **Cumulo** to run and maintain a **Limonata** (`limonata_10777-1`, EVM chain-id `10777`) node and validator.

The goal of this repository is to provide **transparent, auditable, and continuously updated documentation** of our real-world operations as a professional validator and infrastructure provider.

This is not a generic tutorial.
Everything documented here reflects **how we actually operate the node in production**.

---

## 🧭 Scope

This repository covers:

- Deployment of a Limonata full node and validator
- Node configuration and pruning strategy
- Systemd-based service management
- Validator setup and operational policies (including ValGrant)
- Monitoring and observability
- Upgrade and maintenance procedures
- Security and key-management practices
- Ongoing operational changes and upgrades

---

## 🌐 Network

- **Network:** Limonata
- **Chain ID:** `limonata_10777-1`
- **EVM Chain ID:** `10777`
- **Denom:** `aLIMO`
- **Client:** `limonatad`
- **Type:** EVM Layer 1 on Cosmos SDK (single-slot ~0.3s finality target, gasless UX)

---

## 🧑‍🚀 Cumulo as an Operator

Cumulo is a multi-chain infrastructure operator running production-grade validators and nodes across multiple ecosystems.

Our focus as an operator is:

- Security-first infrastructure
- High availability and operational reliability
- Transparent processes
- Long-term commitment to the networks we support
- Public documentation of real operational practices

This repository is part of that commitment.

---

## 📁 Repository Structure

```
limonata_10777-1/
├─ README.md              # Network-specific overview
├─ install/                # Node installation and configuration
├─ validator/              # Validator setup and policies
├─ operations/             # Upgrades, backups, incident handling
├─ monitoring/             # Metrics and observability
├─ configs/                # Example configuration files
├─ systemd/                # Service definitions
└─ scripts/                # Operational helper scripts

data/
└─ validators_testnet.json # Community validator resources (RPC, explorers, state-sync, etc.)
```
