---
layout: CV
title: Curriculum Vitae
permalink: /CV
---
# Yashwanth Sai Sollu

**[Twitter](https://x.com/Yash_1208)** - **[Linkedin](https://www.linkedin.com/in/yashwanth-sai-sollu-009a50201/)** -
**[Github](https://github.com/yashwanth-supra)** - **[Blog](https://yashwanth1208.github.io/)** - **yashwanthsaisollu@gmail.com** 

Blockchain engineer building trustless cross-chain infrastructure. At Supra I build [Supranova](https://supranova.ai/), a light-client-based bridge connecting Ethereum, BSC and Supra that is live on mainnet: on-chain light clients and bridge contracts in Move and Solidity, and the Rust relayer stack that feeds them proofs.

---

## Work Experience
### Supra
**Blockchain Engineer** *Oct 2024 - Present*

Supranova is the industry's first Move-based trustless bridge on mainnet ([supranova.ai](https://supranova.ai/)). Instead of trusting a multisig or an off-chain committee, the destination chain verifies the source chain's consensus and the inclusion of each message directly on-chain.

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

**Token bridge (Move, Solidity)**
- Built the bidirectional token bridge (Ethereum/BSC to Supra and Supra to Ethereum) supporting native SUPRA, fungible assets and wrapped tokens
- Lock/release token vault, wrapped-token deployer with an on-chain v1 to v2 migration, and three verification tiers (finality, safe, optimistic) trading security for latency
- Per-token release and lock limits, minimum bridge amounts and pause guards on every permissionless entry point
- Worked on the Solidity contracts for the EVM side: upgradeable HypernovaCore with on-chain BLS12-381 committee verification, FeeOperator, TokenVault and WrappedTokenFactory, keeping message formats, fees and limits in parity with the Move implementation

**Fees and incentives**
- Implemented a cross-chain dynamic fee model using oracles and Uniswap V3 TWAP pricing, with price staleness checks
- Cross-decimal normalization between chains with overflow and precision-loss handling
- Tiered relayer and committee-updater rewards, and a protocol treasury

**Off-chain infrastructure (Rust)**
- Event listener that tracks bridge events on the source chain with a persistent cursor and publishes them to RabbitMQ
- Trustless relayer that generates and submits light-client proofs; anyone can run it. Concurrent batch processing with bounded parallelism, serialized ACK/NACK handling and graceful drain on shutdown
- Sync-committee updater that keeps the on-chain Ethereum light client in step with committee rotations
- Beacon block-roots indexer that backfills roots into SQLite per sync period and serves them over HTTP for historical proofs
- Productionized the 10-service Rust workspace: per-service configs, encrypted secrets, CLI, shared logging, task orchestration, build versioning and Docker/Jenkins deployment

**Hard-fork upgrades**
- Upgraded the bridge for Ethereum Pectra, Fusaka (proposer lookahead), EIP-7702 (type-4 transaction receipts) and Gloas/Glamsterdam (new gindices, proofs through the execution block hash, fork-aware signing domains)
- Upgraded for the BSC Fermi hard fork

**Security, testing and review**
- Went through the full audit lifecycle for all smart contracts, coordinating fixes and mitigations across Move and Solidity codebases
- Remediated findings including fund-release and decimal-precision issues, replay protection and proof-input validation
- Designed and ran testing pipelines with unit, mutation and end-to-end tests, and raised coverage across all core Move modules
- Conducted PR reviews of protocol design and implementation across Rust, Move and Solidity

**Research**
- Studied Ethereum consensus specifications, the sync-committee protocol, beacon chain architecture and the security models of trustless bridges to design the light clients
- Researched the consensus and finality of Avalanche, Sonic, BSC, Arc, Tempo and Plasma to assess light-client feasibility, leading to the BSC and Supra EVM integrations


### Amazon
**ML Summer School Mentee** *Sep 2023 - Oct 2023*
- Selected for Amazon ML Summer School 2023, a 4-week apprenticeship covering supervised and unsupervised learning, deep learning, sequential models and generative AI

## Talks
### BITS Pilani, Dubai Campus
**Guest Lecture** *May 2026*

Delivered an industry session on smart contract vulnerabilities for undergraduate students as part of the Industry Connect initiative for BITS463 Cryptography, integrating practical security insights with academic instruction.

### BINUS University, Indonesia
**Workshop** *Jul 2026*

Invited to deliver a Web3 security workshop for undergraduate students, covering blockchain security fundamentals, smart contract auditing concepts, common attack patterns, and industry best practices.

## Education
VNR Vignana Jyothi Institute of Engineering and Technology, Telangana, India *2020 – 2024*

*Bachelor of Technology, Computer Science and Engineering* **CGPA: 8.13/10**

## Technical Skills

**Languages:** Rust, Move, Solidity, Go, TypeScript, Python, C/C++

**Blockchain:** EVM, Move VM (Supra, Aptos), Solana, cross-chain interoperability, bridges, light clients, consensus (PoS, BFT, PoSA)

**DeFi:** AMMs (Uniswap), oracles and TWAP pricing, token bridges, fee and incentive design

**Cryptography:** BLS12-381, ECDSA, Ed25519, Merkle and Merkle-Patricia proofs, ZK fundamentals

**Security:** smart contract auditing and remediation, bridge threat modeling, replay protection, rate limiting

**Tools:** Foundry, Docker, RabbitMQ, SQLite, Jenkins, Git

**Web2 Frameworks:** Flutter, ReactJS, SQL


## Personal Projects
### XeroDrop~Share Securely [Published Paper – A Survey on Cryptography](https://thegrenze.com/index.php?display=page&view=journalabstract&absid=3114&id=8)
*Python, AES, Flutter, React*
- Built a secure document upload and printing platform for xerox shops, with AES-encrypted file transfer and integrated delivery
- Developed 3 Flutter apps (user, shopkeeper, delivery partner) and a React web dashboard for shopkeepers

## Achievements
- Secured 3rd place in IBC Continuum Web3.0 Hackfest, making the top 300 among 20,000+ participants
- Secured 1193rd position among 35,000+ participants in Google Code Jam (Round 3, 2021)

## Domains of interest
- Blockchain Interoperability
- Light Clients and Consensus
- Security Research
- Smart Contract Auditing
- DeFi
- Zero Knowledge Proofs
