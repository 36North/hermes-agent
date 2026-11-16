Adds `plugin-catalog/pandora.yaml`, pinning
`36North/Pandora-for-Hermes` at commit `b2016f1687e3b51dc6daa8c39b52debd2748aca3`
(subdir `pandora`). Public repo, MIT, Windows-only (exercised end to end on
Windows 11). The entry below passes `scripts/validate_plugin_catalog.py`
(`OK: 1 file(s) valid`).

## What it does

Pandora radio inside Hermes: sign in with your own Pandora account, list and
create stations, search, play/pause/skip/volume, and a status-bar chip in the
desktop app. Playback is local via mpv, downloaded on demand from the official
upstream release (see rule 17 below). Agent tools are `pandora_playback`,
`pandora_stations`, `pandora_search`, `pandora_playlists`, `pandora_devices`,
`pandora_feedback`, `pandora_collection`; CLI is `hermes pandora …`.

## Terms of Use (rule 13)

> **Terms of Use.** This plugin uses automated access to pandora.com that may
> violate Pandora's Terms of Use; use at your own account risk.

Unofficial and unaffiliated. No Pandora logo or artwork is bundled; all images
are original. The same sentence is in the README and in the catalog
`description`. The plugin calls the same private web GraphQL endpoint the
pandora.com player uses — no API key, no partner access, no DRM bypass.
Credentials are sent only to pandora.com.

## Disclosures

### Rule 8 — raw desktop bridge, kept deliberately

`rest()` in `pandora/desktop/plugin.js` retries `window.hermesDesktop.api` when
`ctx.rest` fails with a 404/not-found. `plugins validate` flags this as a raw
bridge (warning). It is intentional cross-profile support, not a workaround for
a broken transport:

- The desktop bundle is app-level (`<hermes-home>/desktop-plugins/pandora`), so
  the chip renders in every profile.
- The dashboard backend is per profile. On a gateway serving several profiles,
  every profile other than the launch one runs its dashboard plugins in its own
  plugin host, so `ctx.rest` is scoped to the active profile and 404s when
  Pandora is installed only in the launch home.
- The fallback re-issues the same path on the primary connection so the chip
  keeps working when the user switches profiles.

`rest()` is the single funnel for all nine backend calls (`/state`, `/stations`,
`/search`, `/playlists`, `/devices`, the POST mutations, `/station`,
`/shutdown`). The existing error handlers still turn an unrecovered 404 into
`notInstalledMessage(activeProfile)`, so the install-here banner appears when no
profile has Pandora at all.

Hermes has no all-profiles install/enable today, and agent plugin discovery is
per profile: `collect_directory_manifests()` scans `get_hermes_home()/plugins`,
which resolves to the active profile's home. There is no shared code root that
would make one install visible to every profile, so the fallback is the only way
one Pandora backend can serve the whole app. Removing it would show "install
here" in every profile except the one that owns the backend.

This is a convenience, not a lock-in: a user who wants the full toolset in
another profile can install/enable Pandora there from the Plugin store (or
`hermes -p <profile> plugins install 36North/Pandora-for-Hermes#pandora
--yes-deps --enable`), after which `ctx.rest` resolves natively and the fallback
never fires. That path is already documented in the README's Setup section —
"Plugins install **per profile**" and "1. Store: the app installs the plugin into
the profile it is running and adds the status-bar chip immediately" — so the
fallback is a convenience for the profile that has not opted in, not a
requirement. The password lives in the OS keychain per OS user, so that second
profile needs no new login — it re-logs in automatically and gets its own
Pandora device id. If a public seam for cross-profile plugin requests lands, I
will remove the fallback in favour of it.

### Rule 9 — private core imports

Session state is persisted in `auth.json` under `providers.pandora` via
`hermes_cli.auth`, the same provider-state convention the official Spotify
plugin uses (`NousResearch/hermes-official-plugins`, `spotify/spotify_auth.py`).
There is no public plugin API for a third-party session token. Seven imports are
flagged:

    hermes_cli.auth.get_provider_auth_state
    hermes_cli.auth._provider_state_transaction
    hermes_cli.auth._save_provider_state_to_source
    hermes_cli.auth.clear_provider_auth
    agent.vault_keychain.KeychainUnavailable
    agent.vault_keychain.backend_label
    agent.vault_keychain.os_keyring

Read and clear go through the public helpers. The one write seam is
`_provider_state_transaction` + `_save_provider_state_to_source`, which writes
back to the store the state was read from and never touches `active_provider`.
The keychain helpers are used to report which OS backend holds the password and
to fail cleanly when it is unreachable.

