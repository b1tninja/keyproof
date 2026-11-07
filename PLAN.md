# keyproof: a public register of demonstrated key compromise

Status: **candidate - Tier 1 by merit, wave 3 by cost.** A service and a client library,
built as a module of `certledger` rather than a separate site.

**Name.** `keyproof` returns 404 on both PyPI signals with **zero names one keystroke
away**, checked against a 931,439-name snapshot on 2026-11-05. Rejected: `keyloss` (live
neighbours `keyless`, a reserved-and-empty name, and `keyross`, live since 2026-10-03) and
`untrust` (taken: a TEE deployment auditor that publishes no files at all - a pure name
claim). `krl` is unclaimed, worth knowing because that is OpenSSH's revocation-list
abbreviation, not ours.

## The idea

Publish evidence that a private key is no longer secret, so anyone can stop trusting the
matching public key immediately, without waiting for whoever operates it to act.

The October 2026 eMudhra incident is the precedent, performed by hand. The reporter did not
leak the keys and did not issue a rogue certificate. He **signed the message "We have a
problem here - Wayne" with each of the eight compromised intermediate private keys**,
published a `proof.zip` containing for each key the certificate, its public key and the
signature, and posted the verification commands:

    openssl x509 -in EM-DV-TLS-CA-G3B.crt.pem -noout -pubkey | diff - EM-DV-TLS-CA-G3B.pub.pem
    openssl dgst -sha384 -verify EM-DV-TLS-CA-G3B.pub.pem -signature EM-DV-TLS-CA-G3B.sig message.txt

That is the whole primitive. Anyone can verify it, it proves possession without revealing
the key, it needs no cooperation from the operator, and it costs ten minutes. What it
lacked was a place to put it, a format, a way to tell whether a signature was produced
*for this purpose* or recycled from elsewhere, and a client that would act on it. That gap
is this project.

## Two claims, and only one is safe to automate

**Proof of possession.** The claimant has the target key sign a fixed, purpose-built
statement. This is what Wayne did. It demonstrates control now, it reveals nothing, and it
requires no harmful act.

**Proof of use.** Third-party evidence that the key already signed something: a rogue
certificate logged in CT, a signed binary, an SSH certificate presented at login, a minted
ID token. It needs nobody's cooperation and is often how compromise is actually discovered
- but it establishes only that the key signed *X*, and whether *X* evidences compromise or
ordinary correct operation is a **judgement**, not a verification.

For TLS certificates, **CT is already a proof-of-use registry**: a logged certificate is
durable evidence the CA signed it, which is exactly why CT exists. What CT does not cover
is every other key in the world. So the non-duplicative scope is:

- **possession proofs for any key type**, which nothing publishes today; and
- **use evidence for non-TLS keys**, where there is no transparency log at all.

A registry that accepted "use" claims about TLS certificates would mostly re-report CT and
import the judgement problem for free. It should refuse them and link to CT instead.

## What a proof proves, and what it must never claim

Verifiable, therefore publishable:

> A signature valid under public key K, over this exact byte sequence beginning `KEYPROOF1`,
> was presented on date D and committed to an append-only log.

Not verifiable, therefore never stated:

- **Who** produced it. An owner testing the service, a researcher, a coerced employee, an
  attacker, and a program exploiting a signing oracle are indistinguishable here - and the
  security conclusion needs none of them.
- **Why**, or whether the key was ever secret. A key that can be made to sign
  attacker-chosen bytes is untrustworthy whatever the cause; the cause is not knowable from
  the artifact.
- **That the owner misbehaved.** Not an accusations channel. It publishes a mechanical fact
  about a key, never a narrative about a party - a narrative is what would make the service
  legally indefensible and would suppress the disclosures that make it useful.

The consequence to internalise: **this service cannot invalidate anything.** Only a trust
store, a CA, or a willing client can. What it can do is make compromise cheap to
demonstrate and impossible to quietly deny, and give clients that want a day-zero signal a
place to get one. The incident argues that this is worth having: Chrome and Firefox blocked
on day 0, Microsoft on day 3, Apple on day 5, macOS's own `curl`, `git` and `python3` were
still trusting the keys on day 8, and Electron apps stayed exposed until each was rebuilt.

