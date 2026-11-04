# NitroMTA first delivery notes

- Installed NitroMTA **0.2.1** from the GitHub release archive `nitromta-0.2.1-x86_64-linux.tar.gz` (also pulled container `ghcr.io/nitrosend/nitromta:0.2.1` and ran the engine there as uid 65532).
- Host OS/arch for the run: **Linux x86_64** (glibc box). The archive binary refused to start under the host's inherited capability set (`CapInh` too wide / root-in-user-ns), so the working path was the published container image with `--network host`.
- Exact setup/config path used:
  1. `nitromta setup --config /etc/nitromta/config.toml --capture --hostname localhost`
  2. Removed the `[capture]` section; under `[delivery]` set `port = 2525`, `resolver = "127.0.0.1:5353"`, `dane = false`, `mta_sts = false`
  3. `openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out dkim.pem`
  4. Admin ops: `put_account` account 1; `put_identity` name `festouille.test` selector `mail` with that DKIM key; `create_login` user `app`
  5. Started with `nitromta run --config /etc/nitromta/config.toml`
  6. Submitted one message to `proof@nitromta.com` over SMTP `127.0.0.1:2587` with STARTTLS, From `sender@festouille.test`, Reply-To set to a mailbox I control
- Wall time from archive download/setup to the proof receipt email: about **6 minutes** (DNS for the proof MX was the long pole; once resolver/host routing was correct, delivery completed in under a second: metrics showed `nitromta_outcome_seconds` ~0.45s for `delivered`).
- Slowest step: outbound delivery DNS. The environment's default resolver answers `198.18.0.1` for public names, so MX/`mta-proof.nitrosend.com` never reached the real proof listener on port 2525 until a tiny local resolver on `127.0.0.1:5353` returned MX `mta-proof.nitrosend.com` and A `2.25.136.14` (confirmed with a direct SMTP banner `220 mta-proof.nitrosend.com Python SMTP 1.4.6` on that IP:2525).
- Doc/tooling gap with repro: the release Linux binary exits immediately when `CapInh` is non-empty even if effective caps are zero (`nitromta holds capabilities it does not need (CapInh …)`). Repro on a similarly capability-rich sandbox: extract `nitromta-0.2.1-x86_64-linux.tar.gz`, run `./nitromta run --config config.toml` after `setup --capture`. The container image avoids that by starting with a clean capability set. Install docs emphasize packages/systemd/`CAP_NET_BIND_SERVICE` but do not mention this CapInh failure mode for bare archives in restricted agent hosts.
- Change I would make: document a "restricted host / agent sandbox" path that recommends the container image first, and/or accept a `NITROMTA_IGNORE_INHERITABLE_CAPS=1` (or automatically clear inheritable when not root) so the archive binary matches the container behavior without Docker.
