# BringID: Technical Paper

**BringID** is a privacy-preserving protocol that enables users to prove humanness using verified Internet credentials — without ever linking accounts or revealing identity.

This repo contains the **technical paper** describing the BringID protocol design, trust assumptions, and security considerations.

> 📄 [Read the paper → `whitepaper.md`](./whitepaper.md)

---

## What is BringID?

BringID allows users to privately verify ownership of Web accounts (e.g., Uber, GitHub, Airbnb) using [MPC-TLS](https://github.com/tlsnotary/tlsn), and then generate unlinkable identity proofs using [Semaphore](https://semaphore.pse.dev).

It offers a practical and modular approach to fighting Sybil attacks in crypto by increasing the cost of forgery while preserving privacy.

---

## Key Concepts

- **MPC-TLS Verifications**  
  Web credentials are verified via [TLS Notary](https://github.com/tlsnotary/tlsn) without exposing plaintext data.

- **Privacy via Zero-Knowledge**  
  Credentials are converted into unlinkable commitments and grouped by type in separate Semaphore groups.

- **Composable Proofs**  
  Users generate zero-knowledge inclusion proofs from multiple credential groups to prove humanness.

- **Custom Trust Scoring**  
  Verifying parties define their own trust rules, combining group proofs with credential age, weights, and thresholds.

---

## Design Priorities

- **Privacy-first**: No linking between Web accounts or crypto addresses  
- **Progressive decentralization**: Launch with minimal trust, decentralize over time  
- **Sybil deterrence**: Cost of forgery exceeds reward when verifying parties use good scoring policies

---

## Trust Assumptions Over Time

| Phase             | Who is Trusted                     | Privacy Guarantee         |
|------------------|------------------------------------|---------------------------|
| Initial Setup     | BringID team                       | MPC-TLS + Semaphore      |
| TEE Phase         | Hardware vendor (e.g., Intel SGX)  | MPC-TLS + Semaphore      |
| Decentralized     | Independent notary network         | MPC-TLS + Semaphore      |

---

## Get Involved

We're looking for feedback from identity researchers, privacy builders, and crypto devs.

- [Telegram Group](https://t.me/bringid) 
- [Open a GitHub issue](https://github.com/bringid/whitepaper/issues) to discuss or suggest improvements

---

## 📄 License

This project is open-sourced under the [MIT License](./LICENSE.md).
