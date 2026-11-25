# AGENTS.md
 
Agent guide for the Shopify desktop app: a small native Rust app that lets a merchant run day-to-day store operations (products, collections, channels, catalogs and price lists, menus, reports) without opening the browser. Linux and macOS first, Windows later.
 
This file is the contract for every coding agent in this repo. Read it fully before changing code. If a task conflicts with it, stop and ask the maintainer. Product and contribution policy lives in CONTRIBUTING.md; these instructions add implementation constraints for agents.
 
## Status and open decisions
 
- Phase: feasibility spike. Goal: prove egui handles a 50k-row editable table, a product description editor and one report chart, with acceptable startup time and memory on Linux and macOS.
- The fallback framework (Tauri or Electron) is chosen after the spike, only if the editor or tables fail. Do not start a migration on your own.
- Shopify Admin API version: PINNED_VERSION (fill in). Never change it without being asked.
- Do not decide these without asking: how the connector and the desktop app pair, whether reports are available on the target plans and scopes, and whether the connector is public or unlisted.
## Product boundaries
 
- Keep this a small native Shopify client. Do not add a browser engine, telemetry, analytics, or a hosted backend that stores merchant data.
- Only use Shopify's documented Admin API. Do not scrape the admin, reuse browser sessions, or use undocumented endpoints. Do not advertise a capability because a field or enum exists in the schema; confirm the operation works for the merchant's plan and scopes.
- Do not broaden a task into adjacent features or a general refactor. Preserve existing user behaviour unless the task explicitly changes it.
- Do not add a setting without the maintainer's approval in the issue first. Prefer a good default, a shortcut or a command.
- The connector stays small. No logic that the desktop app needs lives there.
## Repository layout
 
```
Cargo.toml                 workspace root
mise.toml                  pinned toolchain and tasks
crates/
  shopify-api/             GraphQL client, rate limits, generated types
    graphql/               .graphql queries and mutations
    schema/                pinned Admin schema file
  store-core/              domain models, SQLite cache, sync engine,
                           mutation queue, preview/diff, undo log
  app-ui/                  egui widgets and views (no network code)
  desktop/                 binary: wiring, auth, keychain, async runtime
connector/                 Shopify-facing app (not a Cargo member)
fixtures/                  recorded Shopify responses, scrubbed
docs/
  specs/                   one spec per task
  _reference/              contracts agents must read (see below)
  decisions/               short decision records
xtask/                     optional repo scripts (codegen, fixtures)
```
 
Dependency direction is enforced by the compiler:
 
- `app-ui` depends on `store-core` only. It cannot import `shopify-api`.
- `store-core` depends on `shopify-api`. Only the sync engine and mutation queue call it.
- `desktop` depends on all three and is the only place for auth and keychain access.
- `shopify-api` depends on no internal crate.
Generated GraphQL types live in `crates/shopify-api/src/generated/` and are never edited by hand.
 
## Architecture
 
- `app-ui` draws views and emits actions. Apply actions after drawing, in the app layer. Never mutate application state from inside a borrowed view.
- Network and sync work belongs on the async runtime in `desktop` and `store-core`, never as blocking work on the UI thread. File and database writes that wait on disk also stay off the UI thread: encode on the UI side, hand the bytes to a writer task.
- The cache is the source of the UI. Sync is incremental using `updated_at` cursors. A desktop app has no public endpoint, so there are no webhooks. Large exports and imports use bulk operations.
- Queries and mutations live in `.graphql` files and compile to typed Rust through codegen against the pinned schema. No hand-built query strings.
- Rate limits are handled in one place, the Shopify client: read the cost and throttle status from each response and back off. Never retry in a tight loop.
- Keep platform integrations behind target-specific modules or `cfg` blocks. A fix for one platform must keep the other targets compiling.
- Settings and state files must stay readable, backward compatible and atomically written. Never log credentials, tokens or authorization responses.
- Prefer existing dependencies. Explain any new crate in `Cargo.toml` next to the dependency when the reason is not obvious, and ask before adding one.
- Products is the template module (schema, sync, list, detail, edit, bulk edit). New modules copy its structure.
Read before changing the matching area:
 
- `docs/_reference/how-it-connects.md`: authentication, scopes, token storage, Shopify requests and network behaviour.
- `docs/_reference/sync.md`: sync cursors, bulk operations and conflict rules. Its rules are the contract, and the sync tests enforce them.
- `docs/_reference/what-shopify-allows.md`: what each API and plan offers, and the requests none of them can serve, each with its reason. Before building or promising a Shopify-facing feature, find it there. A request in the unsupported section is answered with that reason, not worked on. If the reason lapses because the API gained the capability, update the page in the same change.
(Create these reference pages as the area is built. If one is missing for the area you are touching, write it as part of the task.)
 
## Interface behaviour
 
- The interface is optimistic. A control shows its result the moment it is used: an edited price shows the new value, a toggled status shows the new status. The backend then makes it true and Shopify's state catches up behind. If a later answer still tells the story from before the user's action, hold the shown state and ask again instead of letting the lagging answer undo it. Nothing the user did may flicker away and come back.
- The optimistic rule never applies to destructive or bulk actions. Those go through the preview described below first.
- Show progress and errors for every async operation, in plain language, with a next step.
- Text is always passed to the text layer in logical order. Do not reorder strings before layout, so right-to-left languages stay correct.
## Mutation safety (non-negotiable)
 
- No mutation reaches Shopify without a preview: a per-item diff of what changes, shown to the user and confirmed.
- Every bulk edit writes an undo log with previous values before it runs, and can be reverted from the app.
- Every mutating command supports dry-run, which runs the full pipeline except the final Shopify call.
- Destructive actions (delete, unpublish, status changes on many items) require an extra confirmation that names the count.
- Mutations run through one queue with retry and conflict handling. If the remote item changed since the cached version, stop and show the conflict. Never overwrite silently.
- Automated code and agents use the development store or fixtures only. Never point automated code at a live store.
## Secrets and auth
 
