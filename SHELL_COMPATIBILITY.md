# Shell compatibility and upstream work

## Shell 0.4 baseline

STIR pins public IDAX Shell 0.4.0. The release contains the complete generic extension contract and effective-permissions session surface that STIR previously applied as two temporary patches.

Equivalence was checked against a clean `v0.4.0` checkout: both former patch files apply cleanly in reverse and cannot apply forwards. This proves their changes are already present in the tagged source. The patches are therefore removed rather than stacked a second time.

Generic behavior: manifest routes are validated and unique/nonoverlapping; reserved API/system roots are forbidden. GET/HEAD SPA handlers exist only at declared prefixes. Static assets retain their handlers and protected APIs retain authentication. Frontend matches the manifest route rather than a list of product IDs. SDK exposes activeTenantId; a tenant change remounts the module and locale changes refresh SDK translation context.

Shell 0.4 also uses Core's public tenant-creation capability for local initialization. Runtime does not regain broad tenant INSERT privileges.

## Remaining public-source compatibility patches

Ledger 0.4 still needs the small external-migrations switch. osTRIS 0.4 still needs that switch plus STIR's discovery, provisioning and multi-device public-pilot API surface. Composition sets module migrations off after the one-shot migration jobs. These patches change no economic authority boundary: osTRIS remains the sole policy/authorization/commit authority, and no runtime receives bootstrap database credentials.

The initializer verifies upstream commit, exact origin, patch applicability and unexpected untracked source. A compatibility patch is removed only after an equivalent tagged public release is demonstrated from a clean checkout.
