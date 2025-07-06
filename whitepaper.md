# BringID: Private Identity Based on Verifiable Internet Credentials (Draft)
## Abstract
BringID is a privacy-preserving protocol for verifying humanness using existing Internet accounts. By aggregating verifiable credentials from services like Uber, GitHub, or Airbnb — without ever revealing user identities — BringID enables crypto applications to assess trust and reputation without compromising privacy. Leveraging MPC-TLS and zero-knowledge group proofs, the system produces unlinkable attestations and customizable trust scores. This approach addresses the growing need for Sybil resistance, privacy-preserving airdrops, and composable identity in an environment increasingly flooded with bots and synthetic activity.

## Motivation

Crypto ecosystems face an escalating challenge: distinguishing real users from bots and Sybil identities. Governance votes, airdrop allocations, and protocol incentives are vulnerable to exploitation by actors who spin up thousands of wallets to game systems designed for individuals.

This challenge is amplified by advances in AI, which make it easier than ever to automate identity creation and behavior simulation at scale.

Web accounts (such as rideshare profiles, developer contributions, or accommodation history) carry implicit reputation that is difficult to fake. However, existing approaches to using these accounts for verification often sacrifice user privacy.

BringID introduces a new model: users prove control over these accounts through MPC-TLS and convert them into zero-knowledge credentials. Each credential is unlinkable, reusable, and composable — enabling applications to validate specific traits or thresholds without ever accessing the underlying accounts.

## Design Principles

- **Privacy-focused**: Can't compromise on privacy.
- **Focused on crypto power users**: Starting with desktop browser extension, which is widespread among crypto users.
- **Progressive decentralization**: Starting with trusted setup, migrating to TEE in the short term, and planning for complete decentralization in the longer term.

## System Components

- **BringID extension**: Browser extension based on the [TLSN extension][1]. Stores secret Master Key required for Credentials, establishes MPC-TLS sessions with target servers, provides proofs of verifications for verifying parties.
- **Credential Registry**: a smart contract storing Credentials for completed verifications. Verifications from the same Credential Group are mapped to a single [Semaphore][2] group; each Credential Group is mapped to a separate Semaphore group. Verifying parties use Credential Registry to validate Semaphore group inclusion proofs.
- **Relayer**: Submits transactions for adding credentials to Credential Registry. For enhanced privacy, transactions are batched together and submitted periodically (typically once per day).
- **Indexer**: Helps BringID extension fetch onchain data such as Merkle trees and completed verifications.
- **TLS Notary**: The open-source [TLSN Notary][3]. It participates in MPC-TLS sessions alongside the BringID Extension, jointly establishing a TLS connection with the target Web service. The Notary evaluates a garbled circuit to validate ciphertext integrity and produces a signed attestation over committed fields. It operates without ever accessing plaintext user data.
- **BringID Attestation Service**: Verifies BringID TLS Notary attestation and generates an attestation — a signed object that certifies the verified credential and can be submitted onchain.

## How It Works

### 1. Verifying Internet credentials over MPC-TLS

To validate a Web account, the BringID Extension and the TLS Notary cooperatively execute a joint MPC‑TLS session with the target Web service. The interaction is shown in Figure 1.

<img width="1248" alt="BringID_account_verification" src="/diagrams/BringID_account_verification.png" />

- **Joint MPC‑TLS handshake**  
  The Extension and Notary behave as a single TLS client. Together they negotiate a session with the Web Service (e.g., Uber) and request only the data needed for the selected credential type.
  - **1 a – Transcript → Extension**. The full, still‑encrypted TLS transcript is delivered to the Extension.
  - **1 b – Attestation → Extension**. Inside a garbled‑circuit MPC execution, the Notary validates the ciphertext, derives commitments to the requested plaintext fields, and signs an attestation. The Notary sees only ciphertext and commitments — never the plaintext data itself.

- **Submit presentation**  
  The Extension packages the attestation, plus the minimally required plaintext fields, into a verifiable presentation and submits it to the BringID Attestation Service.

