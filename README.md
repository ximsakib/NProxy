# NProxy
# NProxy — engineering starter

This is a cross-platform Electron application scaffold for NProxy. It includes a desktop UI, tray/menu-bar entry point, a TUN-engine launch adapter, and Windows/macOS packaging targets.

## Important release blockers

The app UI intentionally exposes only connection controls and an administrator update import. The built-in proxy endpoint and protocol are defined in `src/main.js`; the renderer never receives its credentials. Imported administrator updates are signature-checked and encrypted with Electron `safeStorage` on the device.

TUN routing needs elevated privileges. On macOS, Start now asks macOS for administrator authorization before launching sing-box, and Stop asks to terminate that privileged process. Windows launches with its administrator manifest. A production release should replace this direct elevation path with a signed privileged helper or approved Network Extension and complete platform-specific validation.

DNS is resolved using Cloudflare DNS-over-HTTPS through the configured proxy, avoiding Windows' strict-route DNS block. The HTTP CONNECT proxy carries TCP; UDP traffic is not supported by this proxy path, so the config rejects UDP port 443 to encourage apps to fall back from QUIC/HTTP/3 to TCP.

A normal OS HTTP/SOCKS proxy setting is not equivalent to all-traffic routing. This app uses sing-box TUN mode, which changes system routes:
- Windows: the app requests administrator privileges at launch for TUN routing; production still needs a signed helper/service and compatible driver validation.
- macOS: privileged helper / appropriate Network Extension or approved system networking mechanism, code signing, and notarization.
Do not distribute until these are implemented and tested on clean machines.

## Build prerequisites
- Node.js LTS and npm
- Windows build: Windows machine
- macOS DMG: macOS machine (Apple signing/notarization credentials needed for trusted distribution)
- Bundled engines are sing-box v1.14.4 from the official SagerNet release: macOS ARM64 archive SHA-256 `0d8c7f5ed5d587b0f1a5e9772a708ec3bfa4ac4c9329604b01d8d9c2ba406838`; Windows x64 archive SHA-256 `99185d75d314a07dbba4a130c8ad986cbeb629a07927975636b341ca61f991df`. Each installer includes only its platform's engine resources.

## Development
```bash
npm install
npm start
```

## Packaging
```bash
npm run dist:win
npm run dist:mac
```
Cross-building a trusted macOS DMG from Windows is not supported as a reliable release workflow. Build and sign the DMG on macOS.

## Administrator controls and updates
The admin panel is protected by the configured password and automatically locks after ten minutes. It shows the active proxy endpoint, app/engine/platform information, and recent engine logs. Proxy settings are encrypted using Electron `safeStorage`; signed `.npu` proxy updates remain supported. The GitHub release source is stored encrypted in the user's app data.

App updates are checked against the repository's latest GitHub release. Releases must include a signed `NProxy-app-update.json` manifest and the matching `NProxy-Setup-x.y.z.exe` asset. The app verifies the Ed25519 signature and installer SHA-256 before launching the installer. Releases must be publicly readable; private repositories need a separately designed authentication flow.

The update verification public key is `src/update-public-key.pem`; only the administrator's private key can sign updates. The private key is stored locally as `nproxy-update-private.pem`, ignored by Git, and must be backed up offline. Do not include it in source archives or installers.

To create a replacement endpoint package, put the proxy `protocol` (`http` or `https`), `server`, `port`, `username`, and `password` in an ignored `config-update.json` file, then run:

```bash
npm run update:make -- config-update.json NProxy-config.npu
```

Users import the resulting signed `.npu` in Admin controls → Proxy updates. The app verifies the signature and encrypts the imported settings with OS-backed `safeStorage`. To use a different signing key, generate it once with `npm run update:key` before building installers, and keep that private key for later updates.

For an app release, upload `NProxy-Setup-x.y.z.exe` to a GitHub release tagged `vX.Y.Z`, then create and upload the signed manifest:

```bash
npm run app-update:make -- 0.1.3 release/NProxy-Setup-0.1.3.exe
```

The generated `NProxy-app-update.json` must be attached to that same release. Keep the release signing private key private.

The example configuration is a template only. Use only a proxy endpoint authorized by the client / portal owner.

## Credential handling
The default deployment endpoint is embedded in the Electron main process for first launch, but administrators can now change it in the password-protected admin panel. Credentials embedded in a desktop app can be extracted by someone with access to its installed files. Rotate proxy credentials if exposed.

## Security checklist
- Rotate any credentials exposed in screenshots or chat.
- Never expose credentials to the renderer or write them to logs.
- Use Windows Credential Manager and macOS Keychain.
- Sign Windows installers and macOS app/DMG; notarize macOS distribution.
- Pin and verify the network engine version and checksum.
- Fail closed or clearly warn if the tunnel stops unexpectedly; avoid silently leaking traffic.
- Add an administrator-controlled kill switch / DNS leak protection only after platform-specific testing.
