# FORK-NOTES — chirag127/screenpipe personal-use fork

Branch: `personal-degate`. Base: upstream `screenpipe/screenpipe@main`.

## Why this fork

Unlock the desktop app's paid/pro gates for personal offline use. No account,
no subscription, no cloud dependency required. Source-available license permits
personal-use modification.

## De-gate patches (commit a8c44fd42)

Three minimal flips, no other changes:

| File | Change |
|---|---|
| `apps/screenpipe-app-tauri/lib/app-entitlement.ts:101` | `isDevBillingBypassEnabled()` always returns `true` — bypasses `AppEntitlementGate` (login + subscription wall) for all builds |
| `apps/screenpipe-app-tauri/src-tauri/src/store.rs:1911` | `app_entitled_or_dev()` always returns `true` — bypasses Rust-side entitlement check in release builds |
| `apps/screenpipe-app-tauri/lib/hooks/use-settings.tsx:587` | `makeDefaultPresets()` seeds kilo.ai keyless preset instead of `screenpipe-cloud` |

Effect: app launches without login, records immediately, all features available.

## Keyless LLM default

Primary: **kilo.ai** — `https://api.kilo.ai/api/gateway/v1`, model `kilo-auto/free`.
Fallback: **pollinations** — `https://text.pollinations.ai/openai`, model `openai`.
g4f excluded (broken/unreliable). Both are keyless; no API key needed.

The preset type is `custom` (OpenAI-protocol), so any other OpenAI-compatible
endpoint (Ollama, LM Studio) can be swapped in via Settings → AI Presets.

## GHA build (degate-build.yml)

Trigger manually or on every push to `personal-degate`:

```powershell
gh workflow run degate-build.yml --repo chirag127/screenpipe --ref personal-degate
```

Watch:

```powershell
gh run list --repo chirag127/screenpipe --branch personal-degate --limit 3
gh run watch <run-id> --repo chirag127/screenpipe --exit-status
```

Jobs: Windows x64 NSIS installer + Linux x64 AppImage. ~60-90 min (cold Rust compile).

Artifacts download from:
- Workflow run artifacts (always): `gh run download <run-id> --repo chirag127/screenpipe`
- Private releases repo (when `RELEASES_PAT` secret is set): `https://github.com/chirag127/releases`
  — tag format `win-<YYYYMMDD>-<short-sha>`, e.g. `win-20260812-a8c44fd`.

## Signing (Authenticode, self-signed)

Three secrets set on `chirag127/screenpipe`:

| Secret | Content |
|---|---|
| `WINDOWS_CERT_PFX_BASE64` | Base64-encoded .pfx |
| `WINDOWS_CERT_PASSWORD` | PFX password |
| `WINDOWS_CERT_THUMBPRINT` | `A8A0891DF3B936E09B256378D9DA5B2107EB5260` |

PFX origin: `~/.keys/oriz-codesign.pfx` on the origin machine.
Password: workspace sops vault key `SCREENPIPE_CODESIGN_PFX_PASSWORD`.

The build signs the NSIS installer with `signtool /sha1 <thumbprint>` + DigiCert timestamp.

**One-time per-machine trust step** (run once as admin before installing):

```powershell
Import-PfxCertificate `
  -FilePath "$HOME\.keys\oriz-codesign.pfx" `
  -CertStoreLocation Cert:\LocalMachine\Root `
  -Password (Read-Host -AsSecureString)
```

Or if you only have the cert (not the pfx):

```powershell
Import-Certificate -FilePath oriz-codesign.cer -CertStoreLocation Cert:\LocalMachine\Root
```

Without this, Windows will show a SmartScreen warning but still allow install ("More info → Run anyway").

## Release publish to chirag127/releases

Requires `RELEASES_PAT` secret on `chirag127/screenpipe` (PAT with `repo` scope for `chirag127/releases`).
If absent, the workflow still uploads the installer as a workflow artifact but skips the release.

To add the PAT:

```powershell
gh secret set RELEASES_PAT --repo chirag127/screenpipe --body "<your-pat>"
```

## Syncing with upstream

```bash
git remote add upstream https://github.com/screenpipe/screenpipe.git
git fetch upstream
git rebase upstream/main
# resolve conflicts (patches are minimal — unlikely to conflict)
git push origin personal-degate --force-with-lease
```

Keep patches minimal and rebased so upstream features flow in cleanly.

## License

Upstream: source-available (screenpipe custom license). Personal use permitted.
This fork and its builds are **private, personal-use only — do not redistribute**.
No PR to upstream (patches are personal-use gate removals, not upstreamable).
