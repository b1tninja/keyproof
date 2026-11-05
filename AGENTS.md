# Instructions for coding agents

**Plan only. No code, no package, no runnable command in this directory.** Do not scaffold
one: the design risk here is not implementation difficulty, it is publishing something that
cannot be unpublished.

## Read in this order

`PLAN.md` in full. Two sections are load-bearing and must not be summarised away: the
possession-versus-use distinction (which is why TLS "use" claims are refused - CT already is
a proof-of-use registry), and the oracle problem (which is why this service does not exist
yet, and why an implementation that skips the artifact-shape checks turns a disclosure tool
into a denial-of-service weapon against CAs).

## Specification discipline

Verified this session, and the framing depends on it:

- **`rfclib.read("RFC5280§5.3.1", view="normative")`** for the reason codes. RFC 6960 does
  not define its own - it imports the `CRLReason` type from X.509. Cite §5.3.1, do not repeat
  numbers.
- **`rfclib.read("RFC8555§7.6")`** - the closest standardised precedent. ACME already accepts
  a revocation request with `reason: 1` (keyCompromise) signed by the certificate's own key,
  and states the server MUST consider such a request valid. It is per-CA, TLS-only, the CA may
  disallow reason codes, and it has no third-party path and no transparency. That is precisely
  the gap here: Wayne was not the CA.
- `rfclib.info("RFC9162")` / `errata(...)` if a claim touches CT. RFC 6962 is Obsolete.
- OpenSSH's KRL format is **not an RFC**; read `PROTOCOL.krl` and the release notes. Note the
  verified trap: OpenSSH 9.4 and later refuse KRLs with embedded signatures, so exports use
  detached SSHSIG.
- Anything in `PLAN.md` marked unverified stays unverified until someone reads the source. In
  particular "marker keys" could not be found in `PROTOCOL.krl` or in five release notes and
  must not be attributed to OpenSSH.

## Invariants

1. **No attribution, no causation, ever.** A proof establishes that a key can be made to
   sign. Not who made it sign, not whether the key was ever secret, not that anyone
   misbehaved. Publishing a narrative is what would make this service legally indefensible
   and would suppress the disclosures it exists to collect.
2. **Never store private key material.** The endpoint accepts signatures, not keys. A place
   to publish a stolen key helps nobody and cannot be un-published.
3. **Refuse protocol-shaped artifacts.** Anything that parses as DER, an X.509 structure, a
   handshake fragment or a JWT is rejected: it could have been produced legitimately once.
4. **Entries are immutable.** Corrections are new dated entries. A correction channel that
   can overwrite history is a takedown channel with better wording.
5. **Publish on verification, notify afterwards.** A pre-notification window is a suppression
   window held by the party least likely to want disclosure.
6. **No "invalidated" language.** This service cannot invalidate anything. It publishes
   demonstrated facts; consumers set their own policy about trusting a listed SPKI.
