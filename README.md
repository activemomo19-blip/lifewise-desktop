# LifeWise desktop

LifeWise desktop brings your habits, schedule and goals together in a dedicated app. Sign in with your existing LifeWise account at **https://my.lifewiseapp.com**.

## Run and build

Use Node.js 24 LTS and npm. From this folder:

```sh
npm ci
npm test
npm start
npm run build:mac:test
npm run build:win
```

`build:mac:test` produces a universal Apple Silicon/Intel **release/LifeWise-1.0.0.dmg** using a local ad-hoc signature. This is a test package, not a notarised public release. `build:win` produces **release/LifeWise-Setup-1.0.0.exe**, an NSIS per-user installer with Start Menu/desktop shortcuts and normal uninstall. Build Windows on a Windows machine/runner; the provided Actions workflow does that without changing the app.

For future versions, change `version` in package.json, update the lockfile with `npm install --package-lock-only`, test, then rebuild. Installers are in `release/`. `npm run build:mac` uses the normal signing configuration; for local unsigned testing use the explicitly named test script.

## Signing and public release

macOS public distribution requires a **Developer ID Application** identity (with private key), plus Apple notarisation credentials. An iOS Apple Distribution certificate is not a replacement. Configure the signing identity/certificate through electron-builder's supported `CSC_LINK`/`CSC_KEY_PASSWORD` or your unlocked keychain, and notarisation through `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID` or a supported App Store Connect API key. Do not commit credentials. The normal configuration keeps hardened runtime enabled; the separate ad-hoc test configuration does not use hardened runtime because ad-hoc and vendor framework signatures have no shared signing team. This does not disable Gatekeeper or change machine security settings.

Windows public releases should be Authenticode-signed with a publisher certificate or approved signing service. Without one, the installer is unsigned and may show Windows reputation/publisher warnings. Never claim an unsigned package is signed or bypass OS security warnings.

For manual notarisation using a local Keychain profile (no password in source or shell arguments):

```sh
# Run once, entering your own Apple ID/team and the app-specific password securely.
xcrun notarytool store-credentials LifeWise-Desktop
# After building a Developer ID signed DMG:
xcrun notarytool submit release/LifeWise-1.0.0.dmg --keychain-profile LifeWise-Desktop --wait
# Only after Apple reports Accepted:
xcrun stapler staple release/LifeWise-1.0.0.dmg
xcrun stapler validate release/LifeWise-1.0.0.dmg
```

If a wait times out, use `notarytool info` or `notarytool wait` with the returned submission ID. Do not upload another copy merely because Apple's processing is still in progress. Record final checksums after stapling, then test the final package and publish its permanent release asset.

## Security and sessions

- Sandboxed renderer, context isolation, no Node integration, no webview and no exposed preload/IPC APIs.
- Same exact HTTPS member origin stays inside LifeWise; other ordinary web/email links open in the default browser. Privileged and malformed protocols are denied; popup windows are denied.
- Normal cookies and local storage persist in the `persist:lifewise-member` partition under Electron's LifeWise user-data directory. Quitting, reopening and upgrades do not clear them. No browser cookies/tokens are copied into the app.
- Camera/microphone/location are only offered to the member origin with a visible permission prompt. No background services, updater, tray widget or custom native notifications.
- Existing live web changes are visible on subsequent page loads. Electron/security updates still require rebuilding the desktop wrapper.

## GitHub build source

The repository's `desktop-source.zip` contains this editable project (excluding node_modules, local QA screenshots/logs, release files and credentials). Extract it to work locally. The Windows workflow extracts that exact archive, runs the tests, builds the installer and checks silent install/uninstall on a Windows runner. The local extracted project is the source of truth; regenerate the archive from the same explicit file list before a future CI build. A public release must only be connected to the marketing site's central download config after signing/QA and unauthenticated HTTPS download checks.

## References

- https://www.electronjs.org/docs/latest/tutorial/security
- https://www.electronjs.org/docs/latest/api/session
- https://www.electron.build/docs/mac/
- https://www.electron.build/docs/nsis/
