# Public software boundary

Any instance must build/run using public sources, public documented artifacts and locally generated secrets. Core 0.4.0 is a public binary exception to source reproducibility under its own license. STIR never builds private Core source.

Prohibited: private source repositories, private Maven artifacts, private Docker images, private NPM packages, undisclosed proprietary components, private IDAX Core source, versioned secrets, machine-specific absolute paths, file dependencies and manually compiled unreproducible artifacts.

Checklist:

- Every upstream Git input is public, anonymous and commit-pinned.
- Maven resolves from public repositories with clean cache.
- NPM lock URLs are public and clean install succeeds.
- Docker contexts build source, never developer binaries.
- No local Maven proxy or credentials required.
- No host path, private URL, secret or file dependency.
- Public licenses/notices preserved.
- Secrets and vendor checkouts Git-ignored.
- Empty-volume stack, login and tenant isolation tested.
- Hostname is configuration, not application logic.

Run stir-main's boundary audit. Review every reported match before publishing. Git pinning does not freeze container/Maven transitives: release image digests and SBOM remain future production gates.
