# Contributing
 
This is a native desktop app that lets Shopify merchants run day-to-day store operations (products, collections, channels, catalogs and price lists, menus, reports) without opening the browser. Changes should improve the desktop app without adding a browser engine, a Shopify-scraping layer or a service that stores merchant data.
 
CLAUDE.md is the companion for coding agents. This file is the canonical product and contribution policy, and CLAUDE.md follows it.
 
## Before opening an issue
 
Search open and closed issues first. For a bug, use the bug form and include the requested log and exact steps to reproduce it. Reports without enough information to investigate may be closed.
 
For a feature, explain the merchant's problem, not the solution. Discuss large changes in an issue before writing code. Existing code does not guarantee that a feature fits the project.
 
Some boundaries come from Shopify or from the plan a store is on:
 
- The app uses Shopify's documented Admin GraphQL API only. It does not scrape the admin, reuse browser sessions or call undocumented endpoints.
- What a merchant can do depends on their plan and on the scopes they grant. Some operations, such as reports through ShopifyQL, may be unavailable on some plans. A request the API cannot serve is answered with the reason and closed.
- The app does not embed a browser engine, add telemetry, or introduce a service operated by the project that stores merchant data.
- Proposals that depend on bypassing Shopify's rate limits, permissions or terms are out of scope.
`docs/_reference/what-shopify-allows.md` lists what each API and plan offers, and the requests none of them can serve, with the reason for each. A request in its last section is answered from there and closed.
 
Duplicate, out-of-scope or incomplete issues may be closed with a short explanation.
 
A bug can be closed once its fix is on main and the relevant checks pass, with the commit and release status stated. Reporter confirmation is welcome but not required for closure. Reopen the issue if the problem persists after updating.
 
## Design principles
 
- **Native and fast.** Startup time, idle work, memory use and binary size are product features. Keep the UI thread free of network and disk waits.
- **Focused.** Prefer a complete, coherent workflow over a collection of settings, modes and speculative features.
- **Safe by default.** The app can change a live store's prices and catalog. Previews, undo and dry-run are part of every mutating feature, not extras.
- **Honest integrations.** Use the Admin API for what it supports. Do not scrape, impersonate capabilities, bypass technical protections, or silently replace one source with another.
- **Cross-platform by default.** Linux and macOS are supported, and Windows follows. Platform-specific code must be isolated, and the other targets must keep compiling.
- **Small dependency surface.** Reuse the standard library and existing crates where practical. A new dependency needs a concrete benefit worth its build time, binary size, maintenance and security cost.
- **Visible failure, private data.** Errors should be actionable, rate limits respected, and tokens and customer data must never appear in logs, fixtures or screenshots. Network behaviour belongs in the documentation.
## Settled decisions
 
These are decided. Issues and pull requests that reopen them are closed with a link here. Add to this list as decisions are made.
 
**No Shopify scraping.** The app talks to the Admin API only. A feature that needs the admin's HTML, a merchant's browser session or an undocumented endpoint is out of scope.
 
**Own app credentials.** The app never signs in with another project's Shopify app credentials. A client ID belongs to whoever registered it, and its quota and review status to their users. A pull request that adds someone else's client ID, secret or redirect path is closed.
 
**Tokens in the keychain only.** Access tokens are stored only in the operating system's keychain. Storing them in files, the cache or logs is not accepted.
 
**Windowing fixes go upstream.** Fixes to how the operating system treats the window go to the windowing library upstream first, and the maintainer carries them. The app does not patch native window code itself.
 
**To decide after the spike:** the UI framework fallback (Tauri or Electron), how the connector and desktop app pair, and the appearance and theming scope. Until then, do not propose changes in these areas.
 
## Pull requests
 
Keep each pull request to one change. Unrelated fixes, even small ones, go in separate pull requests. Explain why it belongs in the project, what changed and how you tested it. Avoid unrelated formatting, refactors, generated prose and large mechanical rewrites.
 
