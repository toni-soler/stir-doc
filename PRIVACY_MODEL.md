# Privacy model

Listings are authenticated and tenant-visible, not publicly indexed. Owner IDs are opaque; email, legal identity, credentials and financial risk are not listing data. Optional location is coarse text; do not enter private addresses or contact secrets. No uploads in 0.1.

Future messages and snapshots are party-visible. Legal identity is separate from ParticipantProfile. Marketplace reputation is independent from RiskSubject, Identity Assurance, RiskPolicy and Findings.

Snapshots stay in STIR. Hashing does not anonymize low-entropy data; use private random nonce, selective disclosure and a reviewed retention policy. Logs exclude tokens and request bodies. Moderation, retention and production privacy review remain gates.

0.3: only the AgreementSnapshot's SHA-256 digest crosses into the osTRIS journal (`contractualMetadataDigest`) - descriptions, messages, bios and every other STIR commercial/private field never do. Ed25519 private signing keys are generated client-side and never leave the browser: they are never sent to STIR, never sent to osTRIS, never custodied by either server. STIR only ever relays a signature the client itself produced.