- Tokens are stored only in the OS keychain. Never in files, SQLite, logs, fixtures, screenshots or error messages.
- OAuth uses a localhost redirect. Request read scopes first and add write scopes only when a module needs them. Confirm each scope against the docs for the pinned version.
- Never print or commit client secrets. Secrets come from environment variables or the keychain. `.env` is git-ignored, and only `.env.example` is committed.
- Scrub tokens and customer data from fixtures before committing.
## Conventions
 
- Rust stable, pinned in `mise.toml`. `cargo fmt` and `cargo clippy -- -D warnings` must pass.
- Typed errors with `thiserror` in libraries, `anyhow` only at the binary edge. No `unwrap()` or `expect()` outside tests, except for provably impossible cases with a comment.
- Logging with `tracing`, with no personal data or tokens.
- User-facing text goes through one place so it can be localized later.
- Match the existing style. Do not introduce new patterns in a feature change.
## Disk use and builds
 
- Run `mise trust` once in each new checkout or worktree, or mise refuses to run tools there.
- Give each worktree and each parallel agent its own `target/` directory. A worktree's own `target/` is enough. A second build in the same checkout may use `CARGO_TARGET_DIR=target/<name>`. Never point builds at a shared target directory: Cargo's lock serializes them, and one worktree's test run can execute another's binary.
- Never vary `codegen-units` or other compiler flags per agent. Each variant is a separate cache entry and fills the disk.
- Do not `cargo clean` to save space.
- Never put build output or large scratch files in `/tmp`. Use `.cache/` inside the checkout.
- Delete one-off QA, packaging and release-validation directories once their result is recorded.
## Definition of done
 
A task is done only when all of this holds:
 
- `mise run check` passes (fmt, clippy with warnings denied, and the test suite).
- Changed behaviour has focused regression tests. Use fixtures or the mock layer for Shopify, and `egui_kittest` snapshots for UI.
- No test depends on the network or a real store, except tests marked `#[ignore]` that run against the development store.
- README and docs are updated when user-visible behaviour, settings, files or network access change.
- Do not weaken a lint, delete a test, or add an `allow` just to make CI pass. If the rule does not apply, explain why in the change.
- Report platform coverage honestly. Do not claim a platform was tested when it was only compiled or reasoned about.
- For performance work, report startup time, memory and frame time for the 50k-row table on Linux and macOS, with the command used.
## Interface review
 
- Distinguish an internal UI refactor from an interface redesign. Moving navigation or controls, regrouping menus, changing the window shell, panel ownership or sizing, spacing or visual hierarchy is a redesign, even when behaviour still works.
- Call out every user-visible interface change at the top of the change description. Green CI does not make a redesign merge-ready.
- A redesign needs explicit maintainer approval of the visual scope before merging.
- Provide before and after evidence at representative window sizes and in light and dark themes. Keep that evidence in the pull request description or comments. Never commit captures, comparison pages or recordings to the repository.
## Branches and pull requests
 
- Each agent works in its own git worktree and branch, one task per branch, in small changes. One topic per commit, each compiling and passing the checks on its own.
- Keep main linear. Squash agent branches into one focused commit. Never create or push merge commits. Use fast-forward-only pulls and rebase unpublished local commits.
- Rewriting published history requires explicit maintainer approval.
- Dependencies come from crates.io, or from repositories the maintainer controls. A git dependency, patch or override may point only at such a repository, and adding one needs explicit approval. Never point a dependency at a contributor's fork or any other source. When a dependency needs a fix, send it upstream and ask the maintainer to carry it in a fork they control.
- Review evidence (screenshots, captures, recordings) stays out of the repository.
## Human review required
 
A person reviews and approves, regardless of CI results:
 
- Anything touching auth, tokens, the keychain or OAuth.
- Anything that mutates store data: mutation queue, bulk edit, undo log.
- Release, signing, notarization, packaging and auto-update.
- Dependency additions and API version changes.
## Issue communication
 
- Write replies for the reporter, short and direct, in plain language. Not an investigation log.
- A reply moves the issue forward: a decision, a plan, or one specific thing needed next. Include technical detail only when the reporter needs it to act.
- When a valid issue has a clear, bounded fix that can be done now, do it instead of posting the proposed design.
- Close a bug once its fix is on main and the checks pass. State which commit fixes it and whether it is released.
- Never post two maintainer comments in a row on the same issue. Edit the previous one if nobody has replied.
- Keep private investigation notes out of public threads.
- Never use em dashes in public text. Use a full stop, comma, colon or parentheses.
## Releases
 
A release is not the tag alone. Do these in order, and report any skipped step:
 
1. Bump the version, update the lockfile with a build, and add written release notes in `packaging/release-notes/vVERSION.md`. Wait for every required CI job on the release commit, on both Linux and macOS, before tagging.
2. Push the `v*` tag, which triggers the release workflow. Wait for every artifact and `checksums.txt`, then verify the published notes and download links. Never publish generated placeholder notes.
3. macOS builds must be signed and notarized before publishing. Linux packages (AppImage, deb) must be installed and launched on a clean machine before publishing.
4. After the release exists, update the website download page and package channels (Homebrew cask, Linux packages) from `checksums.txt`. Never point a download page at files that do not exist yet.
5. Before writing release notes, read the previous two releases and match their style: a short plain-language summary, New and Fixed sections, bold user-facing results, issue numbers and a link to the full changelog.
Skipping an applicable step ships a release that lies somewhere.
 
Claude finished the response