## The oracle problem, which is why this does not already exist

A site that accepts signatures turns "can this key be made to sign our literal prefix" into
a public, permanent, globally visible verdict. An attacker who **cannot** extract a key but
**can** coerce a signature from it - an HSM command with weak authorisation, a signing
service that accepts caller-supplied bytes, a token issuer with an algorithm-confusion bug,
a padding oracle - could publish a proof and make well-behaved consumers stop trusting a key
that is still secret. That converts a disclosure tool into a denial-of-service weapon
against the infrastructure it is meant to police. This is the central design risk, and a
plan that does not confront it is a demo, not a service.

Mitigations, roughly in order of value:

1. **A fixed, deliberately unnatural prefix.** The signed artifact starts with a literal
   ASCII magic string and contains the target SPKI hash, a UTC timestamp, a submission
   nonce, and the hash of a *new* key the claimant offers as a contact point. A generic
   "sign this digest" oracle must now be able to sign arbitrary attacker-chosen bytes,
   which is a strictly stronger capability than most real oracles have.
2. **Refuse anything protocol-shaped.** Reject artifacts that parse as DER, as an X.509
   structure, as a TLS handshake fragment, as a JWT, or as a Git or package manifest
   signature. If it could plausibly have been a legitimate request once, it is not evidence.
3. **Declare and pin the algorithm.** The submission states its signature algorithm and
   verification runs against the stored bytes in their original encoding - the DER-fidelity
   rule `certledger/TRUST-MODEL.md` records as invariant I3, applied to signatures.
   Ed25519 and ECDSA over the raw message are preferred because the whole message is
   covered; RSA with PKCS#1 v1.5 requires the chosen-ciphertext surface to be stated
   explicitly rather than assumed harmless.
4. **Publish on verification, notify afterwards.** Notifying an operator before publication
   hands a suppression window to the party least likely to want disclosure. The claim is
   committed and published when it verifies; the operator gets a dated, public correction
   channel and nothing more. Entries are never edited or deleted, because a correction that
   can overwrite history is not a correction.
5. **Make probing useless.** Rate-limit per key, never echo a submission back, and ensure a
   rejected attempt reveals nothing about whether the key signed.

Honest residual: mitigation 1 is an argument, not a proof. A key that will sign arbitrary
bytes beginning with our magic string is in practice already untrustworthy, so the attackable
surface is narrower than it sounds - but it is not empty. The answer is not to pretend the
risk disappears; it is that consumers opt in and set their own policy, every claim is
reviewable against a transparent dated record, and the service states plainly that a proof
establishes capability, not blame.

## Adoption ladder, ordered by what can actually be done

The list is worthless without consumers, and the browser trust stores are not ours to
change. So the ordering runs from "we can do this ourselves on day 0" to "someone else must
decide".

1. **Our own libraries, immediately.** `pkix` rejects any chain containing a listed SPKI
   during RFC 5280 section 6 path validation. `certledger` flags every certificate issued
   beneath a listed key, which turns the list into an instant answer to the question the
   incident actually posed - what did these keys issue, and how much is still valid.
   `namespy` refuses encrypted-DNS endpoints on listed keys. `lgc` reports them.
2. **Distribution and platform blacklists, which already exist.** The incident page's own
   remedies were per-platform deny mechanisms: `security.pki.caCertificateBlacklist` on
   NixOS, `certctl untrust` on NetBSD and FreeBSD, the Windows `Disallowed` store,
   `update-ca-certificates` on Debian-family, manual disable on Android. The plumbing to
   consume a list is there; what is missing is a signed, machine-readable, transparent feed
   for it.
