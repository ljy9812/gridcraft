# GridCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `gridcraft-web` build — the same static site published as
`gridcraft-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from the
HAP's own `rawfile`: no network origin, no server-side code, no Rust/wasm
toolchain involved.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.gridcraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks — attached (document listener) and programmatic/detached (`HTMLAnchorElement.prototype.click` override) — are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` (with a filename field) |
| Saving (FS-Access) | the wasm's `showSaveFilePicker` path is neutralized at page start: this ArkWeb build bridges that API to a select-only picker, and the app's tested TypeError fallback then goes through the download bridge above |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile (latest release)
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.gridcraft.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23, physical keyboard +
touchscreen): install & launch, English-native UI, text input,
open→edit→save→reopen round-trip (two cycles) with byte-verified OOXML
output, ≥3 min continuous interaction.

Known limitations:

- CJK text shows as tofu — the release wasm embeds no CJK font face and
  exposes no runtime font channel. Typed CJK content is preserved
  byte-exact in the saved `.xlsx` (sharedStrings).
