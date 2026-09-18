# Intel and Apple Silicon Support Plan

## Goal

Support both `arm64` and `x86_64` Macs running macOS 14 or later without
regressing the current Apple Silicon build. Fan Control remains an
Apple-Silicon-only feature because it requires its protected helper; the rest
of the application, including Dynamic Island, should be available on both
architectures.

Start by producing separate architecture-specific builds. A universal app is
not part of this work and can be considered only after both independent builds
are proven.

## Guardrails

- Keep the port in a dedicated branch, for example `feature/intel-support`.
- Make narrowly scoped commits. Do not combine this work with formatting,
  refactors, or feature changes.
- Preserve the current Apple Silicon build and tests as a control throughout.
- Reuse the already-proven Intel compatibility behavior in the sibling
  `vorssaint` project where it still applies, but merge it manually so newer
  changes in this project are retained.
- Do not claim Intel support or publish a combined artifact until native Intel
  builds and tests pass.

## Phase 1: Establish the Apple Silicon baseline

Before changing the code, run and record the results of:

```sh
./build.sh
./build/Vorssaint --selftest
./build.sh --test
```

This gives a known-good reference for later regressions. Keep the output with
the implementation notes or pull request.

## Phase 2: Restore architecture-selectable builds

**Commit: `build: restore arm64 and x86_64 target selection`**

Update `build.sh` to:

1. Restore an `ARCH` variable that defaults to `uname -m`.
2. Accept only `arm64` and `x86_64`, with a clear error for other values.
3. Derive `TARGET` as `$ARCH-apple-macosx14.0`.
4. Retain the current test-runner improvements and all newer Now Playing
   adapter sources.

At this point, the initial target is compilation only:

```sh
ARCH=x86_64 ./build.sh
```

Do not add universal-binary logic in this phase.

## Phase 3: Restore the Fan Control architecture boundary

**Commit: `fix: disable fan control safely on Intel`**

Restore the small architecture capability gate that exists in the
Intel-capable sibling project:

1. Add `FanControlArchitectureSupport` to
   `Sources/Vorssaint/Services/FanControl/FanControlSupport.swift`.
2. Return `true` only under `#if arch(arm64)` and `false` on Intel.
3. Use the gate in `FanControlService` so the service does not start or attempt
   recovery on Intel.
4. Use the gate in the feature availability path in `FeatureRuntime` so the UI
   labels Fan Control as unsupported on Intel.

Preserve newer Fan Control logic already in this branch, including the current
fan unlock and retry behavior. This step only restores the compatibility
boundary; it must not roll the Fan Control implementation back.

Expected Intel behavior:

- Fan Control is visible as unavailable rather than silently failing.
- No XPC connection, helper registration, helper removal, or recovery is
  attempted.
- The application otherwise starts normally.

## Phase 4: Make helper compilation and packaging conditional

**Commit: `build: omit fan helper from Intel bundles`**

In `build.sh`, condition all Fan Control helper work on `ARCH=arm64`:

1. Compile and run the helper self-test only on Apple Silicon.
2. Copy the helper executable only on Apple Silicon.
3. Copy and mutate the Fan Control LaunchDaemon plist only on Apple Silicon.
4. Write `VorssaintFanControlHelperVersion` only when the helper is present.
5. Leave main-app and Now Playing adapter compilation enabled for both targets.

Validate the resulting Intel app bundle:

```sh
ARCH=x86_64 ./build.sh
file build/Vorssaint
file build/Vorssaint.app/Contents/Frameworks/libVorssaintNowPlaying.dylib
```

Confirm that the executable and adapter are `x86_64`, while the Intel bundle
does not contain the Fan Control helper or its LaunchDaemon plist.

## Phase 5: Run the Intel test and self-test suite

**Commit: only if a test exposes an actual compatibility defect**

Run:

```sh
ARCH=x86_64 ./build.sh --test
ARCH=x86_64 ./build/Vorssaint --selftest
```

Address failures one subsystem at a time. The initial review found no
architecture-only restriction in Dynamic Island, the Now Playing adapter, or
the checked-in dependencies, so avoid speculative rewrites.

Prioritize failures in this order:

1. App startup and preference migration.
2. System Monitor and IOKit sensor fallbacks.
3. Display brightness and external-display handling.
4. Screen capture and recording.
5. Dynamic Island / non-notched-display layout.
6. App Updates architecture matching and Now Playing adapter loading.

Each genuine fix should have its own focused commit and a regression test when
the code can be tested without hardware.

## Phase 6: Native Intel hardware validation

Use at least one native Intel Mac; cross-compilation alone is insufficient.

Verify:

- Launch, settings, login item, permissions, update/install, and uninstall.
- Dynamic Island on built-in and external displays, including a display without
  a camera cutout.
- Music, timers, downloads, notifications, calendar, files, clipboard, and
  capture controls in Dynamic Island.
- Screenshot and recording behavior with the Island shown and hidden.
- System Monitor and brightness behavior when hardware readings are absent.
- Fan Control stays unavailable and does not leave a helper or daemon behind.

Keep a concise pass/fail checklist with Mac model, macOS version, and build
architecture. Fix only observed differences from the Apple Silicon baseline.

## Phase 7: Add native Intel CI

**Commit: `ci: verify Intel build and bundle contents`**

Add a separate native Intel macOS job. It must run on an Intel runner rather
than only cross-compiling from Apple Silicon.

The new job should run:

```sh
ARCH=x86_64 ./build.sh
ARCH=x86_64 ./build/Vorssaint --selftest
ARCH=x86_64 ./build.sh --test
```

It should also assert that:

- The app executable is `x86_64`.
- `libVorssaintNowPlaying.dylib` is `x86_64`.
- The Fan Control helper is absent.
- The Fan Control LaunchDaemon plist is absent.

Keep the existing Apple Silicon CI job unchanged. Both jobs must pass before
merging future changes that touch shared code or the build pipeline.

## Phase 8: Update support and release messaging

**Commit: `docs: document Intel support and fan control limitation`**

After native Intel validation and CI are green:

1. Change the README requirement badge and hardware requirements to say Intel
   and Apple Silicon.
2. State clearly that Fan Control is Apple-Silicon-only.
3. Add release notes describing the restored Intel build.
4. Name release files unambiguously, for example:
   - `Vorssaint-<version>-arm64.dmg`
   - `Vorssaint-<version>-x86_64.dmg`

## Phase 9: Beta release and stabilization

Release Intel support first as a beta. Collect feedback from Intel laptops,
Intel desktops, and external-display users. Promote it to stable only after:

- both architecture CI jobs pass;
- native Intel validation is complete;
- no helper or daemon is installed on Intel;
- Apple Silicon regression checks remain green.

## Deferred: Universal app bundle

Do not create a universal app as part of this port. The app executable and Now
Playing dylib could eventually contain both slices, but the Fan Control helper
is intentionally Apple-Silicon-only. A universal bundle therefore requires a
separate packaging, signing, update, and installation design review. Treat it
as a follow-up project after separate artifacts are stable.
