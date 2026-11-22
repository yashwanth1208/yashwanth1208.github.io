---
layout: CV
title: Curriculum Vitae
permalink: /CV
---
# Yashwanth Sai Sollu

**[Twitter](https://x.com/Yash_1208)** - **[Linkedin](https://www.linkedin.com/in/yashwanth-sai-sollu-009a50201/)** -
**[Github](https://github.com/yashwanth1208)** - **[Blog](https://yashwanth1208.github.io/)** - **yashwanthsaisollu@gmail.com** - **India (Remote)**

---

## Work Experience
### Supra
**Blockchain Engineer** *Oct 2024 - Present*

Supranova is the industry's first Move-based trustless bridge on mainnet ([supranova.ai](https://supranova.ai/)). Instead of trusting a multisig or an off-chain committee, the destination chain verifies the source chain's consensus and the inclusion of each message directly on-chain. For chains where that is not yet feasible, a multisig-attested bridge extends Supra's reach to L2s.

**Hub-and-spoke messaging layer (in progress)**
- Building a hub-and-spoke architecture with Supra as the hub and Ethereum, BSC and Supra EVM as spokes
- Payload-agnostic message publisher with sequential message IDs and per-destination enablement, so services beyond the token bridge can build on the same transport
- Unified inbound message verifier shared across every source chain, with replay protection across both terminal and relay flows and per-chain expected event signatures
- Spoke-to-spoke relayer that routes messages through the hub, with the first hop proven by the source chain's light client (Ethereum finality, safe and optimistic modes, and BSC fast finality)
- Pluggable per-chain light clients and proof verifiers, a shared treasury and per-chain fee configuration in the core

**On-chain light clients (Move)**
- Ethereum sync-committee light client: processes beacon-chain light-client updates and sync-committee rotations, and verifies historical block-root proofs and receipt/trie (Merkle-Patricia) proofs
- BSC light client: parses Parlia header extra-data, verifies BLS fast-finality votes and ECDSA seals, and tracks epoch-based validator committee rotation with pruning and turn-length checkpoints
- Supra EVM light client and the Supra committee manager for the Supra to Ethereum direction, verifying Supra events and transaction inclusion

**Multisig-attested bridge (L2s)**
- Adapted a multisig-attested bridge connecting Base, Arbitrum, Optimism and Robinhood Chain to Supra, covering chains where light-client or state-commitment verification is not feasible
**Token bridge (Move, Solidity)**
- Built the bidirectional token bridge (Ethereum/BSC to Supra and Supra to Ethereum) supporting native SUPRA, fungible assets and wrapped tokens
- Lock/release token vault, wrapped-token deployer with an on-chain v1 to v2 migration, and three verification tiers (finality, safe, optimistic) trading security for latency
- Per-token release and lock limits, minimum bridge amounts and pause guards on every permissionless entry point
- Developed the EVM-side Solidity contracts: upgradeable HypernovaCore with on-chain BLS12-381 committee verification, FeeOperator, TokenVault and WrappedTokenFactory, keeping message formats, fees and limits in parity with the Move implementation

**Fees and incentives**
- Implemented a cross-chain dynamic fee model using oracles and Uniswap V3 TWAP pricing, with price staleness checks
- Cross-decimal normalization between chains with overflow and precision-loss handling
- Tiered relayer and committee-updater rewards, and a protocol treasury

**Off-chain infrastructure (Rust)**
- Event listener that tracks bridge events on the source chain with a persistent cursor and publishes them to RabbitMQ
- Permissionless relayer that generates and submits light-client proofs; anyone can run it. Concurrent batch processing with bounded parallelism, serialized ACK/NACK handling and graceful drain on shutdown
- Sync-committee updater that keeps the on-chain Ethereum light client in step with committee rotations
- Beacon block-roots indexer that backfills roots into SQLite per sync period and serves them over HTTP for historical proofs
- Productionized the 10-service Rust workspace: per-service configs, encrypted secrets, CLI, shared logging, task orchestration, build versioning and Docker/Jenkins deployment

**Hard-fork upgrades**
- Kept the light clients and bridge compatible across Ethereum hard forks: Pectra, Fusaka (proposer lookahead), EIP-7702 (type-4 transaction receipts) and Gloas/Glamsterdam (new gindices, proofs through the execution block hash, fork-aware signing domains)
- Upgraded the BSC light client for the Fermi hard fork

**Security, testing and review**
- Took all Move and Solidity contracts through the full audit lifecycle, from audit prep through remediation to mainnet release
- Remediated findings including fund-release and decimal-precision issues, replay protection and proof-input validation
- Designed and ran testing pipelines with unit, mutation and end-to-end tests, and raised coverage across all core Move modules
- Reviewed PRs for protocol design and implementation across Rust, Move and Solidity

**Research**
- Studied Ethereum consensus specifications, the sync-committee protocol, beacon chain architecture and the security models of trustless bridges to design the light clients
- Researched the consensus and finality of Avalanche, Sonic, BSC, Arc, Tempo and Plasma to assess light-client feasibility, leading to the BSC and Supra EVM integrations

## Talks
### BINUS University, Indonesia
**Workshop** *Jul 2026*

Invited to deliver a Web3 security workshop for undergraduate students, covering blockchain security fundamentals, smart contract auditing concepts, common attack patterns, and industry best practices.

### BITS Pilani, Dubai Campus
**Guest Lecture** *May 2026*

Delivered an industry session on smart contract vulnerabilities for undergraduate students as part of the Industry Connect initiative for BITS463 Cryptography, integrating practical security insights with academic instruction.

## Writing and Publications
- [OZ's Move-over CTF](/blog/move-over-ctf-writeup) - write-up of OpenZeppelin's Sui Move security CTF, including how Sui Move differs from Aptos Move
- [The 13 Seconds Ethereum Bridge](/blog/the-13-seconds-ethereum-bridge) - what Ethereum's Fast Confirmation Rule means for bridges that wait ~13 minutes for finality
- [A Survey on Cryptography](https://thegrenze.com/index.php?display=page&view=journalabstract&absid=3114&id=8) - published paper

## Technical Skills

**Languages:** Rust, Move, Solidity, Go, TypeScript, Python, C/C++

**Blockchain:** EVM, Move VM (Supra, Aptos, Sui), Solana, cross-chain interoperability, bridges, light clients, consensus (PoS, BFT, PoSA), L1s and L2s

**DeFi:** AMMs (Uniswap), oracles and TWAP pricing, token bridges, fee and incentive design

**Cryptography:** BLS12-381, ECDSA, Ed25519, Merkle and Merkle-Patricia proofs, ZK fundamentals

**Security:** smart contract auditing and remediation, bridge threat modeling, replay protection, rate limiting

**Backend and infrastructure:** async Rust (tokio, alloy), RabbitMQ, Redis, PostgreSQL, MySQL, SQLite, Docker, Kubernetes, AWS/GCP, Jenkins, Git

**Tools:** Foundry

## Education
VNR Vignana Jyothi Institute of Engineering and Technology, Telangana, India *2020 - 2024*

*Bachelor of Technology, Computer Science and Engineering* **CGPA: 8.13/10**

## Achievements
- Placed 3rd in the IBC Continuum Web3.0 Hackfest after reaching the top 300 of 20,000+ participants
- Ranked 1193 among 35,000+ participants in Google Code Jam 2021
- Selected for Amazon ML Summer School 2023, a 4-week program covering supervised and unsupervised learning, deep learning, sequential models and generative AI

## Domains of interest
- Blockchain Interoperability
- Light Clients and Consensus
- Security Research
- Smart Contract Auditing
- DeFi
- Zero Knowledge Proofs
