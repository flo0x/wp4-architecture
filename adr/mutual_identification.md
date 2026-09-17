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

The proposed Regulation on the establishment of European Business Wallets, COM(2025) 838 final of 19 November 2025, addresses this in its Annex. Point 14(2)(b) states that "where Business Wallet owners use their Business Wallets unit to interact with competent national authorities and providers of electronic attestations of attributes, Wallet units shall enable authentication and validation of the Wallet unit components **by presenting the Wallet unit attestations** to those competent national authorities and providers **upon their request**".

Point 14 covers the issuance direction, and CS-01 already implements it. The proposal places no equivalent obligation on a party that requests attestations from a Wallet unit. That gap is what this decision fills. Note the mechanism the Annex describes: a Wallet unit *presents* its attestations *upon request*. That is a request-and-present exchange, not an attestation carried inside a request.

**In presentation, the verification rule applies in one direction only.** CS-02 section 7.2, item 6 requires the Verifier to validate the Holder's Wallet Unit Attestation, along with credential authenticity, holder binding, and nonce and audience binding. Nothing requires the Verifier to prove the same about itself. Today it is identified only as the party that signed the request (CS-02 sections 5 and 8.2). The request carries no EBWOID.

The requester already holds an EBWOID. An EBW is a single wallet unit that plays the Holder, Issuer, and Verifier roles ([BWUA based on TS3](bwua-ts3-attestation.md), CS-02 chapter 4). Nothing new has to be issued to it. What is missing is an agreed way to present what it already holds.

### Why not carry identity inside the request

Two mechanisms can carry identity material inside a Presentation Request, and the consortium should be aware that both have become harder since this ADR was first drafted.

| | `verifier_attestation` Prefix, OpenID4VP 5.9.3 and 12 | `verifier_info`, OpenID4VP 5.11 |
| --- | --- | --- |
| Status in HAIP | **No longer permitted.** HAIP 1.0 became a Final Specification on 24 December 2025. Section 5 now requires the `x509_hash` Client Identifier Prefix for signed requests; `verifier_attestation` and `x509_san_dns` are no longer listed | Not mentioned by HAIP 1.0 Final |
| Status in CS-02 | CS-02 section 5 allows it, but CS-02 normatively cites HAIP Implementer's Draft 1 (reference [2], accessed 24 November 2025), one month before HAIP Final | Not used today |
| Identity carriers per request | One JWT in the `jwt` JOSE header. Sector authorisations such as a banking licence have nowhere to go | Two, checked alongside `client_id` |
| Wallet obligation | Wallet must trust the attestation issuer | OpenID4VP 5.11: "It is at the discretion of the Wallet whether it uses the information from `verifier_info`". A profile cannot raise this for all Wallets |

Building on `verifier_attestation` would mean profiling a Client Identifier Prefix that the profile CS-02 cites has since removed, and that every EBW would have to implement *in addition to* the `x509_hash` path it needs anyway for the EUDI direction (see Decision 2). Every participant builds an X.509 verifier stack regardless; no participant has another reason to build a `verifier_attestation` stack.

