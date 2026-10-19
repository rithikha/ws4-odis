# The ODIS Passport

*Status: draft proposal for discussion. This document is not part of the ODIS specification. It revises the architecture proposed in PR #39, drawing on issues #15, #16, #20, #27, and #33, and the follow-up discussion in the ws4-odis Slack channel. The WS4 Agent-Manifest-Agent Credentials-ODIS overlap call discussion has not yet been fully integrated into the RFC yet. In this document, § refers to sections of ODIS.md and "Section" to sections of this document.*

## Summary

ODIS divides agent access across three layers. The [Passport](../ODIS.md#31-layer-1--the-passport-identity--attestation) describes the agent's software and running instance; the [Bridge](../ODIS.md#32-layer-2--the-bridge-delegation--access) records who the agent acts for and what authority it holds; the [Router](../ODIS.md#33-layer-3--the-router-discovery--governance) governs calls to tools and services.

This document proposes that the Passport produce evidence and platform-level results, and never consult the Agent Registration Record. Every check that needs the record (which logical agent this is, whether its software is approved, whether it is active) moves to a new Verifier in the Bridge, the only layer ODIS already connects to the record. That Verifier issues the Agent Runtime Credential. The Bridge keeps its name and existing role; the Verifier joins its Delegation Service, Provider Adapter, and Cache and Revocation Manager as a fourth subcomponent. In terms of the RATS architecture ([RFC 9334](https://www.rfc-editor.org/rfc/rfc9334)), the Passport holds the Attesters and any platform verifiers, the Bridge holds the Verifier that appraises the agent, and the Router is a Relying Party.

## Scope

This document specifies what the Passport produces, what the Bridge's Verifier does with it, and the Agent Runtime Credential the Verifier issues. It builds on PR #38, which folds Agent Manifest content into the Agent Registration Record as optional fields; where those fields are present, the Verifier appraises against them (Section 3.4). Section 4 lists the changes this requires in ODIS.md and in related proposals.

Out of scope:

1. Delegation and call governance, which ODIS [§3.2](../ODIS.md#32-layer-2--the-bridge-delegation--access) and the Router proposal (PR #37) define. This document specifies only the Verifier's place in the Bridge and what it hands to the delegation service.
2. Wire formats. The Agent Runtime Evidence and the Agent Runtime Credential are abstract data models. A claims profile (e.g., TRACE) defines the claims a Verifier emits, and a format profile (e.g., a WIT-SVID or a Verifiable Credential) defines how they are signed and bound to a key. ODIS defines neither; Section 3.5 states what any profile must carry and lists the formats recommended today.
3. Deployment patterns, including managed endpoints that run agent harnesses the deployment cannot modify (#13). Any placement must meet the isolation rule in PSP-02.

## 1. Architecture

### 1.1 Components and flow

The Passport's evidence producers and platform verifiers feed the Bridge's Verifier, which issues the credential the Delegation Service relies on:

1. **Produce evidence:** components outside the agent, including any platform verifiers, describe the instance, the software it runs, and its holder key ([2.1](#21-passport)).
2. **Resolve the registration:** the Verifier maps the evidence to one active Agent Registration Record ([BRG-02](#22-bridge-verifier)).
3. **Appraise:** the Verifier checks the evidence against the record's approved issuers, approved software, and reference values.
4. **Issue:** the Verifier signs a short-lived Agent Runtime Credential bound to the holder key, carrying results and references to the evidence.
5. **Delegate:** the delegation service accepts the credential and proceeds under ODIS §3.2.

### 1.2 Role mapping

| RATS role | ODIS component |
|---|---|
| Attester | Passport evidence producers: software signer, workload attestor, hardware root |
| Endorser | Hardware manufacturer or supply-chain signer, trusted through the Agent Registration Record or Verifier configuration |
| Reference Value Provider | The Agent Registration Record: `approved_software_refs`, and `reference_values` if PR #38 lands |
| Verifier | The Bridge's Verifier, for every claim that depends on the Agent Registration Record. Platform verifiers in the Passport, such as a SPIRE server or a TEE remote attestation service, are Verifiers for platform claims only |
| Attestation Result | Platform results, such as an SVID or TEE attestation token, which are inputs to the Bridge's Verifier; and the Agent Runtime Credential, whose claims follow a claims profile and whose packaging follows a format profile (Section 3.5) |
| Relying Party | The delegation service, the Router, and native-mode targets |

The flow matches the RATS passport model: the presenter carries the Verifier's results to each relying party, and no relying party contacts the Verifier per call.

### 1.3 Where the line falls

As written, ODIS splits verification across two layers. ODIS-L1-02 requires the Passport to match software evidence against `approved_software_refs`, while ODIS-L2-14 has the Bridge check the same record, the §3 diagram connects the record only to the Bridge, and §1.3 allows the issuer to be separate from the registration authority (#33). Both layers would need the record. A rule that "the Passport does no verification" does not resolve this, because SVID issuers and TEE remote attestation services are verifiers in their own right and naturally sit in the Passport.

This document therefore draws the line at the registration record, not at verification. The Passport may verify platform facts against policy it owns, such as which workload is running or whether a TEE is genuine, and pass the results on. Any check that needs the record happens once, in the Bridge's Verifier. This would settle four open points:

- **#33:** one architecture. The component that issues the credential reads the registration record, which keeps the issuance-time enforcement the #33 thread preferred, and the Passport never reads it.
- **#27:** later layers receive results, not evidence. The credential references evidence instead of embedding it.
- **#28 and PR #36:** `agent_id` and `registration_record_ref` come from the record the Verifier resolved, so they can remain MUST fields.
- **Existing platforms:** SPIRE and TEE attestation services keep their role. Their SVIDs and attestation tokens become inputs the Verifier accepts from issuers the record approves.

### 1.4 Alternative placement

WS4 has also discussed splitting the Passport in two: an evidence part, and a verification part that issues SVIDs and TEE attestation results. The Bridge would then focus on whose authority the agent acts under, rather than on the agent's identity. Platform verifiers alone do not need the registration record, so the choice turns on where the record-dependent checks go. If they stay in the Bridge, the model is the one in Section 1.1. If the Passport's verification part also performs them, both layers read the record: the Passport for agent identity and software approval, the Bridge for delegation modes and entitlements (L2-14). That keeps the Bridge principal-focused, at the cost of two readers that must agree on the current record version (BRG-12).

The BRG requirements bind the Verifier wherever it sits. Choosing the Passport placement would move them and change Section 4, not their content. Either way, the specification must name the one component that performs the registration check.

### 1.5 Trust inputs

| Fact | Produced by | What makes it trustworthy | Appraised by |
|---|---|---|---|
| Running instance and its holder key | Workload identity system, such as a SPIRE server acting as a platform verifier | Producer outside the agent, authenticated by the Verifier (PSP-02, BRG-03) | Verifier |
| Platform state and key protection | TEE or TPM, often through a remote attestation service | The service's own policy, trusted through Verifier configuration | Verifier, which accepts or rejects the service's result |
| Software the instance executes | Workload attestor, container runtime, or hardware | As above; hardware removes trust in the host | Verifier |
| Software origin | Signing or provenance service | Verified signature or provenance statement, or a governed approval entry | Verifier |
| Software approval, logical agent, sponsor, lifecycle state | Agent Registration Record | Authoritative `record_issuer` and the §6.1 resolution rule | Verifier |
| Current risk or compromise | Security monitors | ODIS-L1-12 signal checks and declared freshness | Verifier at issuance and renewal |
| Requested identity or software | The agent itself | Not trusted; never sufficient for any check | Not applicable |

### 1.6 Threats addressed

- A rogue agent claiming a registered identity.
- An approved artifact registered, but a different one running.
- An agent asserting its own `agent_id` or software digest.
- An agent reading its holder key and using it outside the authorized path.
- Evidence from different machines or workloads combined into one credential.
- Replay of another instance's evidence, which is not secret.
- A credential in a widely accepted format, such as an SVID, used directly against services without delegated authority.
- Evidence checked against keys the agent chose.
- A stale registration version keeping authority after its approvals changed.
- Later layers trusting evidence formats they cannot appraise.

## 2. Requirements

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals.

- **PSP** requirements bind the Passport. **BRG** requirements bind the Verifier, a new Bridge subcomponent; if WS4 placed the Verifier in the Passport instead (Section 1.4), they would move with it unchanged. These are working IDs, kept separate so they are not confused with existing ODIS IDs during review; Section 4.1 gives the ODIS IDs they would take on merge.
- **Profile** is a provisional tag naming the lowest ODIS conformance profile ([ODIS §8.1](../ODIS.md#81-profile-definitions)) at which the requirement applies.
- **Ref** lists related ODIS requirements without the `ODIS-` prefix, ODIS sections (§), and sections of the Router proposal (PR #37), cited as "Router: section name". Where a requirement changes ODIS.md, Ref ends with its status in italics: *Agreed #n*, settled in the thread on issue #n; *Proposed #n*, proposed here and not yet settled; or *New*. A requirement with no status restates ODIS.md.

### 2.1 Passport

| ID | Title | Requirement | Profile | Ref |
|---|---|---|---|---|
| PSP-01 | Evidence role | The Passport MUST produce Agent Runtime Evidence (Section 3.1). It MUST NOT be required to consult an Agent Registration Record or to issue the Agent Runtime Credential. The Passport MAY include platform verifiers, such as a SPIRE server or a TEE remote attestation service, that appraise platform evidence against policy they own and issue platform results, such as an SVID or attestation token. Those results are Agent Runtime Evidence; the Bridge's Verifier decides whether to accept them. | Core | §3.1, L1-11 · *Proposed #33* |
| PSP-02 | Isolation | Each evidence producer, platform verifier, and holder-key store MUST be outside the control of the model-visible process, for example a node agent, container runtime, orchestrator, or endpoint security agent. That process MUST NOT be able to read their keys, alter their code or configuration, or author evidence about itself. The conformance declaration MUST state the separating mechanism for each component. | Core | L1-01, L1-11, §9.2 · *Proposed #32, #14* |
| PSP-03 | No static secrets | The agent MUST authenticate to ODIS components with short-lived, automatically rotated, cryptographically verifiable credentials. Static API keys, long-lived service-account secrets, and credentials in agent-visible environment variables MUST NOT be used on that path. | Core | L1-01 |
| PSP-04 | Required evidence | For each runtime instance the Passport MUST produce (1) software provenance evidence: a digest, signature, or provenance statement for the artifact; and (2) runtime/workload evidence that identifies the instance and identifies, by digest, the artifact it executes. One evidence object MAY satisfy both, as when a workload attestor also checks the container image's signature. An identity credential that does not state the artifact digest, such as an SVID on its own, does not satisfy (2). An SBOM alone MUST NOT serve as software provenance evidence. | Core | L1-02, L1-03, L1-08 · *Agreed #15* |
| PSP-05 | Hardware evidence | Safety-critical and high-assurance agents SHOULD supply hardware evidence about the execution environment, platform measurements, or holder-key protection; the Safety profile requires it. Hardware evidence MUST supplement, and MUST NOT replace, the evidence in PSP-04, and MUST commit to the holder key or to the runtime/workload evidence it supports, for example through a quote's report data, so the Verifier can link the two (BRG-05). | Extended | L1-04, §8.1 · *Proposed #21* |
| PSP-06 | Holder key | The holder key MUST be generated and held outside the model-visible process, and runtime/workload evidence MUST commit to its public key so the Verifier can bind the credential to it. The model-visible process MAY ask an authorized presenter to construct a proof (L2-13); it MUST NOT read or export the key. | Core | L1-09, L2-12, L2-13 · *Proposed #19* |
| PSP-07 | Integrity and freshness | Each evidence object MUST be integrity-protected by its producer and MUST state its subject, `issued_at`, `expires_at` or `maximum_age`, and `verification_method`. Where the evidence type supports a Verifier-supplied nonce, the producer MUST include it. | Core | L1-11, §6.2 · *Proposed #5* |
| PSP-08 | No asserted agent identity | Evidence identifies a workload and an artifact, not a logical agent. A Verifier MUST NOT rely on an `agent_id` or `registration_record_ref` carried in evidence. | Core | §6.2, L2-14 · *Proposed #28* |
| PSP-09 | Platform credentials | A platform credential, such as a Kubernetes service-account token, cloud instance token, or SPIFFE SVID, MAY serve as runtime/workload evidence when a producer outside the model-visible process obtains it. On its own it MUST NOT authorize delegation. | Core | §3.1, L1-09 · *Proposed #14* |
| PSP-10 | Evidence retention | The deployment MUST retain each evidence object, or an authenticated reference to it, for its declared audit retention period so that `evidence_refs` in a credential resolve. | Core | CC-01 · *New* |

### 2.2 Bridge: Verifier

| ID | Title | Requirement | Profile | Ref |
|---|---|---|---|---|
| BRG-01 | Verifier role | The Bridge MUST include a Verifier subcomponent that appraises Agent Runtime Evidence and issues the Agent Runtime Credential before the delegation service issues or mediates authority. The Verifier and the delegation service MAY be one component; when separate, the delegation service MUST authenticate the Verifier. A platform MAY host the Passport and the Bridge together, provided PSP-02 holds and its conformance declaration names each role. | Core | §3.2, §3.5 · *Proposed #33* |
| BRG-02 | Registration resolution | The Verifier MUST resolve the evidence to exactly one active Agent Registration Record, using a workload identity listed in `workload_identities` (Section 3.4) and a producer listed in `approved_runtime_issuers`, and MUST apply the §6.1 resolution rule. No match, or more than one, MUST fail closed. `agent_id` MUST come only from the resolved record, and MUST name one logical agent within its trust domain; a shared execution role or service account MUST NOT serve as `agent_id`. Deployments MAY distribute, replicate, or cache records centrally or decentrally, provided resolution meets the §6.1 rule. | Core | L2-14, §1.3, §6.1, CC-02 · *Proposed #6, #17, #28* |
| BRG-03 | Evidence admission | The Verifier MUST authenticate each evidence producer using keys from an authoritative source, such as a configured trust bundle or OIDC discovery from the authenticated issuer, and never from a location the agent supplies. It MUST confirm the producer is authorized for the agent's `trust_domain` and MUST reject evidence outside its validity period. Evidence supplied only by the agent process MUST NOT satisfy any requirement. | Core | L1-11, §6.1 · *Proposed #5, #15* |
| BRG-04 | Software appraisal | The Verifier MUST match software provenance evidence against `approved_software_refs` in the resolved record and MUST confirm the artifact's origin. A digest match against an entry made through the governed registration workflow (CC-05) satisfies this in Core; signatures and provenance statements are stronger. Endorsements MUST come from endorsers the record or the Verifier's configuration trusts. | Core | L1-02, L1-08 · *Agreed #26; Proposed #18* |
| BRG-05 | Instance binding | Evidence from separate producers is useful only if it describes one instance. The Verifier MUST establish that (1) every runtime/workload and hardware evidence object describes the same instance, by each committing to the same holder key, by one committing to a digest of the other (`links`, Section 3.1), or by each carrying the same Verifier nonce; (2) the artifact digest in that evidence equals the approved artifact, recorded as `software_hash`, which is how software provenance evidence links to the instance; and (3) the requester holds the holder key, by signing a fresh Verifier challenge with it. Evidence that cannot be linked MUST NOT be combined. If the Verifier cannot establish (1), (2), and (3), it MUST NOT issue or accept the credential. | Core | L1-03, L1-09 · *Agreed #15; Proposed #21* |
| BRG-06 | Reference values | Where the record carries `reference_values` or `evidence_requirements` (PR #38), issuance and continued acceptance MUST satisfy them. An assurance level MUST NOT be claimed unless a published evidence profile defines what the evidence proves for that relationship. | Core | §6.1 · *Proposed #21* |
| BRG-07 | Fail closed | The Verifier MUST NOT issue, renew, or accept a credential if any required check cannot complete, including when evidence, verification keys, collateral, registration data, or a required risk-signal source is unavailable. | Core | L1-11, L2-14 |
| BRG-08 | Results, not evidence | The credential MUST carry appraisal results (Section 3.2) and authenticated references to the evidence, not the evidence itself. The delegation service, the Router, and policy engines MUST NOT be required to parse evidence. | Core | §6.2, §6.4 · *Proposed #27* |
| BRG-09 | Binding and lifetime | Each credential MUST be bound to the holder key in the evidence and to one runtime instance. Its lifetime MUST NOT exceed the declared `maximum_agent_runtime_credential_lifetime_seconds` or the earliest expiry of the evidence it rests on, and SHOULD be minutes, not hours. Credentials MUST rotate automatically before expiry. | Core | L1-05, L1-09 · *Agreed #16* |
| BRG-10 | Assurance results | The Verifier MUST state, for each relationship in Section 3.3, the level its appraisal reached, and MUST state `holder_key_protection`. Only the Verifier sets these values; the agent cannot raise them. A relying party MUST read a missing or unknown value as the lowest level. | Core | L1-03, L1-04, §8.2 · *Proposed #15, #21* |
| BRG-11 | Risk signals | The Verifier SHOULD consume current risk and compromise signals at issuance and renewal. A confirmed compromise signal MUST cause revocation and require fresh evidence before reissue. A declared signal source with no signal inside its `maximum_signal_age_seconds` is indeterminate, and the Verifier MUST NOT issue or renew while it is. | Core | L1-12 · *Proposed #30* |
| BRG-12 | Registration changes | The `agent_id` and its record persist across instances and credentials. A credential naming a superseded record version MAY be accepted until its `expires_at` only if the current version is active and the evidence the credential was issued on still satisfies it, including that `software_hash` is still approved. The resolver MUST resolve the current version, and the Bridge MUST apply L2-01 and L2-14 against it. Renewal MUST reference the current version. Otherwise the credential MUST be rejected; work already admitted MAY drain under L1-10 when the change is administrative. A de-provisioned `agent_id` remains identifiable for audit and MUST NOT authorize new credentials. Routine changes MUST take effect within one credential lifetime; urgent changes, such as suspension or compromise, MUST propagate as revocation events (L3-04). | Core | L1-06, L1-10, L2-14, L3-04, Router: Change timing · *Agreed #16; Proposed #33* |
| BRG-13 | Revocation | Revoking an Agent Runtime Credential MUST also revoke the Delegation Records and tokens derived from it. Revocation events MUST identify their subject by `credential_id`, `runtime_instance_id`, or `agent_id`, and MUST reach other trust domains that accepted the credential within their declared propagation bound. Holder keys are per instance and expire with the credential, so de-provisioning needs no separate key-registry revocation. | Core | L3-04, L3-05, Router: Credential sources · *Proposed #17* |
| BRG-14 | Bridge token | Where the Bridge issues its own token for the Router (Router: Credential sources), the token MUST carry the Verifier's results and MUST NOT outlive the credential. A deployment MAY issue one Bridge artifact that serves as both, provided it carries the fields in Section 3.2. | Core | Router: Credential sources, Check frequency · *New* |
| BRG-15 | Credential format | The Verifier MAY issue the credential in any format profile that carries the fields in Section 3.2 and binds the credential to the holder key, and SHOULD use a format recommended in Section 3.5. A bearer format MUST NOT serve as the credential. Where the format is one that services outside ODIS already accept, such as an SVID, the deployment MUST limit acceptance to ODIS components, through the audience of the credential or its presentation proof, or by distributing the Verifier's trust bundle only to them. A service that accepts the credential without a Delegation Record grants the agent its own authority, not delegated authority, and is not a native-mode target (L2-15). | Core | L1-09, L2-15, Router: Router isolation · *New* |

### 2.3 Optional

| ID | Title | Requirement | Profile | Ref |
|---|---|---|---|---|
| BRG-16 | Federated acceptance | A trust domain MAY accept credentials issued by another domain's Verifier. Acceptance MUST be an explicit, configured trust decision naming that Verifier, and the receiving Bridge issues its own token (Router: Check frequency). | Extended | L1-07 |
| BRG-17 | Independent re-appraisal | A relying party that appraises evidence itself MUST obtain it through `evidence_refs` with the collateral its evidence profile requires (`collateral`, Section 3.1), and MUST fail closed if a check cannot complete. Embedded certificate chains do not supply the relying party's trust anchors or replace its freshness, revocation, and appraisal-policy checks. Accepting another Verifier's results instead is the separate trust decision in BRG-16. | Extended | L1-07, L1-11 · *Proposed #20* |

## 3. Schema definitions

Field names are provisional. This section replaces ODIS §6.2 with two models: Agent Runtime Evidence (3.1), the Passport's output, and the Agent Runtime Credential (3.2), the Verifier's output.

### 3.1 Agent Runtime Evidence

One object per piece of evidence. The first seven fields are those ODIS §6.2 already requires inside `attestation_evidence`.

| Field | Required | Description |
|---|---|---|
| `type` | Yes | Kind of evidence, such as an image signature, provenance statement, workload attestation document, hardware quote, or platform result (e.g., an SVID or TEE attestation token) |
| `issuer` | Yes | Producer of the evidence |
| `subject` | Yes | What the evidence describes: a workload, an artifact, or a key |
| `issued_at` | Yes | Time the evidence was produced |
| `expires_at` or `maximum_age` | Yes | How long the evidence remains valid |
| `proof` | Yes | Embedded proof, or an authenticated reference to it |
| `integrity` | Yes | Integrity metadata for the object |
| `supports` | Yes | Relationships supported: `software_provenance`, `instance_to_software`, `instance_to_key`, `platform_state` (*Proposed #21*) |
| `profile` | Yes | URI of the evidence profile that defines what this evidence proves (*Proposed #21*) |
| `verification_method` | Yes | `public_jwks`, `configured_trust_bundle`, `mtls_client_certificate`, or `platform_attestor_api` (*Proposed #5*) |
| `workload_id` | Conditional | Workload identity, such as a SPIFFE ID, for runtime/workload evidence (*New*) |
| `artifact_digest` | Conditional | Algorithm-prefixed digest of the artifact the subject executes or describes (*Agreed #15*) |
| `holder_key` | Conditional | Public key or thumbprint the instance holds, for evidence supporting `instance_to_key` (*Proposed #19*) |
| `nonce` | Conditional | Verifier-supplied freshness value, where the evidence type supports one (*New*) |
| `collateral` | Recommended | `embedded` or `retrievable`: whether the evidence includes the verification material its profile requires (BRG-17) (*Proposed #20*) |
| `links` | Conditional | Digest of another evidence object or key this evidence commits to, such as a workload key in a TEE quote's report data (BRG-05) (*New*) |

### 3.2 Agent Runtime Credential

The credential carries the Verifier's results bound to the holder key. These are the fields every claims and format profile must carry (Section 3.5). Compared with ODIS §6.2, the issuer moves to the Bridge, `attestation_evidence` is replaced by `assurance` and `evidence_refs`, and the three fields PR #36 would relax stay required.

| Field | Required | Description |
|---|---|---|
| `credential_id` | Yes | Collision-resistant identifier for this credential |
| `format_version` | Yes | Version of the descriptor or carrier profile |
| `issuer`, `issuer_key_ref` | Yes | The Bridge's Verifier and the key that protects the credential (*Proposed #33*) |
| `holder_key_ref` | Yes | Confirmation data for the holder key the evidence committed to (PSP-06) |
| `agent_id` | Yes | Copied from the resolved record (BRG-02) (*Proposed #28; keeps the current MUST that PR #36 would relax*) |
| `registration_record_ref` | Yes | `record_issuer`, `record_id`, `record_version`, and `record_digest` of the record version current at issuance. `record_digest` names its algorithm and covers the record's payload as issued (*Proposed #22; keeps the current MUST that PR #36 would relax*) |
| `runtime_instance_id` | Yes | The one instance the credential is bound to; a binding profile MAY derive it, for example from the holder-key thumbprint (*Agreed #16*) |
| `software_hash` | Yes | Digest of the verified artifact as packaged. It covers prompts, skills, or model files bundled into the artifact, not components managed separately, such as a model selected at runtime (*Agreed #16*) |
| `assurance` | Yes | Results per relationship and for holder-key protection (Section 3.3) (*Proposed #15, #21*) |
| `evidence_refs` | Yes | Authenticated references to the evidence the results rest on (*Proposed #27*) |
| `trust_domain` | Yes | Organizational trust boundary |
| `issued_at`, `expires_at` | Yes | Issue and expiry times (BRG-09) |
| `audiences` | Conditional | Required where the carrier binds an audience; where the audience belongs to each presentation proof, as with DPoP, the verifier checks it there (*Proposed #27*) |
| `supply_chain_ref` | Recommended | Reference to the artifact's origin: signing authority, provenance statement, or governed approval entry (*Proposed #18*) |

### 3.3 Assurance results

`assurance` records what appraisal established, not what kind of evidence arrived. Levels run from lowest to highest: `host-platform` (reported by a component on the agent's host; the Core minimum), `independent-observer` (reported by a component the host does not control), and `hardware` (rooted in a TEE quote or TPM measurement under an evidence profile).

| Field | Description |
|---|---|
| `instance_to_software` | Level at which the instance was shown to run `software_hash` |
| `instance_to_key` | Level at which the instance was shown to hold the key in `holder_key_ref` |
| `holder_key_protection` | Where the holder key is kept, using the §8.2 `key_custody` values |

### 3.4 Registration record fields the Verifier reads

This document adds `workload_identities` to §6.1 (*Proposed #6, #33*): the workload identities, such as SPIFFE IDs, that resolve to the record. The Verifier also reads `lifecycle_state`, `approved_runtime_issuers`, `approved_software_refs`, `trust_domain`, `record_issuer`, and `record_version`.

PR #38 carries Agent Manifest content in the record as optional fields. Where present, the Verifier appraises typed entries in `approved_software_refs` (e.g., model, tool catalog, or prompt digests) under BRG-04, and `reference_values` and `evidence_requirements` under BRG-06. A record without these fields remains conformant, and the Verifier applies only the checks the record calls for. Switching between options the record already permits, such as two approved models, does not change the record; changing what it permits does, and BRG-12 applies. The Router governs the actions that result from behaviour within an approved composition (#16).

### 3.5 Credential formats

Relying parties differ in what they accept, so ODIS separates what a Verifier says from how it is packaged, and fixes neither. A claims profile (e.g., TRACE) defines the claims; a format profile signs them and binds them to a holder key. Any combination that carries the fields in Section 3.2 and binds the holder key is acceptable (BRG-15). ODIS recommends formats rather than mandating one, and the list can grow. Formats that bind a holder key are recommended today:

| Format | Holder binding | Where the results go |
|---|---|---|
| WIT-SVID (SPIFFE profile of the WIMSE Workload Identity Token) | `cnf`, proven by a Workload Proof Token or HTTP signature | Additional claims; `sub` is the agent's SPIFFE ID |
| X509-SVID | Certificate key, proven by mutual TLS | A profile-defined extension, or `evidence_refs` and a results lookup |
| Verifiable Credential with a DID subject | Key in the DID document | `credentialSubject` |
| EAT or EAR | `cnf` | Claims; EAR is designed for attestation results |

A JWT-SVID is a bearer token and MUST NOT serve as the credential (BRG-15). A workload SVID is already a credential for the workload identity part, issued by a platform verifier; the Agent Runtime Credential adds what only the registration record can establish. An SVID-form Agent Runtime Credential therefore names the logical agent, and its SPIFFE ID is distinct from the workload SVID the Passport supplied. This keeps the door open for agent credentials in organizations that standardize on SPIFFE or WIMSE, subject to the acceptance limit in BRG-15.

## 4. Changes required elsewhere

### 4.1 ODIS.md

| Section | Change |
|---|---|
| §1.3 | Agent Runtime Credential: "issued by the Bridge's Verifier after appraisal of Agent Runtime Evidence". Remove "or agent class" from `agent_id` (BRG-02). Replace the paragraph allowing the issuer to be separate from the registration authority: the Verifier reads the record; Passport producers need not. |
| §1.4 | Add Agent Runtime Evidence, platform verifier, Verifier, and Evidence profile. |
| §2.3 | Pillar 3 gap: the credential is "issued after appraisal of attestation evidence", not "issued after attestation". |
| §3 diagram | The Passport outputs Agent Runtime Evidence, not the Agent Runtime Credential. Add a Verifier to the Bridge that receives the registration record lookup and issues the credential. |
| §3.1 | Retitle "Layer 1 — The Passport (Attestation Evidence)". The Passport produces evidence and platform results for software provenance, runtime/workload state, and optionally hardware, and does not consult the registration record. Move the credential properties (lifetime, rotation, sponsor) to §3.2. Item (b) states that the attestation identifies the software artifact the instance is executing (PSP-04). The platform-credential paragraph becomes PSP-09. |
| §3.2 | Add the Verifier as a fourth subcomponent ahead of the Delegation Service. The Delegation Service "accepts the Agent Runtime Credential from the Verifier" instead of validating Passport output. |
| §3.5 | Split "Attestor / Identity Issuer" into "Evidence Producer and platform verifier (Passport)" and "Verifier (Bridge)". |
| §3.6 | Steps 2–3: producers supply evidence to the Verifier, which appraises it against the active record and issues the credential. Step 5 no longer repeats the registration resolution. |
| §5.1 | Replace with PSP-01 to PSP-10, renumbered ODIS-L1-01 to ODIS-L1-10, and dispose of each current ODIS-L1 requirement as in Section 4.2. |
| §5.2 | Add BRG-01 to BRG-17 as ODIS-L2-16 to ODIS-L2-32, or as a new §5.2.1 "Verification". L2-14 now names the Verifier. |
| §5.4 | Receive ODIS-L1-06 and ODIS-L1-10, which govern the registration record rather than the Passport. |
| §6.1 | Add `workload_identities` (Section 3.4). |
| §6.2 | Replace with Sections 3.1 and 3.2. |
| §6.4 | `agent_runtime` carries the credential's results (Section 3.2), never evidence. |
| §7.1 | Add credential format profiles (Section 3.5) beside the token binding profiles in §7.1.1. |
| §7.2 | Gateway diagram: "L1: evidence collection" and "L2: verification, token exchange". |
| §8.1, §8.2 | Core bullet "verifiable agent identity bound to attested software" names the Verifier. Add `minimum_assurance` (per relationship), `minimum_holder_key_protection`, `passport_isolation`, and `risk_signal_sources`; the first two replace `attestation_level` (#15). Safety requires `hardware`. |
| §9.1, §9.2 | Threat rows cite PSP and BRG IDs. Trust boundary 1 becomes "Evidence production"; boundary 2 "Verification and delegation", where appraisal is the gate. |

### 4.2 Disposition of ODIS-L1 (Passport) requirements

| ODIS ID | Disposition |
|---|---|
| L1-01 Secret-Zero Elimination | Stays in the Passport as PSP-03 |
| L1-02 Software Attestation | Split: evidence production PSP-04; matching against the record BRG-04 |
| L1-03 Runtime / Workload Attestation | Split: PSP-04; BRG-05 |
| L1-04 Hardware Attestation | Split: PSP-05; BRG-06 |
| L1-05 Credential Lifecycle | Moves to the Bridge as BRG-09 |
| L1-06 Identity Provisioning | Moves to §5.4; governs the record, not the Passport |
| L1-07 Federated Trust | Moves to the Bridge as BRG-16 |
| L1-08 Trusted Distribution | Split: PSP-04; BRG-04 |
| L1-09 Holder-of-Key Authentication | Split: key custody PSP-06; credential binding BRG-09 |
| L1-10 Accountable Sponsor | Moves to §5.4; drain rules referenced by BRG-12 |
| L1-11 Attestation-Bootstrapped Trust | Rewritten as PSP-02 and BRG-03: evidence must be independently verifiable by the Verifier |
| L1-12 Runtime Security State | Moves to the Bridge as BRG-11 |

### 4.3 Related proposals

| Proposal | Effect |
|---|---|
| PR #36 (§6.2 fields to SHOULD) | Addresses the same problem differently. Because the Verifier reads the record, `agent_id`, `registration_record_ref`, and `runtime_instance_id` can stay required, so the relaxation would no longer be needed. |
| PR #37 (Router) | "Trust sources" and "Layered trust": the Bridge carries the Verifier's results rather than the Passport's conclusions, and the Passport issues no credential. "Credential sources" and "Check frequency": the issuance-time checks are the Verifier's (BRG-02 to BRG-05). The open question on registration checks (#33) is answered by Section 1.3. The Summary and the glossary entry for Passport change to match. No Router rules change otherwise. |
| PR #38 (§6.1 manifest fields) | Builds on it. Agent Manifest content in the record (typed software references, `reference_values`, `evidence_requirements`) is optional and, where present, feeds BRG-04 and BRG-06 (Section 3.4). |
| Claims and format profiles (e.g., TRACE, CoSAI Agent Credentials) | Possible sources of the profiles in Section 3.5, referenced informatively. ODIS fixes only the fields in Section 3.2. |
| PR #39 (Passport) | Builds on it and proposes a different placement: the appraiser and issuer move to the Bridge, and its two registration-check routes (PSP-15, and the lookup case of PSP-16) become one. Its other requirements carry over in substance, as PSP requirements where they govern evidence and BRG requirements where they govern appraisal or the credential. |

## 5. Example

A finance team registers an invoice-reconciliation agent. Its Agent Registration Record, version 7, names `agents/invoice-reconciler`, lists the approved image digest, the SPIFFE ID its workloads receive, and the SPIRE server as an approved runtime issuer. This is the agent the Router proposal follows through a payment call.

1. **Produce evidence.** The agent starts in a Kubernetes pod. The SPIRE agent, acting as a platform verifier, works from the caller's process to its container, reads the container's image digest and Sigstore signature from the kubelet and container runtime, and issues an SVID bound to a key the instance generated in an isolated signer. One attestation covers both the instance and the software it runs.
2. **Resolve.** The Verifier finds the one active record listing the SVID's SPIFFE ID and confirms the SPIRE server is an approved issuer.
3. **Appraise.** The image digest from the workload attestation, not the SVID itself, matches the signed artifact, which appears in `approved_software_refs`. Both relationships reach `host-platform`.
4. **Issue.** The Verifier signs a 30-minute credential bound to the SVID's key. The evidence stays with the producers and is referenced, not copied.

```json
{
  "credential_id": "arc_3e91c0",
  "format_version": "1",
  "issuer": "https://verifier.example.com",
  "issuer_key_ref": "https://verifier.example.com/keys/2026-10",
  "holder_key_ref": { "jkt": "NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs" },
  "agent_id": "agents/invoice-reconciler",
  "registration_record_ref": { "record_issuer": "https://registry.example.com", "record_id": "reg_19", "record_version": 7, "record_digest": "sha256:9b2e…0d2f" },
  "runtime_instance_id": "ri_04d1",
  "software_hash": "sha256:4f7a…d9f1",
  "assurance": { "instance_to_software": "host-platform", "instance_to_key": "host-platform", "holder_key_protection": "isolated-component" },
  "evidence_refs": ["spiffe://example.com/ns/finance/sa/invoice-reconciler#svid-51a8", "https://ci.example.com/attestations/img-4f7a"],
  "trust_domain": "example.com",
  "issued_at": "2026-09-30T14:00:00Z",
  "expires_at": "2026-09-30T14:30:00Z"
}
```

Variations:

- The pod runs a digest the record does not approve. The Verifier issues nothing (BRG-05).
- The agent presents a service-account token read from its own filesystem. It is not evidence (PSP-09).
- Ten agents share one cloud execution role. The role cannot resolve to a single record, so resolution fails (BRG-02).
- The Verifier issues the credential as a WIT-SVID naming `spiffe://example.com/agents/invoice-reconciler`. Its trust bundle goes only to the Bridge and the Router, so an internal service that accepts SVIDs rejects it (BRG-15).
- A TEE quote arrives whose report data does not commit to the SVID's key. The Verifier cannot link it to the instance and appraises without it (BRG-05).
- The record moves to version 8 and keeps the digest. Credentials naming version 7 remain valid; renewals name version 8 (BRG-12).

## 6. Open questions

- **Verifier placement:** Should the record-aware Verifier sit in the Bridge, as proposed, or in the Passport with the Bridge focused on the principal (Section 1.4)? Either way, the specification needs one stated answer.
- **Naming:** WS4 discussion uses "Agent Credential" for the Verifier's output. Should ODIS adopt it, or keep Agent Runtime Credential to limit churn across ODIS.md? Should the Passport keep its name, given that RATS uses "passport model" for the whole flow?
- **SVID-form credentials:** To issue a WIT-SVID or X509-SVID, the Verifier must act as a SPIFFE issuer. Does it sign in the workload trust domain or its own, and how does that interact with the acceptance limit in BRG-15 and the Router's no-bypass rule (Router: Router isolation)?
- **Several credentials:** Platform verifiers already issue credentials for parts of the picture: an SVID for workload identity, an attestation token for TEE state. Does an instance hold one Agent Runtime Credential that summarizes them, or several credentials that the Bridge combines?
- **Co-location:** When one platform hosts both a platform verifier and the Verifier, as a SPIRE deployment with registration access might, how does its conformance declaration show the two roles, and does §9.2 still count them as separate boundaries?
- **Latency:** Does adding a Verifier ahead of delegation change the CC-03 benchmark boundary?
- **Evidence profiles:** Who publishes the profiles that BRG-06 and BRG-10 depend on? (#21)
- **Sandbox identity:** Does `runtime_instance_id` name a sandbox or each agent it hosts? (Router: Required facts)
- **Revocation subjects:** Event standards lack identifiers for agents, credentials, and runtime instances. ODIS should define them once for this document and the Router. (#25)
- **Profile tags:** The tags in Section 2 are provisional. WS4 leans toward SHOULD where a requirement is not a security invariant; which MUSTs here could relax?

## Glossary

**Agent Registration Record.** An organization's governed record of a logical agent: owner, sponsor, lifecycle state, approved software, approved issuers, and the workload identities that resolve to it ([ODIS §6.1](../ODIS.md#61-agent-registration-record)).

**Agent Runtime Credential.** The short-lived, holder-bound credential the Verifier issues to one running instance, carrying appraisal results (Section 3.2).

**Agent Runtime Evidence.** Signed data or platform results about a running instance, its software, or its holder key, produced by the Passport. Input to appraisal; never required by later layers (Section 3.1).

**Appraisal.** Checking evidence against reference values and appraisal policy to produce results.

**Claims profile.** A definition of the claims a Verifier emits, independent of packaging (e.g., TRACE) (Section 3.5).

**Endorsement.** A statement from a trusted third party, such as a manufacturer or supply-chain signer, that the Verifier uses to judge evidence.

**Evidence producer.** A Passport component outside the agent that produces evidence, such as a workload attestor, CI/CD signer, or hardware root.

**Evidence profile.** A published definition of what a type of evidence proves for each relationship it supports.

**Format profile.** A definition of how a credential's claims are signed and bound to a holder key, such as a WIT-SVID or a Verifiable Credential (Section 3.5).

**Model-visible process.** Any execution context the model can read or influence ([ODIS §1.4](../ODIS.md#14-normative-terminology)).

**Platform verifier.** A Passport component that appraises platform evidence against policy it owns and issues a platform result, such as a SPIRE server issuing an SVID or a TEE remote attestation service issuing an attestation token. It never consults the registration record.

**Reference value.** An expected value, such as an approved digest or launch measurement, against which evidence is appraised.

**Verifier.** A proposed Bridge subcomponent, alongside the Delegation Service, Provider Adapter, and Cache and Revocation Manager, that resolves the registration record, appraises evidence and platform results, and issues the Agent Runtime Credential. It is the only component that reads the record to appraise an agent.
