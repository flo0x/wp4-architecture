# Mutual identification for European Business Wallet presentation requests

**Authors/Contributors:**

- Florin Coptil, Bosch, Germany
- Werner Folkendt, Bosch, Germany
- Lal Chandran, iGrant.io, Sweden
- George J Padayatti, iGrant.io, Sweden
- Eelco Klaver, Credenco, The Netherlands
- Leif Johansson, SIROS Foundation, Sweden
- <Please add more .. >

**Obsoletes:** N/A

## Context

An EBW in the Holder role stores attestations that its owner treats as confidential, such as ultimate beneficial ownership and control structure. In the BU use cases, requests arrive backend to backend with no person present. The Holder EBW must decide alone, so it needs three answers a machine can check:

1. Which legal entity is asking?
2. Is the requesting software a real wallet unit, and is it still valid?
3. Does the owner's policy allow this entity to receive this attestation?

The proposed Regulation on the establishment of European Business Wallets, COM(2025) 838 final of 19 November 2025, addresses this in its Annex. Point 14(2)(b) states that "where Business Wallet owners use their Business Wallets unit to interact with competent national authorities and providers of electronic attestations of attributes, Wallet units shall enable authentication and validation of the Wallet unit components by presenting the Wallet unit attestations to those competent national authorities and providers upon their request".

Point 14 covers the issuance direction, and CS-01 already implements it. The proposal places no equivalent obligation on a party that requests attestations from a Wallet unit. That gap is what this decision fills.

**In presentation, the verification rule applies in one direction only.** CS-02 section 7.2, item 6 requires the Verifier to validate the Holder's Wallet Unit Attestation. Nothing requires the Verifier to identify the legal entity behind it. Today it is identified only as the party that signed the request. The request carries no EBWOID.

### The normative stack has already decided where this material travels

The question of how a Relying Party identifies itself to a Wallet in a Presentation Request is no longer open. Commission Implementing Regulation (EU) 2026/1731 of 15 July 2026 amends Implementing Regulations (EU) 2024/2977, 2024/2979, 2024/2980 and 2024/2982 as regards applicable standards, and its new Annex II applies ETSI TS 119 472-2 V1.2.1 (2026-03), clauses 4.1, 4.2, 5 and 6.

Clause 6 of that specification settles two points that bear directly on this decision.

**The Client Identifier Prefix is fixed.**

> OIDFVP-HAIP-COMMON-REQ-01: The Authorization Request shall use the Client Identifier Prefix `x509_hash`.

The terms `verifier_attestation`, `x509_san_dns` and `openid_federation` do not appear anywhere in ETSI TS 119 472-2 V1.2.1. The specification also incorporates OpenID4VC-HAIP 1.0 by reference (OIDFVP-HAIP-GEN-01: "All the mandatory requirements defined in clauses 5, 5.3, 7 and 8 of HAIP shall apply"), and binds both roles to clause 6 (OIDFVP-HAIP-SUPPORT-01 for the Wallet, OIDFVP-HAIP-SUPPORT-04 for the Relying Party). An earlier draft of this ADR proposed the `verifier_attestation` prefix; that option is closed.

**Relying Party identity travels in `verifier_info`, and it is mandatory.**

> OIDFVP-HAIP-COMMON-REQ-RO-01: The RO JWT body shall contain the `verifier_info` parameter.
> OIDFVP-HAIP-COMMON-REQ-RO-02: The `verifier_info` parameter shall contain RP Registrar-provided data.
> OIDFVP-HAIP-COMMON-REQ-RO-04: The value of the `format` member ... shall be: `"registrar_dataset"`.
> OIDFVP-HAIP-COMMON-REQ-RO-13: If the RP has a registration certificate, one of the elements of the `verifier_info` parameter shall include it.
> OIDFVP-HAIP-COMMON-REQ-RO-15: The value of the `format` member ... shall be: `"registration_cert"`.

The concern that `verifier_info` is optional in OpenID4VP and may be ignored by Wallets, which shaped earlier drafts of this ADR, is resolved: in the profile that the implementing act applies, it is a `shall` in the Request Object body, binding on both the Wallet and the Relying Party. The registrar dataset (OIDFVP-HAIP-COMMON-REQ-RO-06 to RO-12, per ETSI TS 119 475 Annex B) already carries the RP's identifier, service description, registry URI, intended-use identifier, purposes, privacy policy URI, and optionally the registered attestations and attributes for that intended use.