- **Receive attestation**  
  The BringID Attestation Service checks the Notary attestation's signature, recomputes the commitments, maps the credential to its Semaphore group ID, and returns a signed attestation. This attestation certifies that the user controls the verified account and can be fed directly to the Credential Registry smart contract.

Further details on the MPC‑TLS construction and its security guarantees are available in the [TLSN documentation](https://docs.tlsnotary.org).

### 2. Adding Credentials to Registry

The process is shown in Figure 2, where multiple user-submitted attestations are collected by the Relayer and submitted as a single batch to the Credential Registry smart contract.

<img width="1408" alt="submitting_vcs" src="/diagrams/submitting_vcs.png" />

1. BringID Extension generates a Semaphore identity commitment using the private, locally stored Master Key and prepares a task for the Relayer, which includes the Credential and the generated commitment.
2. The Relayer batches tasks from multiple users and submits them to the Credential Registry periodically (typically once per day).
3. The Credential Registry verifies each attestation, ensures the credential is not duplicated, and publishes the identity commitment to the appropriate Semaphore group. Each Credential Group maps to a distinct Semaphore group, enabling granular proof targeting and application-specific trust logic.

The identity commitment is generated as follows:  
`commitment = Hash(master_key || credential_group_id || account_id_hash)`

To learn more about Semaphore and its group inclusion proofs, visit the [Semaphore documentation](https://docs.semaphore.pse.dev).

### 3. Generating Proofs for Verifying Parties

The proof delivery process is shown in Figure 3. A verifying party requests inclusion proofs from the user's BringID Extension, which are then submitted onchain for validation.

<img width="1568" alt="providing_proofs_to_verifier" src="/diagrams/providing_proofs_to_verifier.png" />

1. **Request proofs**  
   The verifying party (a Web App) requests specific credential proofs from the user's BringID Extension.

2. **Generate and provide proofs**  
   The Extension generates Semaphore inclusion proofs and sends them back to the verifier.

3. **Submit proofs**  
   The verifier submits the received proofs to a custom verifier smart contract.

4. **Validate proofs**  
   The verifier smart contract calls the Credential Registry smart contract to validate inclusion proofs using current Merkle roots and group mappings. After successful validation, the custom verifier contract may execute application-specific logic, such as airdropping tokens to the verified recipient or granting access to gated resources.

- Each Credential Group corresponds to a separate Semaphore group, enabling flexible proof targeting.
- This structure ensures that verification logic and trust policies can remain fully application-specific.
- Privacy is preserved through Semaphore's use of zero-knowledge proofs, which allows users to prove group membership without revealing their specific identity or any linkable information across credentials.

## Trust Score Calculation

- Verifying parties can either use BringID Trust Score or use a custom formula with custom weights for each Credential Group.
- Verifying parties can target specific Credential Group (e.g., eligible users should complete at least one Uber account and stay at least once at an Airbnb).

Below is a simplified example of a scoring function implemented in pseudocode, where each Credential Group has a fixed score weight and an associated maximum allowed age. Only credentials verified within the allowed timeframe contribute to the score:

```javascript
const weights = {
    github: { score: 20, maxAgeDays: 90 },
    uber: { score: 30, maxAgeDays: 60 },
    airbnb: { score: 30, maxAgeDays: 60 }
};

function computeScore(proofs) {
    let score = 0;
    const now = Date.now();
    
    for (const proof of proofs) {
        const { credentialGroup, timestamp } = proof;
        const params = weights[credentialGroup];
        
        if (!params) continue;
        
        const ageInDays = (now - timestamp) / (1000 * 60 * 60 * 24);
        if (ageInDays <= params.maxAgeDays) {
            score += params.score;
        }
    }
    
    return score;
}
```

- This function gives more weight to certain Credential Groups.
- It ensures only recently verified credentials (within their allowed age window) are included in the score, ignoring outdated or stale credentials.

## Privacy guarantees

BringID enforces strong privacy by design through unlinkable commitments and zero-knowledge group proofs.

Each web account is turned into a unique, unlinkable identity commitment, derived using a secret Master Key that never leaves the user’s device. These commitments are grouped by credential type (e.g., GitHub, Airbnb) and published onchain into separate Semaphore groups.

When a user later interacts with a third-party application, they generate zero-knowledge membership proofs for the relevant Credential Groups. These proofs confirm group membership without revealing which specific account was used or allowing any linkage across credentials.

Figure 4 below illustrates how the user’s accounts are committed to Credential Groups, and how independent group proofs are combined and sent to a Verifying Party:

<img width="1632" alt="privacy_2" src="/diagrams/privacy_2.png" />

This architecture ensures Verifying Parties can compute Trust Scores or apply eligibility filters—without ever learning user identities, linking accounts, or tracking behavior across apps.

### Privacy properties:
- Cannot link accounts together
- Cannot identify a specific user
- Cannot track across Verifying Parties
- Can only verify group membership
- Each proof is application-specific and unlinkable

## Progressive Decentralization

1. **Initial phase**: Trusted setup with BringID team controlling key infrastructure such as TLS Notary and BringID Attestation Service. Even in this phase, no plaintext data is ever visible to operators.
2. **Short term**: Transition to Trusted Execution Environment (TEE)-backed infrastructure for key components, shifting trust towards hardware manufacturers.
3. **Longer term**: Full decentralization with multiple independent nodes, removing reliance on any single entity. Currently in the research phase.

## Trust Assumptions

BringID is designed to preserve user privacy at every stage while progressively reducing the need to trust centralized actors. The system uses MPC-TLS to ensure that sensitive account data remains encrypted and never exposed—even during verification. What changes over time is *who* users must trust for the correctness of attestations.

The table below outlines the evolution of trust across BringID's rollout phases:

| Phase              | Infrastructure Operator | Trust Requirement                               | Privacy Guarantee          | Key Risks                        |
|--------------------|-------------------------|--------------------------------------------------|-----------------------------|----------------------------------|
| **Trusted Setup**  | BringID team            | Users trust BringID to run open-source code and not forge attestations | Enforced via MPC-TLS       | Operator could forge identities |
| **TEE Phase**      | BringID (via TEE)       | Users trust hardware vendor (e.g., Intel SGX) to enforce code integrity | Enforced via MPC-TLS | Hardware vulnerabilities         |
| **Decentralized**  | Independent operators   | Trust minimized through open participation and auditing | Enforced via MPC-TLS       | Requires honest-majority model  |

At all stages, plaintext user data remains private and inaccessible to any party, including BringID itself. The transition path — from trusted setup to decentralization — is intended to remove the need for users to place blind trust in any single entity.

## Master Key Derivation and Recovery
The Master Key could be derived deterministically from the user's crypto wallet signature, ensuring seamless setup without new secrets. At this stage, there is no Master Key recovery mechanism. If the user loses access, they may generate a new Master Key and repeat verifications. In such cases, previously issued commitments become inactive, and verifying applications should apply time-bounded logic to re-verified credentials to mitigate duplication or abuse.

## Security Considerations

BringID is designed with a strong emphasis on privacy, but also makes deliberate choices to reduce the attack surface and ensure the integrity of identity verifications. Below are the core security properties, assumptions, and mitigations.

---

### 1. Attestation Integrity

- **Threat**: A malicious actor forges a credential without actually owning the corresponding Web account.  
- **Mitigation**: BringID uses [MPC-TLS](https://github.com/tlsnotary/tlsn) to jointly establish a TLS session between the user's BringID Extension and a TLS Notary. The Notary validates ciphertext integrity using a garbled circuit and only signs an attestation when the credential policy is met.  
- **Assumption**: The TLS Notary is honest and runs open-source, verifiable code.

---

### 2. Credential Forgery via Collusion

- **Threat**: The Notary and Attestation Service collude to produce fraudulent credentials.  
- **Mitigation**: This is a risk in the **initial trusted setup phase**. Mitigations include:  
  - MPC ensures the Notary sees no plaintext user data.  
  - All attestations are verifiable on-chain.  
  - The protocol roadmap includes a migration to **TEE-backed infrastructure**, and eventually to a **decentralized network of notaries**.

---

### 3. Replay or Duplication of Credentials

- **Threat**: An attacker reuses a credential proof across applications or users.  
- **Mitigation**: Credentials are committed as one-time identity commitments derived from a private Master Key. Unlinkability is preserved across apps, and reuse is prevented via:  
  - Semaphore’s nullifier logic  
  - Optional time bounds and freshness checks

---

### 4. Master Key Safety

- **Threat**: Loss or theft of the Master Key enables unauthorized re-verification or Sybil abuse.  
- **Mitigation**: The Master Key is stored locally and never leaves the device. If lost, a new key must be generated and all credentials must be re-verified.  
- **Design note**: Re-verifications should be time-bound by the verifying party to avoid abuse of duplicate credentials (e.g., “no more than one new Uber credential per 30 days”).

---

### 5. Web Service Interference

- **Threat**: Target websites detect or block BringID verification sessions.  
- **Mitigation**: BringID uses MPC-TLS, which emulates standard TLS client behavior. Only minimal, read-only data is requested (e.g., number of trips or reviews), and no modifications are required. The verification is indistinguishable at the TLS layer.

---

### 6. Identity Farming

- **Threat**: Sophisticated attackers pre-farm Web accounts (e.g., Uber or Airbnb) to create synthetic but realistic-looking identities at scale.  
- **Mitigation**:  
  - Web accounts are harder to fake than wallet addresses. Creating aged, active, and reputable accounts (with rides, commits, reviews, etc.) carries non-trivial effort and often real-world cost.  
  - Credential policies can require specific thresholds (e.g., minimum number of rides, recent activity).  
  - Because each credential group is independently verified, attackers must farm multiple high-quality accounts to appear legitimate across diverse domains.

---

### 7. Economic Cost Design for Verifying Parties

Verifying parties (e.g., airdrop contracts, governance systems) must **treat identity verification as an economic deterrent**, not a guarantee of uniqueness. The key security assumption is:

> **The value of a reward per verified account must be lower than the cost of producing that Sybil identity.**

Therefore:

- Rewards should be tied to multiple credential groups (e.g., Uber + GitHub), increasing the cost of forgery.  
- Time freshness and credential decay mechanisms should be applied.  
- Eligibility requirements should be designed with real-world friction in mind (e.g., travel, developer contributions).  
- Optional use of scoring thresholds or percentile filters (e.g., “top 20% by trust score”) can reduce exposure.

BringID does not prevent Sybils in a strict cryptographic sense, but **increases the cost of forgery to an economically infeasible level** when used correctly.


## Summary

BringID introduces a practical, privacy-preserving protocol for proving humanness in a decentralized ecosystem. By leveraging  web credentials verified over MPC-TLS, zero-knowledge group proofs, and a modular trust scoring model, BringID makes it possible to differentiate real users from Sybils without compromising user privacy.

[1]: https://github.com/tlsnotary/tlsn-extension "TLSN Browser Extension Source"
[2]: https://semaphore.pse.dev "Semaphore: Privacy-Preserving Zero-Knowledge Protocol"
[3]: https://github.com/tlsnotary/tlsn "TLSN Notary Reference Implementation"

## Appendix: Glossary

- **Attestation**: A cryptographically signed statement about a verified fact
- **Credential**: A privacy-preserving proof of account ownership
- **Credential Group**: A category of verified accounts (e.g., Uber account with at least one completed trip, Airbnb account with at least one stay, etc) that share the same verification requirements and scoring weight
- **Credential Registry**: Onchain storage for zero-knowledge commitments
- **Trust Score**: Numerical measure of likely human authenticity
- **Master Key**: User-controlled cryptographic key that binds all credentials
- **MPC-TLS**: Multi-Party Computation TLS protocol
- **Verifying Party**: Application or service consuming humanity proofs
- **Semaphore**: Zero-knowledge group membership protocol
- **Verifiable Credential**: Cryptographic proof of account verification
