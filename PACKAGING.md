# Packaging
 
How the desktop app is built, signed, packaged and published. Nothing here exists yet: this is the target design, modeled on a release process that already works for a similar native Rust app. Build it up as the app matures, and replace each "planned" note with the real command once it exists. Tool versions are placeholders to pin when you set up CI.
 
Application assets and packaging recipes live in `packaging/`. A single configuration file, `packaging/packages.yaml` (planned), declares the targets, package contents, dependencies and downstream destinations, so one place says what ships.
 
## Targets
 
| Platform | Artifacts | Notes |
| --- | --- | --- |
| Linux amd64, arm64 | `.tar.gz` portable archive, DEB, RPM, AppImage | Built on pinned Ubuntu 24.04 runners |
| macOS | Universal `.dmg`, signed and notarized | Homebrew cask points at the dmg |
| Windows (later) | Signed `.msi`, `.zip` portable | Not part of the first releases |
 
Public download names, package names and the installed executable use one name, set in `packages.yaml`. Linux packages install that name as the only executable.
 
## Principles
 
- **One source of truth.** The configuration file declares everything. Recipes for downstream channels (Homebrew, AUR) are generated from the release's `checksums.txt`, never hand-edited per release.
- **Package scripts never write into user homes.** Per-user setup happens when the app launches. Uninstalling a package never deletes the user's settings or state.
- **Verify what you ship.** Every package is installed, launched and removed in a clean environment before it is published. A successful build is not a tested package.
- **Report coverage honestly.** An install check proves installation and library resolution. It does not prove the app works on a running desktop against a real store.
- **Signing is human-reviewed.** Anything touching certificates, secrets or the release workflow needs maintainer review (see CLAUDE.md).
## Linux
 
### Desktop integration
 
Linux builds provide a desktop entry and an icon, named to match the application's window class and the desktop-entry ID the toolkit reports. Set the window class to match, or the dock and task switcher will show a generic icon. Ship:
 
- `packaging/linux/<app>.desktop`
- `packaging/linux/<app>.svg`
### Packages
 
Build DEB and RPM with nFPM (or `cargo-deb` and `cargo-generate-rpm` if you prefer Cargo-native tools), driven from the configuration file. The binary comes from the release build, and only small assets (desktop entry, icon, licence) come from the checked-out repository.
 
Declare runtime dependencies explicitly. Tools that inspect the ELF file find only linked libraries. GUI toolkits load Wayland, X11, xkbcommon and OpenGL libraries at runtime, so those must be listed by hand. Keychain access needs a Secret Service provider (for example `gnome-keyring` or KWallet) at runtime. The app must degrade clearly when none is available, with a visible message and never a plaintext fallback.
 
Baseline: build on Ubuntu 24.04 runners (glibc 2.39), and target Ubuntu 24.04 and Debian 13 and newer, and a recent Fedora. Library package names differ between distributions, so map them per format. Revisit the baseline each year.
 
### Commands (planned)
 
```
mise run package:validate
mise run package:build -- --release v1.2.3 --target linux-amd64 --target linux-arm64
mise run package:build -- --release v1.2.3 --target appimage-amd64 --target appimage-arm64
```
 
Local builds also need nFPM, `bsdtar`, `readelf`, and for AppImage `mksquashfs` (squashfs-tools) and `zsyncmake`. Outputs go to `dist/packages/1.2.3`.
 
### Install tests
 
For every package, in clean Ubuntu 24.04, Debian 13 and current Fedora containers on native amd64 and arm64 runners:
 
1. Create a settings fixture in the user's config directory.
2. Install the package, then run `<app> --version`.
3. Load the GUI libraries with `dlopen`, and check the desktop entry and icon.
4. Remove the package and confirm the settings fixture survived.
Repeat locally on the matching architecture with `bash packaging/test-install.sh ubuntu:24.04 dist/packages/1.2.3`.
 
### AppImage
 
Build AppImages in the release's Linux build jobs, not in a separate packaging workflow, so `checksums.txt` and its signature cover them. Embed update information so AppImageUpdate and similar tools find the newest release's `.zsync` file and download only changed blocks. The release check must refuse a stable release when an AppImage is missing, when its embedded update string is wrong, or when the `.zsync` file does not describe its AppImage. Verify a built AppImage with `python3 packaging/check-appimage.py <file> x86_64`.
 
### Flatpak (optional, later)
 
Flatpak needs its own application ID and sandbox permissions for network access and the Secret Service portal. Treat it as a separate project once the native packages are stable.
 
