# stv-io/zentty: a patched build of Zentty

This fork adds one feature to Zentty and keeps it on top of each upstream release until the feature is merged:

- **New Named Worklane in This Window…** in the Move Pane to Worklane menu ([issue #182](https://github.com/dedene/zentty/issues/182), upstream PR [#183](https://github.com/dedene/zentty/pull/183)).

The build installs as **`/Applications/Zentty Dev.app`**. That's upstream's own dev-build mechanism (`scripts/build-dev-release`), so it runs alongside the official `Zentty.app` without conflicts:

- **Separate app:** it has its own bundle id (`be.zenjoy.zentty.dev`) and an orange DEV badge on its icon.
- **Settings:** in `~/.config/zentty-dev`, copied from `~/.config/zentty` the first time and kept separately after that.
- **No auto-update:** it never updates itself. Updating it is what this document is for.

## Branches and remotes

| Name | What it is |
|---|---|
| `upstream` remote | `dedene/zentty`. Fetch only; push is disabled. |
| `origin` remote | `stv-io/zentty` (this fork). |
| `stv/release` | **What you run:** the latest upstream release tag, the feature commit, and this doc and script. |
| `feat/named-new-worklane` | **The PR branch:** based on upstream `main`, containing only the feature commit. |

## Day to day

```bash
cd ~/code/personal/zentty
scripts/fork-sync --check   # is there a new upstream release? has the PR merged?
scripts/fork-sync           # rebase onto the latest release, push, rebuild, reinstall Zentty Dev
```

Run it from the official Zentty or another terminal, **not from a Zentty Dev pane**. Installing quits Zentty Dev.

`fork-sync` does the following:

1. Fetches `upstream` with tags and looks up the latest release (`gh release view --repo dedene/zentty`).
2. Rebases `stv/release` onto that tag. Only the fork's commits are replayed, and any commit already upstream is dropped.
3. Pushes `stv/release` to the fork with `--force-with-lease`.
4. Rebuilds GhosttyKit if `scripts/ghosttykit.lock` changed, using the Zig version that file names (installed with mise).
5. Runs `scripts/build-dev-release --adhoc`, which builds Release and installs `Zentty Dev.app`.

The build is signed ad hoc (we don't have Zenjoy's Developer ID), so macOS asks again for privacy permissions after each rebuild.

Upstream's `--adhoc` mode leaves the hardened runtime on. Its library validation rejects the ad hoc signed Sparkle and Sentry frameworks, so the app crashes at launch with "Library not loaded … different Team IDs". The fork's `stv/release` fixes this with a separate commit that sets `ENABLE_HARDENED_RUNTIME=NO` for ad hoc builds. It's also proposed upstream as [PR #184](https://github.com/dedene/zentty/pull/184). Keep that commit when rebasing until #184 is in a release; after that, the rebase applies it as a no-op.

Options: `--tag vX.Y.Z` targets a specific release, `--no-build` only rebases and pushes, and `--no-push` keeps the branch local.

## If the rebase conflicts

The feature touches these files:

- `MoveToWorklaneMenuBuilder.swift`
- `MainWindowController.swift`
- `AppDelegate.swift`
- `RootViewController.swift` (the `Name New Worklane` extension at the end)
- `WorklaneStore+DragDrop.swift` (`transferPaneToNewWorklane` gained `title:`)
- Two test files.

To resolve a conflict:

1. Fix the files, then run `git add` and `git rebase --continue`.
2. Run the two test suites:
   ```bash
   TEST_RUNNER_SWIFT_BACKTRACE=enable=no xcodebuild test -scheme Zentty -destination 'platform=macOS' \
     -only-testing:ZenttyLogicTests/MoveToWorklaneMenuBuilderTests \
     -only-testing:ZenttyLogicTests/PaneStripStoreTests \
     CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO
   ```
3. Rerun `scripts/fork-sync`.

If upstream changed the same code, also rebase `feat/named-new-worklane` onto `upstream/main` so the PR stays mergeable, then push it with `git push --force-with-lease origin feat/named-new-worklane`.

## Build prerequisites (one-time, on a new Mac)

- **Xcode,** plus its Metal Toolchain: `xcodebuild -downloadComponent MetalToolchain`.
- **mise,** for Zig. Devbox's `zig` is the wrong version, and the script uses `mise exec zig@<lock version>`.
- **`gh`,** logged in.
- **Remotes:** `git remote add upstream https://github.com/dedene/zentty.git && git remote set-url --push upstream DISABLED`.

## When the PR is merged

Once an upstream **release** includes the feature:

1. Quit Zentty Dev and go back to `/Applications/Zentty.app`, which updates itself through Sparkle.
2. If you want to bring over settings you changed in the dev build, copy them from `~/.config/zentty-dev`.
3. Delete `/Applications/Zentty Dev.app`, `~/.config/zentty-dev`, and `~/Library/Application Support/Zentty Dev`.
4. Optionally archive or delete the fork.

`fork-sync` checks PR #183 on every run and tells you once it is merged (override the number with `ZENTTY_FORK_PR`).