`main` has a linear history. Outside pull requests are squash-merged into one focused commit with contributor credit, and merge commits are not accepted. Maintainer work is committed on main one topic per commit, or arrives from a worktree branch by squash or fast-forward. Use fast-forward-only pulls and rebase unpublished local commits when needed. Do not rewrite published history without explicit maintainer approval.
 
The same rules apply to hand-written and AI-assisted changes. The author must understand every line and answer review comments with specific reasoning.
 
Code changes should include tests for behaviour that can regress. UI changes should include before and after screenshots or a short recording in the pull request, not in the repository, and should use demo mode where possible. User-visible behaviour, settings, files and network access must be documented in the same pull request.
 
Changes that mutate store data (the mutation queue, bulk edit, undo log) and changes to auth, tokens, the keychain or OAuth need extra scrutiny and explicit maintainer review, whatever CI says.
 
Dependencies come from crates.io or from repositories the maintainer owns. A git dependency or `[patch]` entry may point only at such a repository, and adding or changing one needs explicit maintainer approval. Never point a dependency at your own fork or any other source. If a dependency needs a fix, send it upstream and link it from the pull request, and the maintainer can carry it in a repository they control until a release includes it.
 
## Settings
 
Do not add a setting (a new field in settings, a toggle, a slider, a choice of options) without the maintainer's approval in the issue first. The answer is usually no: prefer a good default that serves everyone, or a keyboard shortcut or command for people who want something else. A pull request that adds a setting without that approval will be asked to remove it.
 
## Test data and safety
 
Tests and agents use fixtures, the mock layer or a development store, never a live store. Fixtures are recorded from a development store and scrubbed of tokens and customer data before committing. Tests that call a real development store are marked `#[ignore]` and run on demand.
 
## Visual reviews
 
Review evidence lives in the pull request, never in the repository. Attach captures and recordings to the pull request description, or to a comment when you update them. Do not commit screenshots, comparison pages or recordings. Only images the project itself uses, such as a documentation figure or an icon the app draws, belong in the repository.
 
For any interface change:
 
- Use actual before and after captures, from demo mode where possible. Match the data, page, interaction state, zoom and window size, so the difference shows the proposed change.
- Lay them out as a table with Before and After side by side, for light and dark themes at a narrow and a normal window size.
- Show relevant open menus, loading, error and empty states, and attach a short recording when the change moves.
- Say which platforms actually produced the captures.
Moving navigation or controls, regrouping menus, changing the window shell, panel ownership or sizing, spacing or visual hierarchy is a redesign, even when behaviour still works. A redesign needs the maintainer's explicit approval of the visual scope before merging. Approval covers the described appearance and interaction, survives rebases that keep that scope, and an explicitly requested adjustment can be made and re-evidenced without asking again.
 
## Checks
 
Run the checks CI runs before submitting:
 
```
mise run check
```
 
This runs formatting, clippy with warnings denied and the test suite across the workspace. Expanded, it is:
 
```
cargo fmt --all --check
cargo clippy --workspace --locked --all-targets -- -D warnings
cargo nextest run --workspace --locked
cargo test --workspace --locked --doc
RUSTDOCFLAGS='-D warnings' cargo doc --workspace --locked --no-deps
```
 
As the project grows, add CI scripts that fail the build on the same policy violations this file describes: git dependencies from unapproved sources, review captures committed to the repository, and new settings added without the approval label. Document each script here when it lands.
 
Linux needs the system packages listed in the README under build from source. CI repeats the test suite on Linux and macOS. Passing CI is required, but does not replace review for correctness, product fit, maintainability or security.
 
Credential-storage changes also need a native keychain round trip against the real store, using temporary dummy values that are deleted afterward. The ordinary test suite uses an isolated fake store and never reads a real token.
 
Changes to `Cargo.lock` or the pinned Admin API version also need the generated GraphQL types regenerated and committed, and the tests re-run against the recorded fixtures. Releases must wait for all required CI jobs on the version commit before the tag is pushed.
 
## Licence
 
By contributing, you agree that your contribution is licensed under the project's licence (to be set by the maintainer).
 
Claude finished the response
Closed tab CLAUDE.md