`verifier_info` is therefore not an extension point this decision has to argue for. It is the mandatory channel, it already carries two named `format` values, and it is where a third belongs.

## Decision

This decision introduces no new Client Identifier Prefix, no new attestation type, no new trust infrastructure and no new protocol message. It makes no change to rb-ebwoid, CS-04 or CS-05. It registers one `verifier_info` `format` value and states how it is bound and validated.

**1. WE BUILD registers the `ebwoid` Verifier Info format.** An EBW acting as Verifier identifies itself by including an additional element in the `verifier_info` array of the Request Object, alongside the elements ETSI TS 119 472-2 already requires.

- The Authorization Request uses the `x509_hash` Client Identifier Prefix, per OIDFVP-HAIP-COMMON-REQ-01. Nothing about the Client Identifier layer is profiled here.
- The element is a JSON Object which shall not contain the `credential_ids` member, following the pattern of OIDFVP-HAIP-COMMON-REQ-RO-03 and RO-14.
- The value of its `format` member shall be `"ebwoid"`.
- The value of its `data` member shall be the base64url encoding of an EBWOID presentation as defined in rb-ebwoid v1.0.0, including its Key Binding JWT. This follows the encoding pattern of OIDFVP-HAIP-COMMON-REQ-RO-16 for the registration certificate.

**Binding.** The `aud` claim of the Key Binding JWT shall be the `client_id` of the Presentation Request. Because `client_id` under the `x509_hash` prefix is the hash of the certificate that signs the Request Object, this binds the EBWOID presentation to the requesting party's certificate: the element cannot be copied into another party's request, because that party's `client_id` differs and it cannot produce a Key Binding JWT without the key in the EBWOID's `cnf` claim, which rb-ebwoid v1.0.0 section 3.2 makes mandatory.

> NOTE: Where a Wallet Provider issues the EBWOID bound to the same key as the certificate that signs the Request Object, the Request Object signature is itself the proof of possession and the Key Binding JWT adds nothing. This profile does not require that arrangement, so that implementers are free to separate credential keys from protocol keys.

**Validation.** The Holder EBW shall validate the presented EBWOID under rb-ebwoid v1.0.0 section 5: signature, issuer certificate chain to the QTSP trust anchor in the eIDAS Trusted List located via the `trust_anchor` metadata claim, `exp` and `iat` freshness, and revocation status per rb-ebwoid section 6. It shall verify the Key Binding JWT under the `cnf` key and check that its `aud` equals the request's `client_id`.

**2. This obligation applies to EBW-to-EBW traffic, and to requests for attestation types governed by an EBW rulebook.** An EBW always holds an EBWOID, so it can always comply. An attestation rulebook MAY declare that a given attestation type MUST NOT be released unless the request carries a valid `ebwoid` element, and an owner MAY apply stricter rules for its own wallet. A valid EBWOID does not by itself give a right to a response.

**This obligation does not apply to interactions with EUDI Wallets.** An EBW can also act as Verifier towards an EUDI Wallet, for example to request a PID or an attestation from a natural person. In that direction it sends the `verifier_info` elements ETSI TS 119 472-2 requires and nothing further. No EUDI Wallet is required to recognise the `ebwoid` format; a Wallet that does not recognise it ignores that element, as OpenID4VP already provides for unsupported Verifier Info types, and the request proceeds on the registrar dataset alone.

Verifiers that are not EBWs are not excluded in the other direction either. Their requests carry no `ebwoid` element, and the Holder decides what to release under Decision 3.

**3. One place where policy is decided.** The validated EBWOID is an input to the automatic approval list from [EBW EAA exchange automation](EBW-EAA-exchange-automation.md), not a second gate in front of it. The approval list is keyed on the EBWOID `id`, which is the EUID or an equivalent cross-border unique identifier, and on the attestation type. Where the owner approved a requester and attestation combination in advance, that approval is the Holder's consent for CS-02 section 7.1, and the wallet unit MUST record the release and show it to the owner. Otherwise it MUST ask the owner or reject.

Where an owner's policy requires more than legal identity, the registrar dataset that ETSI TS 119 472-2 already mandates carries the requester's registered intended use, purposes, and the attestations and attributes registered for it (OIDFVP-HAIP-COMMON-REQ-RO-09 to RO-12). Policy expressed against those members, rather than against a list of company names, is attribute-based and needs nothing further from this decision.

