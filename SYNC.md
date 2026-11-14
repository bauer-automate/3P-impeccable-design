# Bauer fork: hardened engine and upstream sync

This repo is a fork of `pbakaus/impeccable`. We run our own build of the engine on client projects. This file lists what we changed and what to check on every upstream sync.

## What we changed

1. **No account service.** `crates/account/src/http.rs` refuses every request unless the account service is on loopback (`127.0.0.1`, `localhost`). This turns off, in one place:
   - `impeccable login`, `logout`, `whoami`
   - DESIGN.md and critique uploads (both on by default upstream once signed in)
   - health, taste, world and setup uploads
   - Pro engine download and dispatch (a signed artifact fetched and run)

   Upstream tests that use a local fake issuer still pass. One new test, `a_remote_account_service_is_never_contacted`, pins the guard.

2. **Our own engine binaries.** The launcher downloads from our releases, and `skill/scripts/engine-checksums.txt` holds our digests, not upstream's. See "Release" below.

## What is not changed

These still make network calls. Control them with environment variables on every machine:

- `IMPECCABLE_NO_TELEMETRY=1` and `DO_NOT_TRACK=1`: choice pings and roll telemetry.
- `concept-seed` calls the roll API at impeccable.style when it has no local catalog.
- `impeccable install` and `update` fetch from impeccable.style.

If a client needs these gone too, patch `crates/context/src/telemetry.rs` and `crates/skills` the same way.

## Sync checklist (every upstream sync PR)

Do not auto-merge. Before you merge, check each item.

1. `ENGINE_VERSION`: note the new version and read the upstream changelog.
2. `git diff <old> <new> -- crates/` for new `ureq`, `Command::new`, `TcpStream` and file writes outside the project.
3. `crates/account/src/http.rs`: the loopback guard and its test must still be there. Re-apply the patch if the merge dropped it.
4. New crates or dependencies in `Cargo.lock`.
5. `skill/scripts/impeccable` and `impeccable.cmd`: download base and checksum checks must still fail closed.
6. `.github/workflows/`: upstream uses a paid runner profile (`namespace-profile-impeccable`) that does not exist here.
7. Rebuild, record new digests, and publish a new release (below) before the sync reaches client machines.

## Release (we build locally, not in Actions)

Run the commands below from a clean checkout of the merged commit.

1. Build Windows on a Windows machine: `cargo build --release --locked -p impeccable --target x86_64-pc-windows-msvc`
2. Build Linux (cloud sessions use this): install `musl-tools`, then `CC_x86_64_unknown_linux_musl=musl-gcc cargo build --release --locked -p impeccable --target x86_64-unknown-linux-musl`
3. Run `impeccable engine-probe` on each binary. It must print `impeccable-engine <ENGINE_VERSION>`.
4. Hash each binary with SHA-256. Replace the matching lines in `skill/scripts/engine-checksums.txt`. Remove lines for targets we do not build (the launcher then refuses them).
5. Create a GitHub Release tagged `engine-v<ENGINE_VERSION>` on this repo. Name the assets `impeccable-linux-x64` and `impeccable-windows-x64.exe`.
6. Point the launcher at it: change the default `IMPECCABLE_DOWNLOAD_BASE` in `skill/scripts/impeccable` and `impeccable.cmd` to `https://github.com/bauer-automate/3P-impeccable-design/releases/download`.

Do step 6 only after step 5 exists. Until then the launcher would fail on every machine.
