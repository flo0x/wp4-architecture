# MVP Onboarding Tracker: PA3_Scenario1_Bosch_Company(Holder)

| Parameter | Value                            |
|:---|:---------------------------------|
| **Use Case ID** | PA3_Scenario 1,2,3               |
| **Partner Name** | Robert Bosch GmbH                |
| **Business Role** | Company                          |
| **Technical Role** | Holder / EAA Issuer (self)       |
| **Onboarding Contact** | Florin Coptil (cof8fe@bosch.com) |

---

## 🏁 Phase 1: Wallet Instance Deployment

| ID | Requirement | Status | Evidence / Notes                          |
|:---|:---|:---|:------------------------------------------|
| 1  | **EBW** Installed & Reachable | 🟢 Complete | Procivis EBW (✅ 2026-05-04)               |
| 2  | **EUDI Wallet** Installed | 🟢 Complete | Procivis One mobile Wallet (✅ 2026-05-04) |
| 3  | **EUDI Wallet** Installed | 🟢 Complete | Credenco mobile Wallet (✅ 2026-08-04)     |
| 4  | **EUDI Wallet** Installed | 🟢 Complete | Digidentity mobile Wallet (✅ 2026-10-04)  |
....
---

## 🏁 Phase 2: Interoperability Test Bed (ITB) & CTS Testing

| Check | Requirement                            | Status | Evidence / Notes |
|:---|:---------------------------------------|:---|:---|
| 🟢 | **ITB Portal Account** Created         | 🟢 Complete | Verified and active on ITB portal |
| 🟢 | **CS-Basic (VCI-006)** Passed (EBW)    | 🟢 Complete | ✅ 2026-05-04 |
| 🟢 | **CS-Basic (VP-001)** Passed (EBW)     | 🟢 Complete | ✅ 2026-05-04 |
| 🟢 | **CS-Basic (VCI-006)** Passed (EUDI.1) | 🟢 Complete | ✅ 2026-06-25 |
| 🟢 | **CS-Basic (VP-001)** Passed (EUDI.1)  | 🟢 Complete | ✅ 2026-06-25 |
| ❌ | **CS-Trust** Passed (EBW)              | ❌ Deferred / N/A | De-prioritized / CS-05 not ready |
| ❌ | **CS-Trust** Passed (EUDI)             | ❌ Deferred / N/A | De-prioritized / CS-05 not ready |
| ❌ | **CS-Signature** Passed (EUDI.3)       | ❌ Deferred / N/A | Not implemented yet |

---

## 🏁 Phase 3: Cryptographic Trust List Registration

* **Target Trust Register:** None (No public trust registry required for pure Holder role. Trust is established by presenting credentials that trace back cryptographically to the registered Issuers).
* **IDunion Trust Console Account:** ❌ N/A for pure Holder role

---

## 🏁 Phase 4: Credential Provisioning Tracker (Scenario 1)

| Check | Credential Type | Status | Issuance Date | Issuance Portal / Source Used                     |
|:---|:---|:---|:---|:--------------------------------------------------|
| 🟢 | **PID** (EUDI Wallet) | 🟢 Complete | 2026-06-23 | Procivis PID Issuer (`https://issuer.eudiw.dev/`) |
| 🟢 | **EBWOID** (EBW) | 🟢 Complete | 2026-06-23 | Spherity (`https://eida-issuer.spherity.dev/`)    |
| 🟢 | **EUCC** (EBW) | 🟢 Complete | 2026-06-10 | Spherity (`https://eida-issuer.spherity.dev/`)    |
| 🟢 | **CompanyInfo** (EBW) | 🟢 Complete | 2026-07-14 | EBW Wallet                                        |
| 🟢 | **ContactPerson** (EBW) | 🟢 Complete | 2026-07-14 | EBW Wallet                                           |
| 🟢 | **VAT** (EBW) | 🟢 Complete | 2026-07-14 | EBW Wallet                                           |
| 🟢 | **Owner** (EBW) | 🟢 Complete | 2026-07-14 | EBW Wallet                                           |
| 🟢 | **Control** (EBW) | 🟢 Complete | 2026-07-30 | EBW Wallet                                           |
| 🟢 | **UBO** (EBW) | 🟢 Complete | 2026-07-30 | EBW Wallet                                           |
| 🟢 | **AuthorisedSignatories** (EBW) | 🟢 Complete | 2026-07-30 | EBW Wallet                                           |

---

## 🕒 Execution & Test Run History

| Date | Scenario | Result | Tester | Comments |
|:---|:---|:---|:---|:---|
| 20.05.2026 | Wallet installation | Partial | Florin | Initial wallet deployment |
| 10.06.2026 | EBWOID issuance (EAA) | **PASS** | Florin / Stephan | EBW active |
| 10.06.2026 | EBWOID presentation | **PASS** | Florin / Stephan | Active verification |
| 10.06.2026 | EUCC issuance (EAA) | **PASS** | Florin / Stephan | Verified EU company credential |
| 10.06.2026 | EUCC presentation | **PASS** | Florin / Stephan | Complete presentation |
| 10.06.2026 | EUCC presentation (SD) | **FAIL** | Florin / Stephan | SD-JWT signature verification issues |
| 23.06.2026 | PID issuance (EAA-procivis) | **PASS** | Florin | Procivis-One PID acquired |
| 10.06.2026 | PID issuance (EAA-eudi issuer) | **PASS** | Florin | Fallback issuer verification |
| 10.06.2026 | PID issuance (EAA-other issuer) | **PASS** | Florin | |
| 23.06.2026 | PID presentation | **PASS** | Florin / Stephan | Successfully verified with Bank |
| 23.06.2026 | PID presentation (SD) | **FAIL** | Florin | Key Attestation validation issue |
| 25.06.2026 | Mutual authentication | **FAIL** | Florin / Stephan | Manual handshake validation failed |
| 25.06.2026 | IBAN Schema creation | **PASS** | Florin / Stephan / Ricky | Bank schema initialized |
| 25.06.2026 | IBAN Schema issuance (DB) | **PASS** | Florin / Stephan / Ricky | Deutsche Bank issued IBAN |
| 25.06.2026 | IBAN Attestation acceptance (BOSCH) | **PASS** | Florin / Stephan / Ricky | Stored securely in Bosch EBW |
| 25.06.2026 | Onboarding using wallet with Business Data (EBWOID, EUCC) | **PASS** | Florin / Stephan | Complete business onboarding flow |
