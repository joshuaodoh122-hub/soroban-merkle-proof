# 🌳 soroban-merkle-proof

**Verify Merkle proofs inside Soroban smart contracts.**

A small `no_std` Rust crate that lets a contract store one 32-byte Merkle root and check that a piece of data belongs to the set behind it, instead of storing the whole set on-chain.

[![Rust](https://img.shields.io/badge/Rust-no__std-orange?logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Stellar](https://img.shields.io/badge/Stellar-Soroban-7D00FF?logo=stellar&logoColor=white)](https://soroban.stellar.org)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-early%20development-orange)](#-status)

[Why](#-why) · [Design](#-design) · [Usage](#-usage-planned) · [Security](#-security-notes) · [Status](#-status)

---

> **📝 Note:** this project is in **early development**. The design below is the intended behavior. Anything marked *(planned)* is not implemented yet, and the [status table](#-status) shows what exists today. No gas or performance claims are made until they are measured.

## ✨ Why

Whitelists, airdrop allocations and registries can have thousands of entries. Storing each one in contract storage costs ledger fees for every entry. With a Merkle tree you store a single root, and each user submits a short proof that their entry is part of it.

This crate is the reusable verification piece. It is meant to be imported by contracts such as airdrops, allowlists and claim pages, rather than being an application itself.

- 🔒 **Domain-separated hashing**: leaves and internal nodes are hashed with different prefixes, so an internal node can't be passed off as a leaf
- 🧭 **Sorted pairs**: no left/right flags in proofs, so proofs are just a list of hashes
- 🧱 **`no_std`**: built for the Soroban WASM environment
- 🔍 **Small and auditable**: one verification function, documented hashing rules, and shared test vectors

## 📐 Design

All hashing is SHA-256, computed through the Soroban host (`env.crypto()`).

```
leaf  = SHA256( 0x00 || data )
node  = SHA256( 0x01 || min(a, b) || max(a, b) )
```

- `data` is the raw bytes of one entry, for example an encoded address and amount. The **verifier hashes it**, so callers never submit a pre-hashed leaf.
- `min` and `max` compare the two 32-byte children as byte strings, so a proof needs no direction information.
- A proof is a `Vec<BytesN<32>>` of sibling hashes from the leaf up to the root.
- When a level has an odd number of nodes, the unpaired node is carried up unchanged. It is not duplicated.
- Proofs longer than a fixed maximum depth are rejected *(planned, limit to be decided)*.

The off-chain builder must use exactly the same rules. Generic libraries such as `merkletreejs` do not add the `0x00`/`0x01` prefixes by default, so their roots and proofs will not verify here unless configured to match. A matching builder is planned so you don't have to wire this up yourself.

## 📦 Installation

Until the crate is published, depend on it from GitHub:

```toml
[dependencies]
soroban-merkle-proof = { git = "https://github.com/joshuaodoh122-hub/soroban-merkle-proof" }
```

A crates.io release is planned once the crate has tests and a tagged version.

## 🚀 Usage *(planned)*

### 1. Build the tree off-chain *(planned)*

A builder will take your entries, apply the hashing rules above, and print the root plus a proof for each entry:

```
merkle-builder entries.csv  →  root + proofs.json
```

The root is what you store in your contract. Each user gets their own proof.

### 2. Verify inside a contract

The signature below is the intended API. Treat the example as illustrative: it has not been compiled or tested yet.

```rust
pub fn verify(
    env: &Env,
    proof: &Vec<BytesN<32>>,
    root: &BytesN<32>,
    data: &Bytes,
) -> bool
```

A claim contract must **store the root itself** and must **record claims** so an entry can't be used twice:

```rust
#![no_std]
use soroban_merkle_proof::verify;
use soroban_sdk::{
    contract, contractimpl, contracttype, xdr::ToXdr, Address, BytesN, Env, Vec,
};

#[contracttype]
enum DataKey {
    Admin,
    Root,
    Claimed(BytesN<32>), // keyed by the hash of the entry
}

#[contract]
pub struct ClaimContract;

#[contractimpl]
impl ClaimContract {
    pub fn init(env: Env, admin: Address, root: BytesN<32>) {
        if env.storage().instance().has(&DataKey::Admin) {
            panic!("already initialized");
        }
        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(&DataKey::Root, &root);
    }

    pub fn claim(env: Env, claimant: Address, amount: i128, proof: Vec<BytesN<32>>) {
        claimant.require_auth();

        // The root comes from storage, never from the caller.
        let root: BytesN<32> = env.storage().instance().get(&DataKey::Root).unwrap();

        // The entry binds the claimant and the amount, so a proof is only
        // valid for that exact pair.
        let data = (claimant.clone(), amount).to_xdr(&env);

        if !verify(&env, &proof, &root, &data) {
            panic!("invalid proof");
        }

        // Stop the same entry from being claimed twice.
        let key = DataKey::Claimed(env.crypto().sha256(&data).to_bytes());
        if env.storage().persistent().has(&key) {
            panic!("already claimed");
        }
        env.storage().persistent().set(&key, &true);

        // ...transfer `amount` to `claimant` here.
    }
}
```

The off-chain builder has to encode each entry to the same bytes the contract builds with `to_xdr`. A mismatch in encoding is the most common reason a valid-looking proof fails, so the repo will ship test vectors that both sides must pass.

## 🛡 Security notes

- **Never accept the root from the caller.** A caller who picks the root can always supply a matching proof. Store it in the contract and let only an admin set it.
- **Prevent double claims.** A valid proof stays valid forever. Record each used entry in contract storage.
- **Bind the claimant.** Include the address (and amount, if relevant) in the entry data, so one user can't reuse another user's proof.
- **Domain separation matters.** Because the verifier hashes the leaf with its own prefix, a 32-byte internal node can't be submitted as a leaf with a shorter proof. Pre-hashing leaves off-chain is *not* a substitute for this when the verifier accepts the hash directly.
- **Sorted pairs only remove direction flags.** They don't stop duplicate or repeated entries. If your data may contain duplicates, deduplicate before building the tree.
- **Not audited.** Review the code and the test vectors before using it with real funds.

## 🗂 Project layout *(planned)*

```
soroban-merkle-proof/
├── Cargo.toml
├── src/
│   └── lib.rs             verify() and hashing rules (no_std)
├── tests/                 valid, tampered and edge-case proofs
├── testdata/              shared vectors for the contract and the builder
├── builder/               off-chain tree and proof generator
└── examples/
    └── claim-contract/    the claim contract above, with tests
```

## 🛣 Status

| Area | State |
| :-- | :-: |
| Crate scaffold (`Cargo.toml`, `no_std`) | ⬜ planned |
| `verify` with domain-separated, sorted-pair hashing | ⬜ planned |
| Tests: valid proof, tampered proof, wrong root, single leaf, empty proof, odd leaf counts, max depth | ⬜ planned |
| Shared test vectors | ⬜ planned |
| Off-chain builder that passes the same vectors | ⬜ planned |
| Example claim contract with tests | ⬜ planned |
| CI: `cargo test`, `clippy`, wasm build | ⬜ planned |
| Testnet deployment of the example | ⬜ planned |
| crates.io release | ⬜ planned |
| Gas and resource measurements | ⬜ planned |

**v0.1 target:** `verify` implemented and tested against shared vectors, with the example contract on testnet.

## ⚠️ Limitations

- Proves membership only. It says nothing about what an entry is worth or who should be allowed to claim it.
- Proof size grows with the tree depth, so very large sets mean longer proofs.
- Roots from other Merkle libraries will not verify unless they use the same hashing rules.
- Not audited.

## 🤝 Contributing

Issues and pull requests are welcome. Please add tests for any change to the hashing rules, and update the shared vectors when a rule changes.

## 📄 License

[MIT](LICENSE)