### What this decision does not change

| Area | Already decided in |
| --- | --- |
| Client Identifier Prefix, Request Object structure, `verifier_info` content | ETSI TS 119 472-2 V1.2.1 clause 6, applied by CIR (EU) 2026/1731 Annex II |
| Protocols | [Baseline protocols](base-protocols.md) |
| EBWOID claims, encoding, trust model, revocation | rb-ebwoid v1.0.0, authoritative and unamended |
| Attestation structure, validity, revocation, binding | CS-04 for the WUA, CS-05 for the BWUA, both authoritative |
| Trust lists | [Trusted lists](trusted-lists.md) |
| What a Verifier checks in a response | CS-02 section 7.2, item 6 |

## Consequences

### What becomes easier?

A Holder EBW can identify the requesting legal entity from a credential the requester already holds, anchored in a member-state business register and validated against the eIDAS Trusted List, delivered in the parameter the profile already requires it to send.

There is one identity artefact and one trust path. Nothing new is issued, and nothing has to be kept in step with anything else. An earlier draft of this ADR created a verifier-side attestation that had to be revoked in lockstep with the BWUA; that burden does not arise here, because no such attestation exists.

The decision sits inside the profile rather than beside it. An implementation that already meets ETSI TS 119 472-2 adds one array element on the sending side and one validation routine on the receiving side.

Requests can be answered without a person present, which is a precondition for using the EBW inside internal systems. Consent is given once, by the owner, in a list the owner controls.

### What becomes more difficult?

**Question 2 is answered differently than this ADR first assumed.** Under ETSI TS 119 472-2 a Relying Party authenticates with an access certificate that chains to a Trusted List; the profile requires no Relying Party to prove that it is a wallet unit, and no EUDI Relying Party does so. This decision identifies the legal entity and its registered intended use; it does not attest the requesting software. A WE BUILD `ebw_wallet_unit` format carrying a BWUA could be registered later on the same pattern if the group decides the certificate and Trusted List are insufficient, but it would be an addition on top of the profile rather than part of it, and it is not proposed here.

Owners must decide which attestations are confidential. Classification will vary until common practice develops.

Requesters without an EBWOID will not receive confidential attestations. For KYC and PA3 this must be explained before participants design their integration.

### Open items this decision depends on

These are not introduced by this decision, but it cannot be implemented while they stand.

1. **EBWOID revocation is unspecified.** rb-ebwoid v1.0.0 section 6 records "TODO: WE BUILD WP4", with an interim of `exp` plus an OAuth status list and revocation required for validity beyond 24 hours. WP4 should close this, since Decision 1 depends on a revocation check.
2. **Registrar data in EBW-to-EBW traffic.** OIDFVP-HAIP-COMMON-REQ-RO-02 requires `verifier_info` to carry RP Registrar-provided data. Whether an EBW acting as Verifier towards another EBW registers with an RP Registrar, or whether WE BUILD states that the `ebwoid` element stands in its place for that traffic, is unresolved and should be decided explicitly.
3. **There is no backend-to-backend transport for a Presentation Request.** CS-02's only invocation interface is `openid4vp://?request_uri=<URL>`, with the Verifier redirecting a user-agent, and both defined flows assume a person. [The credential offer endpoint registry](ebw-endpoint-lookup-service.md) is issuer-initiated and issuance-only. This blocks the BU use cases independently of this decision and should be resolved in its own.
4. **CS-02 section 7.1 forbids auto-consent**, while [EBW EAA exchange automation](EBW-EAA-exchange-automation.md) requires sharing without human approval for M2M scenarios. Decision 3 treats a prior owner approval as consent; the conflict between those two documents predates this decision and should be resolved explicitly.

### How do we address the risks introduced by this change?

The pre-flight specification should publish a default classification for the BU1 attestation types, so owners start from a common baseline.

The `ebwoid` format value should be registered wherever WE BUILD records Verifier Info format identifiers, so that it does not collide with a future ETSI or OpenID Foundation registration.

## Advice

Once merged, this is our consortium's decision. This does not mean all participants agree it is the best possible decision. In the decision-making process, we have heard the following advice.

- All authors / Contributors
- yyyy-mm-dd, Name, Affiliation, Country: OK or summary of advice