**A third option avoids the Client Identifier layer entirely.** Because an EBW is both Holder and Verifier, the requester can identify itself the way every other party in this ecosystem identifies itself: by presenting a credential. The EBWOID is already a key-bound SD-JWT VC ([rb-ebwoid](https://github.com/webuild-consortium/webuild-attestation-rulebooks-catalog/blob/main/rulebooks/rb-ebwoid/README.md) v1.0.0 section 3.2 makes `cnf` mandatory and forbids selective disclosure), issued by a member-state business register or QTSP and validated against the eIDAS Trusted List (section 5). Presented by its holder to a Relying Party, it is used exactly as rb-ebwoid section 4.1 already defines. No new attestation type is required, and nothing in rb-ebwoid changes.

Legal entities that run an EUDI Relying Party component hold no EBWOID. Making identification mandatory for all requests would exclude them from all traffic, including data that the KYC and PA3 use cases depend on and that is not confidential. The reverse also holds: an EBW may itself request attestations from an EUDI Wallet, and in that direction the EUDI ecosystem's own relying party rules apply.

## Decision

This decision changes nothing in OpenID4VP, introduces no new attestation type, no new Client Identifier Prefix, and no new trust infrastructure. It makes no change to rb-ebwoid, CS-04 or CS-05. It profiles, for EBW-to-EBW traffic, an exchange built entirely from mechanisms OpenID4VP 1.0 already defines.

**1. Mutual identification is reciprocal presentation.** Where an EBW acting as Verifier requests an attestation type that the Holder EBW's rulebook or owner policy classifies as confidential, the requesting EBW MUST present its own EBWOID as part of establishing the request, and the Holder EBW MUST validate it before releasing anything.

The exchange uses the OpenID4VP `request_uri_method=post` flow:

1. The requesting EBW sends the invitation, carrying `client_id`, `request_uri` and `request_uri_method=post`. It contains no EBW-specific material.
2. The Holder EBW POSTs to the `request_uri`, supplying a fresh `wallet_nonce`.
3. The requesting EBW returns the signed Request Object. OpenID4VP requires it to echo `wallet_nonce`; this profile additionally requires it to carry the requester's EBWOID presentation in `verifier_info`, under a Verifier Info type registered by WE BUILD for this purpose, with its Key Binding JWT over that `wallet_nonce`.
4. The Holder EBW validates and then answers or refuses.

The Holder EBW MUST validate the presented EBWOID:

- under **CS-02 section 7.2, item 6**, unchanged. Because the requester is presenting a credential, the Holder EBW is acting as Verifier for that presentation, and item 6 already obliges it to validate the presenter's **Wallet Unit Attestation**, credential authenticity, holder binding, and nonce and audience binding. Question 2 above is answered by existing normative text, with no new artefact and no restatement of BWUA claims.
- under **rb-ebwoid v1.0.0 section 5**: signature, issuer certificate chain to the QTSP trust anchor in the eIDAS Trusted List located via the `trust_anchor` metadata claim, and `exp` and `iat` freshness.

**Binding.** The `cnf` key of the presented EBWOID MUST be the key that signed the Presentation Request Object. The Holder EBW MUST compare the JWK Thumbprint of the EBWOID `cnf` against the thumbprint of the key that verified the Request Object signature, and MUST abort on mismatch. This is the binding CS-01 section 7.4 already applies in the issuance direction, where the WIA `cnf` key is the DPoP key and `cnf.jkt` is matched on receipt. Without it the presentation is replayable into another party's request.

**Loop prevention.** The EBWOID attestation type MUST NOT be classified as confidential, and an EBWOID presented for mutual identification MUST NOT itself trigger a mutual identification exchange. Two EBWs would otherwise challenge each other indefinitely.

**Caching.** A Holder EBW SHOULD cache the validated EBWOID identifier together with the thumbprint of the bound key for a period bounded by its own policy and by the EBWOID's `exp`, so that the exchange runs once per relationship rather than once per request.

**Where the obligation falls.** The MUST in this decision falls on the requesting EBW, which must include the material, and on the Holder EBW's own policy, which refuses to answer without it. No Wallet is required to process `verifier_info` that it does not wish to process. A Wallet that ignores it forgoes the protection and releases nothing it would not otherwise release. This profile does not raise OpenID4VP section 5.11 from a discretionary mechanism to a mandatory one for any Wallet.

**2. This obligation applies to EBW-to-EBW traffic, and to requests for attestation types governed by an EBW rulebook.** An EBW always holds an EBWOID, so it can always comply. An attestation rulebook MAY declare that a given attestation type MUST NOT be released unless the requester has identified itself under Decision 1, and an owner MAY apply stricter rules for its own wallet. A valid EBWOID does not by itself give a right to a response.

**This obligation does not apply to interactions with EUDI Wallets.** An EBW can also act as Verifier towards an EUDI Wallet, for example to request a PID or an attestation from a natural person. In that direction the EBW follows the rules of the EUDI ecosystem, using the Client Identifier Prefix it mandates, which under HAIP 1.0 Final is `x509_hash` with a Relying Party access certificate. Nothing in this decision requires an EUDI Wallet to support anything.

Verifiers that are not EBWs are not excluded in the other direction either. Their requests carry no EBWOID, and the Holder decides what to release under Decision 3, using the Client Identifier Schemes CS-02 section 5 already allows.

**3. One place where policy is decided.** The validated EBWOID is an input to the automatic approval list from [EBW EAA exchange automation](EBW-EAA-exchange-automation.md), not a second gate in front of it. The approval list is keyed on the EBWOID `id`, which is the EUID or an equivalent cross-border unique identifier, and on the attestation type. Where the owner approved a requester and attestation combination in advance, that approval is the Holder's consent for CS-02 section 7.1, and the wallet unit MUST record the release and show it to the owner. Otherwise it MUST ask the owner or reject.

### What this decision does not change

| Area | Already decided in |
| --- | --- |
| Protocols | [Baseline protocols](base-protocols.md) |
| OpenID4VP request and response processing | OpenID4VP 1.0, used as published. No new parameter, prefix or response type is defined |
| Signed requests, `client_id`, allowed schemes, nonce, audience, expiry | CS-02 sections 5, 6.1.1, 6.1.3, 8.2 |
| What a Verifier checks in a response | CS-02 section 7.2, item 6, applied unchanged in both directions |
| EBWOID claims, encoding, trust model, revocation | rb-ebwoid v1.0.0, authoritative and unamended |
| Attestation structure, validity, revocation, binding | CS-04 for the WUA, CS-05 for the BWUA, both authoritative |
| Trust lists | [Trusted lists](trusted-lists.md), applied in CS-01 section 7.4 |

## Consequences

### What becomes easier?

A Holder EBW can identify the requesting entity and check that its wallet unit is sound and not revoked, using a credential the requester already holds, a trust path anchored in a member-state business register, and the response-validation code path its implementation already runs for every presentation it receives.

There is one identity artefact, one binding rule, one trust path and one revocation path. Nothing is issued, and nothing has to be kept in step with anything else. The earlier concern that a new verifier-side attestation would have to be revoked in lockstep with the BWUA does not arise, because no such attestation exists.

The design does not depend on the Client Identifier layer, so it is unaffected by the removal of `verifier_attestation` from HAIP, by the move to `x509_hash`, and by any future change to the set of permitted prefixes.

Requests can be answered without a person present, which is a precondition for using the EBW inside internal systems. Consent is given once, by the owner, in a list the owner controls.

Sector authorisations extend the same mechanism without new machinery. Where an owner's policy requires more than identity, for example a banking licence issued by a supervisory authority, the Holder EBW's requirement is expressed as an ordinary DCQL query for those credentials alongside the EBWOID. This addresses the shift from identity-based to attribute-based access control raised in review, without per-company approval lists and without any new attestation format.

### What becomes more difficult?

The exchange adds a round trip on a cold relationship. Caching reduces this to once per requester and policy period, but implementers should not assume a single request and response.

`request_uri_method=post` is defined by OpenID4VP 1.0 but is the less-travelled path compared with a plain `GET` on `request_uri`. Implementations will need to verify coverage in the stacks they use. CS-02 section 8.1 shows only the `GET` form and would need to permit and profile the `POST` form for EBW-to-EBW.

Owners must decide which attestations are confidential. Classification will vary until common practice develops.

Requesters without an EBWOID will not receive confidential attestations. For KYC and PA3 this must be explained before participants design their integration.

### Open items this decision depends on

These are not introduced by this decision, but it cannot be implemented while they stand.

1. **EBWOID revocation is unspecified.** rb-ebwoid v1.0.0 section 6 records "TODO: WE BUILD WP4", with an interim of `exp` plus an OAuth status list and revocation required for validity beyond 24 hours. WP4 should close this. The exposure is bounded: wallet unit revocation is still covered, because CS-02 section 7.2 item 6 checks the requester's Wallet Unit Attestation, whose status mechanism CS-05 section 7.2 defines. Only revocation of the legal-entity identity depends on the open item.
2. **There is no backend-to-backend transport for a Presentation Request.** CS-02's only invocation interface is `openid4vp://?request_uri=<URL>`, with the Verifier redirecting a user-agent (sections 6.1.2, 7.1.3, 8.1), and both defined flows assume a person. [The credential offer endpoint registry](ebw-endpoint-lookup-service.md) is issuer-initiated and issuance-only. Step 1 of Decision 1 therefore has no defined delivery mechanism. Steps 2 to 4 need no additional addressing, because they ride the exchange step 1 opens. This should be resolved in its own decision.
3. **CS-02 section 7.1 forbids auto-consent**, while [EBW EAA exchange automation](EBW-EAA-exchange-automation.md) requires sharing without human approval for M2M scenarios. Decision 3 treats a prior owner approval as consent, but the conflict between those two documents predates this decision and should be resolved explicitly rather than by assertion.

### How do we address the risks introduced by this change?

The pre-flight specification should publish a default classification for the BU1 attestation types, so owners start from a common baseline, and should state that the EBWOID type is never confidential.

Where a customer runs only a Relying Party component and no Holder component, its provider must decide whether to supply a Holder component so that the customer can present an EBWOID. This is a commercial decision rather than a protocol one.

If the Architecture Group prefers that identity material travel in the Client Identifier layer after all, Decision 1 changes and Decisions 2 and 3 stand. The group should then choose between `x509_hash` with an ETSI TS 119 475 Registration Certificate, which is the mechanism the EUDI ecosystem mandates and which every EBW implements anyway, and the `openid_federation` prefix, which is a live OpenID4VP prefix but requires trust infrastructure the consortium has not established. It should not choose `verifier_attestation`, which HAIP 1.0 Final no longer permits.

## Advice

Once merged, this is our consortium's decision. This does not mean all participants agree it is the best possible decision. In the decision-making process, we have heard the following advice.

- All authors / Contributors
- yyyy-mm-dd, Name, Affiliation, Country: OK or summary of advice
