# Dev Box support in KRDC

This branch adds Microsoft Dev Box / Windows 365 Cloud PC connections to KRDC,
with working Teams media optimization. It builds on two components:

- **[libmsdevbox](https://github.com/TGMM/libmsdevbox)** — Dev Box client
  library (Entra sign-in with PKCE, W365 EndUser API listing, RDWeb ARM
  feeddiscovery, signed `.rdp` generation) and the `teamscap` FreeRDP channel
  that implements the Teams VDI 1.0 optimization handshake.
- **KRDC patches** (this branch) — the session-side wiring.

## Prerequisites

1. A FreeRDP 3 build that contains the `teamscap` channel addin — the channel
   implements the VM-side handshake on `com.microsoft.rdc.dvc.webrtc.1`.
   Build instructions are in libmsdevbox's `channels/teamscap/README.md`;
   the short version is: copy `channels/teamscap` into the FreeRDP source
   tree, register it in `channels/CMakeLists.txt`, add the
   `#cmakedefine CHANNEL_TEAMSCAP_CLIENT` line to `config.h.in`, add the
   `{ "teamscap", FreeRDP_BOOL_UNUSED }` row to `dynChannels[]` in
   `client/common/cmdline.c`, then build and install. KRDC must link this
   build (`-DCMAKE_PREFIX_PATH=...`).
2. libmsdevbox built and installed — it is a Rust library with a plain C
   ABI, so its build is separate from CMake: `cargo build --release` then
   `scripts/install.sh` (installs the shared library, `msdevbox.h` and the
   CMake package config). KRDC's `-DWITH_DEVBOX=ON` then resolves it with
   `find_package(MsDevBox 0.1 REQUIRED CONFIG)`.

## User flow

```bash
# 1. sign in (once; tokens refresh silently afterwards)
msdevbox login --upn you@tenant.com

# 2. list Dev Boxes (optional; needs the EndUser scope)
msdevbox login --upn you@tenant.com --scope enduser
msdevbox list --upn you@tenant.com

# 3. generate the signed .rdp for the box you want
msdevbox rdp --upn you@tenant.com --tenant <guid> --resource <guid>

# 4. connect
krdc ~/devbox.rdp
```

KRDC opens the file, asks once whether to trust its cloud routing properties
(ARM gateway), and remembers the answer per host. During the connection:

- the full routing payload is applied (ARM gateway transport, load balance
  info, sign scope),
- the RDSAAD access token comes from the shared Dev Box token cache —
  sign-in happened in step 1, so no prompt appears; without a cached token
  FreeRDP falls back to its own sign-in,
- the `teamscap` channel answers the Teams VDI 1.0 handshake, so Teams in
  the Dev Box reports **"Teams is AVD Media Optimized"**.

## Patch structure

| Commit | Contents |
|---|---|
| CMake | `WITH_DEVBOX` option; `find_package(MsDevBox 0.1 REQUIRED CONFIG)` against the installed package (Rust library with a C ABI) |
| Preferences | `RdpHostPreferences::rdpFilePath` — the per-host remembered connection file |
| Factory | carry the file path through `ConnectionFile`; consent dialog + remember for ARM-gatewayed files |
| Session | apply routing payload via `freerdp_client_parse_rdp_file`; enforce ARM transport + RDSAAD; `GetAccessToken` from the token cache; add the `teamscap` dynamic channel |

Everything is guarded by `#ifdef HAVE_DEVBOX`; building with
`-DWITH_DEVBOX=OFF` restores stock behavior.

## Notes for upstreaming

- The consent flow deliberately reuses the existing gateway-warning pattern:
  applying routing properties from a file is the same trust decision.
- `GetAccessToken` is the acknowledged TODO in `RdpSession::clientContextNew`
  (`// instance->GetAccessToken = RdpAADView::getAccessToken;`); this branch
  resolves it for the cached-token case.
- The EndUser listing and sign-in UI intentionally live outside KRDC, in the
  `msdevbox` tool — KRDC stays a viewer, consistent with its architecture.