3. **SSH, where the machinery is already specified.** OpenSSH's KRL format revokes plain
   keys by public-key hash, versions the list on every modification, authenticates it for
   delivery over untrusted channels, and defines a critical/optional extension rule. Client
   side there is `RevokedHostKeys` (added in 6.8, 2015-03-18, "text-file or KRL-based
   revocation of host keys") and `ssh-keygen -Y verify -r revocation_file`. SSH certificate
   authorities are **invisible to CT entirely**, which makes this the strongest single
   non-TLS case: the distribution format exists, only the publisher does not. The one hard
   constraint is that an export must use detached SSHSIG signatures, because OpenSSH 9.4 and
   later refuse KRLs carrying embedded ones.
4. **Policy engines and CI.** Signature verifiers in a supply chain can consult a signed
   list at admission time without waiting for any root programme.
5. **Application bundles.** Electron, Python, Go and Rust apps shipping their own CA store
   were the incident's real blast radius, and are precisely the population that cannot
   consume OneCRL.
6. **Root programmes.** OneCRL, CRLite and CRLSets are the incumbents, and the incident
   proved they *work*: an outside party with no standing handed the root programme teams
   signatures from eight compromised intermediate keys on 17 October, and Firefox and Chrome
   blocked that same day, Microsoft on the 20th, Apple on the 22nd. **So the answer to "would
   a root programme act on third-party evidence" is demonstrably yes, and fast.** The failure
   was never intake. It was the population that never consults them - applications shipping
   their own CA bundle, Electron apps that only inherit a fix once rebuilt, and macOS's own
   `curl`, `git` and `python3`, still trusting the keys eight days later with a point release
   *undoing* user-applied fixes. That reframes this project: it is not a faster route to
   blocking, because that route exists and is quick. It is **a signal for everyone the blocking
   path cannot reach**, which is a different and more honest value proposition - and it is why
   the adoption ladder starts with our own libraries rather than with the root programmes.

## Scope beyond TLS, which is where the gap actually is

TLS certificates already have CT. The keys with no answer at all are: SSH certificate
authorities, code-signing certificates, S/MIME, package-registry signing keys, OIDC and JWT
identity-provider signing keys (a stolen IdP key mints tokens forever, and nothing publishes
that today), DNSSEC zone keys, RPKI CA keys - where proof-of-possession procedures are
already mandatory practice under RFC 7382 - and device attestation keys in IoT ecosystems.
One format, one log, one list, keyed on SPKI hash rather than on certificate serial number,
because compromise attaches to the key and not to any one certificate it appears in.

## Prior art, and the specific thing that is missing

The primitives are all standardised. Nobody has pointed one at *withdrawing* trust on
evidence a third party can check.

| Mechanism | Status | Polarity |
|---|---|---|
| CRL and OCSP reason codes `keyCompromise (1)`, `cACompromise (2)` | RFC 5280, Proposed Standard 2008-05, 6 verified errata; the reasonCodes are defined in **section 5.3.1**, given distinct treatment in section 6.3.2, and RFC 6960 §4.2.1 imports the `CRLReason` type rather than defining its own | Only the CA can publish them, and clients routinely ignore revocation - the incident demonstrated that rather than theorising it |
| **ACME certificate revocation** | RFC 8555 §7.6, Proposed Standard, 2019-03, current, no errata on the section | **The nearest standardised thing to this project.** `reason` is "One of the revocation reasonCodes defined in Section 5.3.1 of [RFC5280]", the spec's own second example sends `"reason": 1` - keyCompromise - signed with the certificate's own key pair, and it states plainly: "The server MUST also consider a revocation request valid if it is signed with the private key corresponding to the public key in the certificate." So proof-of-possession-triggered keyCompromise revocation already exists. What it needs, though, is **that certificate's ACME server**: it is per-CA, covers only ACME-issuable TLS certificates, the CA "MAY disallow a subset of reasonCodes", and the request has no transparency, no immutability, and no third-party path. Wayne was not the CA. This project is ACME §7.6 with the CA removed from the trust path and the ecosystem widened past TLS |
| Proof-of-possession: RFC 7800 (JWT), RFC 8747 (CWT, section 3.5 titled "Specifics Intentionally Not Specified"), RFC 7030 section 3.4 (EST), RFC 9646, RFC 9733, RFC 6955 | All current Proposed Standards | **Positive**: prove the key to obtain trust. None defines publishing that the key was demonstrated and should be distrusted |
| RFC 9883 section 6, stating possession of a private key | Proposed Standard, current | Explicitly warns the attribute MUST NOT be used to obtain a signature certificate, since proof of possession is trivially "sign this". The oracle concern is already understood in the standards |
| RFC 7382 section 3.2.1, RPKI CPS template | BCP, current | Mandates proof-of-possession procedure in RPKI: the habit exists, the register does not |
| RFC 5011, automated DNSSEC trust-anchor updates | **Internet Standard, STD 74**, 2007-09 | The best precedent: an automatically consumed key-revocation signal built on one flag bit, described as protecting against "N-1 key compromises of N keys". Asymmetric in the right direction - adding trust is slow, removing it is fast |
| OpenSSH Key Revocation Lists | `PROTOCOL.krl` revision 1.7, 2023-07-17; man page dated 2026-10-30 | **The closest existing design in the whole field, and it verified well.** A KRL header carries `krl_version`, which the spec defines as "a version number that increases each time the KRL is modified", plus `generated_date`. Plain keys are revoked by `KRL_SECTION_EXPLICIT_KEY` or by SHA1/SHA256 **hash of the public key blob** - so key-hash revocation is already normal practice, not a novelty here - and hashes "must appear in numeric order, treating each hash as a big-endian integer". Its signature section exists precisely "to provide cryptographic authentication of a KRL that is retrieved over a channel that does not provide integrity protection", and "Implementations that retrieve KRLs over untrusted channels must verify signatures." But see the caveat that governs our export below |
| OpenSSH KRL signatures | `PROTOCOL.krl` section 6 | **A trap for this project: "KRL signatures are not supported by OpenSSH. OpenSSH >= 9.4 will refuse to load KRLs that contain signatures. We recommend the use of SSHSIG (`ssh-keygen -Y sign ...`) style signatures for KRLs instead."** An authKRL export from us must therefore ship a detached SSHSIG, not an embedded signature section, or the newest clients reject it outright |
| OpenSSH KRL extensions | `PROTOCOL.krl` sections 2.5 and 5 | `extension_name` + `is_critical` + contents: a critical extension the reader does not understand must fail the parse, a non-critical one may be ignored. That is exactly the never-silently-degrade rule this project needs, already specified by someone else and worth copying rather than inventing |
| Certificate Transparency, RFC 9162 | Experimental; CT has never been an IETF Standard | Proof-of-use, TLS only |
| Key transparency | No document in the RFC index; drafts and proprietary deployments | Key-directory consistency, not compromise disclosure |
| Rekor, Sigstore | `sigstore` 4.5.1 live 2026-10-20; PyPI name `rekor` reserved-and-empty, zero files | General signed-object transparency log: right plumbing, wrong semantics |
| Keybase | **Live.** The site carries "©2026 Keybase" with an active Tor onion address; it offers `keybase sig`, signing an arbitrary statement with a key, plus identity proofs bound to Twitter, GitHub, Reddit and domains | The closest user experience ever shipped for "prove you control this key, publicly". But it is account- and identity-scoped, has no purpose-bound artifact, is blind to X.509 and SSH CA keys, and publishes nothing a client can consume as a deny-list |

## Artifact

Canonical bytes, signed over exactly these and stored verbatim:

    KEYPROOF1 | spki=<base64 sha256 of the target SubjectPublicKeyInfo>
              | ts=<UTC RFC 3339> | nonce=<32 random bytes>
              | alg=<declared signature algorithm>
              | contact=<base64 sha256 of a NEW key offered by the claimant>
              | note=<optional UTF-8, 512 bytes, inert text, never interpreted>

plus the signature, the target public key in its original encoding, an optional certificate
chain in original DER, and an inclusion proof in the log below. The `note` field is the only
human text; it is displayed as data and never becomes a claim made by the service.

## The register must itself be auditable

Everything the `certledger` trust model demands applies here more sharply, because this list
changes trust decisions:

- Append-only Merkle tree with signed checkpoints, using the RFC 9162 machinery, so
  "entry E existed on date D" is provable and removal is visible.
- Mirrored outside our infrastructure, so it cannot be quietly edited under pressure.
- **No expiry.** Compromise is monotone: a key that has demonstrably signed
  attacker-chosen bytes is never trustworthy again, and rotating the key yields a different
  SPKI. A list that never expires is cacheable forever and cannot be attacked by serving a
  stale copy - the freshness failures that make OCSP fragile simply do not arise.
- The submission endpoint holds **no private key material, ever**.
- Cost is negligible beside the CT mirror: a few hundred bytes per entry, not one of the
  242 million entries in a single half-year log.

## Obligations

| Id | Spec | Status | Where | Evidence | Rung |
|---|---|---|---|---|---|
| `kp.artifact` | this document | [ ] | `format` | `tests/test_format.py::test_canonical_bytes_are_reproducible` (to be written) | pre-ladder |
| `kp.verify.raw` | `certledger/TRUST-MODEL.md` I3 | [ ] | `verify` | `tests/test_verify.py::test_verifies_over_stored_bytes` (to be written) | pre-ladder |
| `kp.oracle-shape` | this document | [ ] | `admit` | `tests/test_admit.py::test_der_and_jwt_shaped_submissions_rejected` (to be written) | pre-ladder |
| `kp.spki-bind` | this document | [ ] | `admit` | `tests/test_admit.py::test_artifact_bound_to_target_spki` (to be written) | pre-ladder |
| `kp.log-append` | RFC 9162 section 2.1 | [ ] | `log` | `tests/test_log.py::test_checkpoint_and_inclusion_proof` (to be written) | pre-ladder |
| `kp.mirror` | `certledger/TRUST-MODEL.md` I9 | [ ] | `mirror` | `tests/test_mirror.py::test_external_copy_matches` (to be written) | pre-ladder |
| `kp.no-expiry` | this document | [ ] | `list` | `tests/test_list.py::test_entry_never_expires` (to be written) | pre-ladder |
| `kp.feed-signed` | PROTOCOL.krl section 6 | [ ] | `feed` | `tests/test_feed.py::test_feed_signature_verifies_offline` (to be written) | rung 2 |
| `kp.ext-critical` | PROTOCOL.krl sections 2.5, 5 | [ ] | `format` | `tests/test_format.py::test_unknown_critical_extension_fails_open_is_impossible` (to be written) | rung 3 |
| `kp.pkix-hook` | RFC 5280 section 6 | [ ] | `pkix` integration | `test_chain_with_listed_spki_is_rejected` (to be written, in `pkix`'s suite) | rung 1 |
| `kp.ssh-export` | PROTOCOL.krl sections 3, 4 | [ ] | `export` | `tests/test_export.py::test_krl_loads_under_openssh_9_and_later` (to be written) | rung 3 |
| `kp.ssh-sshsig` | PROTOCOL.krl section 6 caveat | [ ] | `export` | `tests/test_export.py::test_export_uses_detached_sshsig_not_embedded_signature` (to be written) | rung 3 |
| `kp.ordered-hashes` | PROTOCOL.krl section 4 | [ ] | `export` | `tests/test_export.py::test_hashes_sorted_big_endian` (to be written) | rung 3 |
| `kp.no-key-storage` | this document | [ ] | `admit` | `tests/test_admit.py::test_endpoint_never_stores_private_material` (to be written) | pre-ladder |
| `kp.claims-discipline` | `certledger/TRUST-MODEL.md` section 9 | [ ] | copy, API | review: no attribution, no causation, no "all keys" claim | pre-ladder |
| `kp.probe-silence` | this document | [ ] | `admit` | `tests/test_admit.py::test_rejected_submission_reveals_nothing` (to be written) | pre-ladder |

`Model` in a Spec cell used to mean `certledger/TRUST-MODEL.md` without saying so; the cells now name it. The
`Rung` column places each row on the adoption ladder above: `kp.pkix-hook` (rung 1, "`pkix` rejects any chain
containing a listed SPKI"), `kp.feed-signed` (rung 2, the "signed, machine-readable, transparent feed") and the
KRL rows (rung 3) quote it; `kp.ordered-hashes` at rung 3 and the register preconditions, marked `pre-ladder` because
they must hold before rung 1, are judgements. No test exists for any row, because this directory holds no code; each Evidence name is the test
that would close it, marked `(to be written)`. The `PROTOCOL.krl` cells cite sections without a pinned
revision: the prior-art table read revision 1.7, and pinning the cells to it waits on the owner confirming
that revision.

## Non-goals

- **Not a revocation authority.** It publishes demonstrated facts. Invalidation is the
  consumer's decision, and every consumer sets its own policy about what it trusts.
- **No TLS "use" claims.** Those belong to CT; link rather than duplicate, and refuse the
  judgement call that deciding whether a logged certificate is suspicious would require.
- **No hosting of private key material.** A place to publish a stolen key is a
  crime-facilitating archive, and also useless: a key cannot be un-known. Signing
  demonstrates compromise; publishing bytes only propagates it.
- **No takedown of verified entries.** Pressure to delete is the attack this design exists
  to resist. The correction channel is a dated follow-up entry, never an edit.
- **Not a replacement for OCSP, CRLs, CRLite or OneCRL**, which perform proper revocation
  with CA authority this service does not have and must never claim to have.

## Unverified - check before repeating any of this

- **OpenSSH "marker keys" - I could not find them, and they are probably wrong.** The
  mechanism I described, where a KRL carries a marker key so an owner learns a stolen
  private key is still being used, does **not** appear in `PROTOCOL.krl` revision 1.7, which
  enumerates the section types exhaustively (certificates, explicit key, SHA1/SHA256
  fingerprint, signature, extension) and includes no marker section. It is absent from the
  6.7, 6.8 and 6.9 release notes too, and the ssh-keygen man page section that might describe
  it was truncated in every fetch attempt. So the honest position is: **unverified, likely
  misremembered, and not load-bearing** - the idea of detecting live misuse of a supposedly
  dead key is independently attractive and worth designing on its own merits, but it must not
  be attributed to OpenSSH until someone reads the passage and confirms it. Nothing else in
  this plan depends on it.
- **Keybase - my assumption was wrong, and it matters.** I believed the public service had
  been discontinued. It has not been: the site is live and dated 2026, with an onion
  address and an active install path. `keybase sig` already does the core act - sign this
  statement with this key, publicly, with identity proofs - which means the honest
  gap statement is **not** "nobody publishes proofs of key control". It is: nobody publishes
  proofs *bound to a purpose*, keyed by SPKI, spanning non-TLS ecosystems, and consumable by
  a client as a deny-list. That is a narrower and more defensible claim than the one I had.
  One caveat remains: the page I retrieved titles itself a staging instance, so the
  production front page should be re-checked before this is published as prior art.
- **The 2025 SSH certificate-authority theft** I raised as a motivating case could not be
  corroborated from what I consulted; the Replit article covers a July 2025 agent incident and
  says nothing about an SSH CA key. Treat SSH-CA compromise as a risk *class* that is real by
  construction - SSH CAs are absent from CT - not as a cited event.
- **OCSP and ACME reason codes are now verified**, and one wording correction came out of
  it: RFC 6960 does not define its own reason codes, it imports the `CRLReason` type from
  X.509, which is defined in RFC 5280 §5.3.1 alongside `keyCompromise (1)`. ACME's `reason`
  field references the same list. So there is exactly one source of truth for those values
  and this project should cite §5.3.1 rather than repeating numbers.
- **Whether a root programme would act on a third-party proof is answered by the incident
  itself: yes, and quickly** - outside evidence of possession was reported on 17 October 2026
  and two browser families blocked the same day. What remains genuinely open is narrower and
  harder: **what a consumer should do with a proof that a root programme has not yet acted
  on.** Blocking a live CA key on third-party evidence carries availability risk for
  unrelated certificates under the same key, and no policy for that trade-off is written
  anywhere yet. Rung 1 of the ladder can proceed without resolving it, because our own
  libraries can report a listed SPKI loudly long before they refuse it.