### Rule 11 — keychain read

The password is stored in the OS credential store (Windows Credential Manager
via `WinVaultKeyring`, macOS Keychain, Linux Secret Service) under service
`Hermes pandora`, account = the login email. The entry is per-OS-user, so one
login covers every Hermes profile of that user. It is exposed both through a
plugin-registered Hermes secret source (`pandora_keychain`) and read directly.
Rule 11 names keychain reads explicitly, so this is disclosed rather than
argued away. Two facts a reviewer should weigh:

- Precedent in the current catalog: `hermes-wincred` and `hermes-dpapi-secrets`
  are both listed at `tier: community` and both read the Windows credential
  store directly (`CredReadW` / a DPAPI blob) to resolve secrets at startup.
  Pandora registers the same kind of secret source (`pandora_keychain_source`)
  and additionally reads the entry directly when the profile's secret scope is
  empty.
- The constraint behind the direct read: a store plugin has to work with zero
  user config. The sanctioned `secrets:` orchestrator only delivers when the
  profile's `config.yaml` has a non-empty, explicitly enabled `secrets:` section
  (`env_loader._apply_external_secret_sources` returns early on an empty config),
  so requiring a hand-written block would break first-run sign-in.

If maintainers want a public seam for a plugin-owned OS-keychain credential —
one that resolves with no profile config — I will move to it and drop the direct
read. Nothing here writes to another tool's credential store: the plugin creates
and reads only its own `Hermes pandora` entry.

There is no silent plaintext fallback. If the keychain is unreachable,
`--save-to auto` (the default) refuses to store the password at all — the
sign-in still works until the token lapses. A plaintext `.env` copy appears only
on the explicit `pandora login --save-to env` opt-in, which prints a PLAINTEXT
warning. A successful keychain write removes any existing `.env` credential
lines.

### Rule 17 — no bundled binaries

The package carries no executables, archives or bytecode.
`scripts/scan_plugin_payloads.py pandora` exits 0 with no findings.

### Rule 3 — the mpv download is consent-gated

mpv is the audio engine local playback streams through. It is not bundled, and it
is fetched only when the user asks. Two doors run the same
`pandora_mpv.download()`:

- the chip's **"Set up audio"** button → `POST /api/plugins/pandora/setup`, which
  runs the download in a worker and reports progress through `GET /setup`;
- `hermes pandora setup` — unchanged, so the README sequence still works.

Nothing downloads on plugin load or on playback. The load-time prefetch was
removed and `PANDORA_MPV_AUTO_DOWNLOAD` defaults to `0`; playing without mpv
returns an error naming the setup step instead of fetching. Core has no
post-install hook, so the chip button is what keeps a store install out of a
terminal. The store card description and the README both state that the step is
required and that it downloads mpv (~27 MB, once per PC).

Clause by clause:

- **Explicit consent** — the click or the command above, never a background fetch.
- **Pinned source** — `DEFAULT_MPV_URL` is
  `github-release://mpv-player/mpv/v0.42.0/mpv-v0.42.0-x86_64-pc-windows-msvc.zip`,
  the official upstream asset, resolved by tag + asset name through the GitHub API.
- **Integrity check** — `DEFAULT_MPV_SHA256` pins the archive and is verified after
  download, before extraction. Precedence is `PANDORA_MPV_SHA256` > the in-code
  pin > a remote `.sha256` sidecar, so a sidecar served by whoever serves the zip
  cannot override a trusted pin. The pinned release publishes no sidecar (only a
  GPG `.zip.asc`), so the pin is the effective check. A custom `PANDORA_MPV_URL`
  with no published checksum is refused before any download unless the user sets
  `PANDORA_MPV_SHA256` or opts in with `PANDORA_MPV_ALLOW_UNVERIFIED=1`.
- **Atomic verified install** — extraction goes to `bin.staging-<pid>` beside the
  target, then `os.replace` into place; an existing build is moved aside and
  restored if the swap fails, so a failed `--force` (Windows locks a running
  `mpv.exe`) leaves the working build intact. Only `.exe`/`.com`/`.dll` members and
  the two licence files are copied, flattened to basename — no zip-slip write.
- **No shell interpolation** — `subprocess.Popen(args, …)` with an argument list.

`tests/test_mpv_setup.py` (22 tests, `file://` archives, no network) covers each of
these, including the refusal, the opt-in, a checksum mismatch and the
atomic-failure path. `tests/test_setup_api.py` (10 tests) exercises the real
FastAPI router — including over HTTP with `TestClient` — and pins the consent
behaviour: one worker per click, a second click joins it, an already-present mpv
is a no-op, and a successful install invalidates the readiness cache so the chip
stops offering the button.

