# ADR: Point-in-Time Validity for Chained Attestations

**Authors:**

- Werner Folkendt, Robert Bosch GmbH, Germany

**Related ADRs:** ADR-168 (EAA Provider Identity Verification via Attestation Chaining)

## 1. Context and Problem Statement

ADR-168 establishes a "Chaining Model" where an EAA's header contains the issuer's full credential (e.g., the EBWOID), creating a self-contained proof of identity. This model is essential for supporting offline and archival verification.

However, a critical challenge arises from the lifecycle of credentials. An issuer's EBWOID may be short-lived (e.g., valid for one year due to key rotation policies) while the EAA it was used to sign has a much longer validity (e.g., a multi-year supplier contract or a non-expiring diploma).

**The Core Problem:** How does a verifier trust an EAA when the issuer's EBWOID, contained in the EAA's header, is now expired? For example:
*   A company issues an IBAN attestation in January, using an EBWOID that is valid until March.
*   In April, a relying party receives the IBAN attestation. The verifier checks the chained EBWOID and finds that it is expired, potentially leading to an incorrect rejection of the valid attestation.

The verifier must not check if the EBWOID is valid *today*. It must have a trusted way to verify that the EBWOID was valid and not revoked **at the moment the EAA was issued**. This is the principle of **Point-in-Time Validity**. This ADR defines the mechanisms to achieve this.

## 2. The Architectural Decision: LTV-Enabled Signatures

To achieve Point-in-Time Validity, the issuance process for chained EAAs **MUST** create a Long-Term Validation (LTV) enabled signature. This is accomplished by embedding two key pieces of information into the signature block at the time of issuance:

1.  **A Timestamp:** A cryptographic proof that the signature was created before a specific date and time.
2.  **A Revocation Proof:** A cryptographic proof of the EBWOID's revocation status at the time of signing.

By packaging these proofs together, the EAA becomes a tamper-proof, self-contained artifact that can be validated against its issuance time, regardless of how much time has passed.

## 3. Implementation Options for Timestamps

The level of trust in the timestamp determines the legal and business value of the long-term proof. The architecture allows for flexibility, enabling issuers to choose a method appropriate for their use case. The following options, from highest to lowest trust, can be supported:

**a) Qualified Timestamping (Highest Trust)**
*   **Mechanism:** For each EAA, the issuer's wallet makes a live request to an external, qualified Time-Stamping Authority (TSA). The TSA returns a legally-binding, signed timestamp token that is embedded with the signature.
*   **Use Case:** Legally binding documents, high-value credentials, government-issued attestations.
*   **Trade-off:** Highest trust and legal value, but incurs a per-transaction cost and a dependency on the TSA service during issuance.

**b) Batch Timestamping (Balanced Trust and Cost)**
*   **Mechanism:** The issuer signs a large number of EAAs locally throughout a set period (e.g., an hour or a day). The system then creates a single hash representing the entire batch of signatures (e.g., using a Merkle Tree) and obtains a single qualified timestamp for that batch hash. When an EAA is presented, it is accompanied by the batch timestamp and a proof of its inclusion in the batch.
*   **Use Case:** High-volume issuance where qualified trust is required but per-transaction costs are prohibitive (e.g., product passports, supplier certifications).
*   **Trade-off:** Balances high trust with very low cost, but adds complexity to the issuance and verification logic.

**c) Wallet-Internal Timestamping (Lowest Trust)**
*   **Mechanism:** The EBW itself, as a certified and integrity-protected device, generates and signs a timestamp using its own internal clock. This timestamp is included with the EAA signature.
*   **Use Case:** Low-risk, short-lived, or informational attestations where the risk of the issuer tampering with their own clock is considered negligible. Not suitable for legally binding archival.
*   **Trade-off:** No external cost or dependency, but offers the lowest level of trust as it is not provided by a neutral third party.

## 4. Decision

The architecture will recommend and support all three timestamping mechanisms. Rulebooks for specific attestations MUST define the minimum required level of timestamp trust. For any attestation intended to have long-term legal value or support compliance use cases (e.g., diplomas, product passports), a qualified timestamp (either per-signature or per-batch) is mandatory.

## 5. Consequences of the Decision

### 5.1 What becomes easier?

*   **True Archival Verification:** This decision makes the promise of ADR-168 a practical reality. It provides a standardized, interoperable mechanism to verify the authenticity of credentials over their entire lifecycle, even decades after issuance.
*   **Clarity for Implementers:** Wallet providers and verifiers have a clear, standards-based path (PAdES/CAdES/XAdES) for implementing validation logic, rather than depending on a hypothetical future API.
*   **Flexible Trust Models:** The architecture does not impose a one-size-fits-all solution. EAA issuers can choose the most appropriate and cost-effective timestamping method based on the specific legal and business requirements of their attestation.
*   **Risk Mitigation:** By embedding the proof of validity at issuance, the risk of future verifier error (e.g., incorrectly rejecting an EAA due to an expired EBWOID) is significantly reduced.

### 5.2 What becomes more difficult?

*   **Issuance Complexity:** The issuance process is no longer a simple "sign and send." It now involves additional steps: fetching revocation information and obtaining a timestamp. This increases the implementation workload for the issuance-side wallet software.
*   **Dependency on Revocation and Timestamp Services:** During issuance, the EAA Provider's system has a live dependency on the QTSP's OCSP service and potentially an external TSA. An outage in these services could temporarily halt the issuance of LTV-enabled EAAs.
*   **Increased Payload Size:** The embedded revocation proofs and timestamp tokens will further increase the overall size of the EAA signature block, adding to the payload that must be transmitted and stored.
*   **Verification Logic:** While based on standards, the LTV verification logic is inherently more complex than a simple signature check, requiring verifier wallets to correctly parse and validate the entire package.