## macOS
 
### Build and sign
 
1. Build `aarch64-apple-darwin` and `x86_64-apple-darwin`, then combine them with `lipo` into a universal binary.
2. Assemble the `.app` bundle with its `Info.plist`, icon and version, then package it as a `.dmg`.
3. Sign with a Developer ID Application certificate, with the hardened runtime on.
4. Notarize with Apple, staple the ticket, validate it, and only then record the final checksum.
Notarization needs a paid Apple Developer account. Configure these repository secrets, which the release job exposes as environment variables:
 
- `APPLE_CERTIFICATE_P12`: base64 PKCS#12 Developer ID Application certificate and private key.
- `APPLE_CERTIFICATE_PASSWORD`: the export password.
- `APPLE_SIGNING_IDENTITY`: the exact `Developer ID Application: Name (TEAMID)` identity.
- `APPLE_ID`, `APPLE_TEAM_ID`, `APPLE_APP_PASSWORD`: Apple email, Team ID and an app-specific password.
A complete set enables signing and notarization. An incomplete set fails the build instead of shipping something half-signed. With no values, local builds stay unsigned. Secret changes apply to future builds only, and published downloads keep their original signatures.
 
### Test without publishing
 
Build the dmg on a Mac from prepared inputs into a throwaway output folder (`dist/macos-packages-test`), then install it, launch with `--version`, run `codesign --verify --deep --strict` and `spctl --assess`, and uninstall.
 
### Homebrew
 
Generate a cask from the release's `checksums.txt`, stage it, then on a native macOS runner install it, launch with `--version`, check the signature and uninstall. Publish to the tap only after that check passes. Automation needs `PUBLISH_HOMEBREW=true` and a tap token.
 
## Windows (later)
 
Ship a per-user `.msi` (WiX) and a portable `.zip`. Sign the executable and the installer with Authenticode and a trusted timestamp. Options for the signing identity are a cloud signing service or an EV/OV certificate, and choose when you get there. The installer should keep one upgrade code forever, install per user without elevation, close a running installed copy before an update, and never close a portable copy. Reputation with SmartScreen builds over several signed releases, so keep signing every release with the same identity.
 
## Portable archives
 
The release workflow puts a marker file next to the executable in the Windows `.zip` and the Linux `.tar.gz`, such as `packaging/<app>-portable.txt` containing a fixed string. The updater reads it to decide that the copy may replace its own executable. On Windows, a portable copy keeps its settings beside the executable. Never ship an installer marker next to a portable copy, because the installer marker wins and the copy would update through the package. Once released, keep the marker's name and contents unchanged, since older releases look for exactly that file.
 
## Release flow
 
1. Bump the version in `Cargo.toml` and update the lockfile with a build.
2. Commit written notes at `packaging/release-notes/vVERSION.md`. CI requires notes for the current version, and the release workflow publishes that file instead of a generated summary.
3. Wait for every required CI job on the release commit, on Linux and macOS.
4. Push the `v*` tag. The release workflow builds the binaries, `checksums.txt` and its signature, then the packages. Package checksums are listed separately from the binary checksums unless the package is built in the same job, as AppImages are.
5. Verify the published notes, download links and checksums. A prerelease stops here and does not update any package channel.
6. For a stable release, update the website download page, then the Homebrew cask and Linux repository recipes from `checksums.txt`. Never point a download page at files that do not exist yet.
`packaging/release-names.py DIST TAG` (planned) checks the release's downloads and writes their checksums. Run its tests when you change this step. Published historical downloads are never rewritten, and release tags stay immutable.
 
## Updater (later)
 
If the app self-updates, define the rules before building it: only replace the executable from a signed `checksums.txt`, update installed copies through their package type, and allow portable copies to replace themselves only when the portable marker is present. Do not build an updater before the first packaged release.
 
## Secrets and automation
 
Enable only the destinations you have configured, and keep each one behind a repository variable such as `PUBLISH_HOMEBREW=true` or `PUBLISH_AUR=true`. Store signing secrets in environment-scoped secrets that deploy only from `v*` tags. Rotate app-specific passwords and tokens on a schedule and note their expiry in the release checklist.
 
## Changing the tooling
 
Change the packaging tool's version in `packages.yaml`, the matching workflow reference and every CI pin together. Applications should not need a packaging lockfile or wrapper.
 
Claude finished the response
Closed tab CONTRIBUTING.md
