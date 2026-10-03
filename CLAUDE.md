# CLAUDE.md — pomvox/pomvox-cleanup-mlx

Every session, human or agent, starts here. This file is condensed from the
owner's vault (`pomvox/pomvox_obsidian_vault`, section `70 Cleanup Engine/`);
the vault wins over this file, and this file wins over an issue's text.

## What this repo is

Pomvox Cleanup MLX is the Apple Silicon runtime for the Pomvox Cleanup Engine,
published as the `PomvoxCleanupMLX` Swift package at exact version `VERSION`
(0.1.0-beta.2) so apps can consume it remotely with no submodule. It pulls the
core SDK (`pomvox-cleanup-engine`) at the matching exact version and pins its
MLX dependencies.

**What it is not: a place to make changes.** This repository is a generated
release distribution of `Runtime/MLX` in `pomvox/pomvox-cleanup-engine`, written
by that repo's `scripts/export-mlx-release.py`; `SOURCE.json` records the source
tag and the SHA-256 of every exported file. Engine-first rule
(pomvox/pomvox#173): cleanup behaviour changes land in the engine repo, get a
tag, are exported here, and reach the app as a pin bump titled
`chore(cleanup): engine vX.Y.Z`. Nothing is hand-edited here, and nothing here
is a production-readiness certification (it is a beta developer SDK).

## Build and test

macOS 14+, Apple Silicon and full Xcode with Swift 6. Mac-only: in a cloud
session do not attempt it. Plain `swift build` does not package the Metal
shader library; build through Xcode. This is what CI (`.github/workflows/ci.yml`)
runs on `macos-15`:

```sh
brew install xcodegen
xcodegen generate --spec Examples/Consumer/project.yml
xcodebuild -project Examples/Consumer/CleanupConsumer.xcodeproj \
  -scheme Consumer -configuration Debug -derivedDataPath "$RUNNER_TEMP/consumer" \
  -destination 'platform=macOS,arch=arm64' build-for-testing
```

CI consumes this package remotely at the exact pushed commit, then checks the
resolved pins: core SDK version equals `VERSION`, `swift-tokenizers` is `0.5.0`,
and every pin is remote source control. On a tag it asserts the tag equals
`v$(cat VERSION)`. CI does not download gated weights; real-model validation and
the remaining release gates live in the source repository (`docs/testing.md`).

## Invariants that block a PR

The SDK contract, from the source repo's `CONTRIBUTING.md` and the vault's
`Review Philosophy`:

1. **Never lose words.** A fallback returns the exact input bytes and no edits.
   A canceled result is never inserted: cancellation throws.
2. **Never block the latency path.** Deadlines bound admitted work; keep one
   cleaner open across dictations rather than opening per request.
3. **Never break local-first.** The runtime never downloads weights or routes to
   cloud; model access is the host's explicit job.
4. **One generation owns the caches.** At most one local generation owns mutable
   caches, and resources stay alive until it returns; a hung worker cannot be
   forcibly interrupted.
5. **Parity with the source.** Exported file hashes must match `SOURCE.json`;
   supported behaviour is English, bounded vocabulary and the frozen prompt.
   Style controls, auxiliary generation and unlimited output are not supported
   (the app's pomvox/pomvox#173 is blocked on engine#2 and engine#3 for those).

## Areas not to touch

| Area | Status | Why |
|---|---|---|
| Everything under `Sources/`, `Tests/`, `Examples/`, `Package.swift`, `README.md` | generated | Edit `Runtime/MLX` in `pomvox-cleanup-engine` and re-export; a hand edit here breaks `SOURCE.json` and is overwritten by the next export. |
| `SOURCE.json`, `VERSION` | written by the exporter | The tag check and the pin check in CI depend on them. |
| The frozen prompt and the pinned pack (`pack.json` in the source repo) | frozen | The runtime verifies all seven pinned artifacts; do not replace the snapshot with a newer revision. |
| The app's `CleanupEngine.swift` (pomvox/pomvox) | frozen, pomvox/pomvox#173 | In-app cleanup bugs are filed on the engine. The app's `vendor/` submodule is replaced by this package under pomvox/pomvox#172. |

## Where the lessons are

Search the vault's `60 Lessons/60 Lessons MOC.md` by symptom. Closest to this
repo: the cleanup-speed lessons (the pass-cost curve is the design; the guard
that ate the fix), The Impossible Optimization, HF Snapshot Globs and the
Adapter Trap, and meta-lessons 8 (if a wrong answer would be silent, the test
must be differential) and 16 (a guard is a belief about a model).

## PR checklist (vault: `Reviewing a Pomvox PR.md` §5)

A PR here is normally an export commit from the source repo at a tag.

- CHANGELOG entry for user-visible changes (in the source repo); version bump if
  this cuts a release.
- Docs that the diff invalidates are fixed in the diff.
- Commit style: conventional commits, 72-char subject at most, GPG-signed.
- Follow-ups discovered during review get filed as issues in the PR thread, not
  silently remembered.

Process rules that go with it: one concern per PR; PRs only, never merge, never
`--delete-branch`; request review from Abhi; at most two open PRs per repo; no
fabricated numbers (a figure comes from a command run in this session on stated
hardware, or from the CHANGELOG, with the source named, else "not measured
here"); closing an issue needs Abhi's go.