Exercised live as well, from an empty state: the owner uninstalled the plugin,
reinstalled it from GitHub at this pin, quit the desktop app, deleted the mpv
directory, and restarted. The chip offered **"Set up audio"**; clicking it showed
the download progress, then the verified SHA-256, then playback worked. That is
the whole consent → pinned source → integrity check → atomic install → play path
with no terminal step and nothing fetched before the click.

## Verification (Hermes v0.23.1, Windows 11, Python 3.14.9)

    hermes plugins validate pandora                 -> Validation passed. (exit 0)
    hermes plugins validate pandora --install-deps   -> Validation passed. (exit 0)
    scripts/scan_plugin_payloads.py pandora          -> exit 0, no findings
        (re-run at the pinned tip b2016f1: 0 bytes of output, exit 0)
    scripts/validate_plugin_catalog.py pandora.yaml  -> OK: 1 file(s) valid
    uv run --python 3.14 --with pytest --no-project \
      python -m pytest tests -q -p no:cacheprovider  -> 164 passed, 1 warning
    hermes -p tinker pandora status                  -> Pandora: signed in /
        Auth: password / Renew: saved credentials / Creds: keychain / Player: idle
    clean-room first run (owner, live)               -> uninstall, reinstall from
        GitHub at this pin, quit app, delete the mpv dir, restart, click
        "Set up audio" -> download progress -> verified SHA-256 -> playback works

Two warnings remain from `plugins validate` (rules 8 and 9), both disclosed
above. No gate failures. The plugin is in daily use on the author's machine at
this pin.

### What the pinned tip contains

The pin is two commits past the mpv change. `26483fc` ("script update") touches
`pandora/scripts/full-wipe-pandora-plugin.ps1` and `STORE.md`; `b2016f1`
("Update STORE.md") is a one-line correction to the store page's verification
table. `git diff 26ebf53 b2016f1 -- '*.py' '*.js' pandora/plugin.yaml` is empty.
The suite (164 passed) and `plugins validate` were run at `26ebf53`, and since
the plugin payload is byte-identical across all three commits, those results
carry over unchanged; the payload scan and the catalog validator were re-run at
the tip.

The script change fixes the optional full-wipe helper, and is worth a reviewer's
attention because it is a credential-hygiene fix:

- it names each profile it uninstalls from and streams the CLI output, instead
  of buffering nine sequential calls behind one pipe (which looked like a hang);
- it sweeps the parked plugin trees that core's startup sweep never reaches on a
  gateway serving several profiles, so a wipe no longer leaves them accumulating;
- it prunes its own `.bak-<timestamp>` backups on success. An `auth.json` backup
  otherwise keeps `providers.pandora` — `auth_token`, `cookies`, `csrf_token` —
  in plaintext after a "full" wipe, which defeats the point of wiping. Backups
  are still created for the duration of the run and kept on failure, so a botched
  rewrite of a file holding other providers' credentials remains recoverable.

Verified by a real wipe on the author's machine (8 homes, all clean afterwards;
both park directories empty; other providers untouched) and by a fixture run
asserting the prune removes wipe-class backups, spares unrelated `.bak` files,
and leaves every live `auth.json` / `config.yaml` / `.env` byte-identical
(sha256 compared before and after).

Interpreter note for anyone reproducing the suite: `python3` on PATH may not
match the plugin environment's ABI. Here `python3` is 3.13.14 while the
environment is 3.14.9 with `_pydantic_core.cp314-win_amd64.pyd`; running
`python3 -m pytest` against it yields 9 spurious failures, all the same
`ImportError: cannot import name 'ArgsKwargs' from 'pydantic_core._pydantic_core'`.
On the matching interpreter the same suite is 164 passed.

## Catalog entry

`pandora.yaml` (beside this file) is the STORE.md template with the final pushed
sha substituted, and is what should land at `plugin-catalog/pandora.yaml`. It is
the only file this PR adds.

Screenshots: one gallery image, `assets/chip-store.jpg` (the chip with its menu
open, 727x675), fetched at the pinned sha and returning HTTP 200. `assets/chip.jpg`
is the same screenshot at half the resolution and is already embedded in the
README, so it is not duplicated in the gallery. The card image is
`assets/card.jpg` (1200x600, 2:1 as the schema asks).

The entry, the card and the screenshot are pinned to the same commit — the
current `origin/main` of the plugin repo, which carries the consent-gated setup
above — so a reviewer reading the pinned commit sees the same code and the same
rationale in `STORE.md`.
