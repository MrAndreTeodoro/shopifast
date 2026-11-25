<h1 align="center">Shopifast</h1>
 
<p align="center"><strong>Your Shopify store, native and fast.</strong><br>A lightweight desktop app for Shopify merchants on Linux and macOS.</p>
<p align="center">
  <a href="CONTRIBUTING.md">Contribute</a> ·
  <a href="PACKAGING.md">Packaging</a> ·
  <a href="CLAUDE.md">Agent guide</a>
</p>
Shopifast is a Shopify admin client written in Rust with [egui](https://www.egui.rs). It lets a merchant manage the day-to-day running of a store (products, collections, channels, catalogs and price lists, menus and reports) without opening the browser, and it has no browser engine.
 
> **Status: early development.** The project is in a feasibility spike. Nothing below is released yet, and the feature list describes what the app is being built to do. Download links and install commands will be added when the first packaged release exists.
 
## What it will do
 
| Area | What you will be able to do |
|---|---|
| **Products** | Browse, search, filter and edit products and variants in a fast table, with inline edits and multi-select. |
| **Bulk edits** | Change prices, status, collections and channel availability across many products, with a preview of every change before anything is sent. |
| **Collections** | Create and edit manual and automated collections, and add or remove products. |
| **Channels** | See and change which sales channels each product is published to. |
| **Catalogs and price lists** | Review and edit catalogs and their price lists. |
| **Menus** | Edit the store's navigation menus. |
| **Reports** | Run reports where the store's plan and permissions allow it. |
| **Offline-friendly** | Work from a local cache that syncs incrementally, so the app opens instantly and stays responsive on large catalogs. |
 
## Designed to be safe
 
The app can change a live store, so safety is part of every feature:
 
- **Preview first.** Every change shows a per-item diff of what will change, and nothing is sent until you confirm.
- **Undo.** Previous values are saved before each bulk edit, and you can revert from the app.
- **Dry run.** Run any change through the full pipeline without calling Shopify.
- **Conflict detection.** If a product changed in Shopify since your last sync, the app stops and shows you instead of overwriting it.
- **Read access first.** The app asks for read permissions at sign-in, and asks for permission to make changes later, one area at a time.
- **Your token stays on your computer.** Access tokens are stored in your system keychain, never in a file, and the app has no telemetry and no hosted backend that stores merchant data.
## How it works
 
- **Native.** A Rust app with no browser engine, aiming for fast startup, low idle use and a small binary.
- **Local-first.** The interface reads from a local SQLite cache. A sync engine updates it incrementally, and bulk operations handle large catalogs.
- **Documented API only.** It talks to Shopify's Admin GraphQL API, and does not scrape the admin or use undocumented endpoints.
- **Cross-platform.** Linux and macOS first, Windows later.
## Connecting a store
 
The sign-in flow is still being designed. The plan is to open your browser to approve access, then return to the app through a local redirect. A custom app token for your own stores will also work. You will need a Shopify store and permission to install apps on it.
 
What the app can do depends on your Shopify plan and the permissions you grant. Some operations, such as reports, may not be available on every plan.
 
## Building from source
 
You need Rust (stable) and, on Linux, a few system libraries for the window and keychain. The toolchain is pinned with [mise](https://mise.jdx.dev):
 
```sh
mise install
mise run check
```
 
`mise run check` runs formatting, lint and tests. The exact system packages for Linux will be listed here once the first build exists. To look at the interface without a Shopify store, a demo mode with sample data is planned:
 
```sh
cargo run --features demo -- --demo
```
 
## Repository layout
 
```
crates/
  shopify-api/   GraphQL client, rate limits, generated types
  store-core/    domain models, SQLite cache, sync engine, mutation queue
  app-ui/        egui views and widgets
  desktop/       the application binary
connector/       Shopify-facing app (install and sign-in)
fixtures/        recorded Shopify responses for tests
docs/            specs and reference pages
```
 
## Contributing
 
Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening an issue or pull request. If you use a coding agent, point it at [CLAUDE.md](CLAUDE.md). Release packaging is described in [PACKAGING.md](PACKAGING.md).
 
## Acknowledgements
 
Built with [egui](https://www.egui.rs). The interface takes its visual direction from Shopify's Polaris design language.
 
Shopifast is an independent project and is not affiliated with or endorsed by Shopify. Shopify is a trademark of Shopify Inc.
 
Licensed under the licence in [LICENSE](LICENSE) (to be chosen).
 
Claude finished the response
