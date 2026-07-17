# Porting GraphSketcher to macOS 27

This fork gets the Mac app building and running on macOS 27 / Xcode 27, natively on Apple Silicon. Upstream targets OS X 10.8 and Xcode 5.1 — a gap of roughly thirteen years.

The app builds, launches, and opens its document and inspector windows. The iPad app has not been ported.

## The short version

Most of this turned out to be **environmental, not rot**. The project builds with `-Werror`, so every "deprecated in macOS 10.10" warning was fatal: roughly 300 errors collapsed to 4 once warnings-as-errors was turned off. Nothing in the app is fundamentally 32-bit-bound, and the code is arm64-clean.

Most of the real fixes already existed upstream. The OmniGroup submodule is pinned to a 2014 commit, but the repository itself was maintained until 2022, and Omni had already fixed several of these exact problems. Checking `origin/HEAD` before writing any code was the single highest-leverage move here.

## Build settings

None of these are baked into the `.xcconfig` files yet, so they must be passed on the command line (see the README). Folding them into the xcconfigs is the obvious next task — it's what would make the Xcode UI's Build button work.

| Setting | Why |
| --- | --- |
| `MACOSX_DEPLOYMENT_TARGET=12.0` | The projects request 10.9; Xcode 27 supports 12.0 and up. |
| `ARCHS=arm64` etc. | The 2014 config hardcodes `x86_64`. On an Apple Silicon Mac without Rosetta, build tools compiled that way fail at *build* time with `bad CPU type in executable`. |
| `GCC_TREAT_WARNINGS_AS_ERRORS=NO` | See above — this is what hides the real errors behind hundreds of deprecation warnings. |
| `-Wno-macro-redefined` | The project defines `OBJC_OLD_DISPATCH_PROTOTYPES=0`; modern Xcode injects `=1`. |
| `CODE_SIGN_IDENTITY="-"` | Ad-hoc signing. The app is sandboxed, and a sandboxed app must be signed to run. |

## The crash worth knowing about

A clean build still segfaulted on launch, inside LaunchServices (`_LSCopyLocalDatabase`), reached from `[NSImage size]` while the inspector panel restored.

The cause is `OBPerformRuntimeChecks()` in OmniBase. It walks every class in the runtime (~17,000 on macOS 27) introspecting method signatures. Afterwards, loading an image by URL crashes inside LaunchServices. Before the sweep, the identical call works — this reproduces in a dozen lines of code with no GraphSketcher involved.

This is not a novel problem, just a worse one. That function already carries a skip-list of class prefixes that "asplode" when introspected — `NS`, `_NS`, `__CF`, `CA`, `VGL`, and a one-off for `DRDevice` — and Omni already disables the whole sweep when running inside `ibtool`. macOS 27 simply has classes the 2014 list never knew about.

It is a **debug-only diagnostic** (`#ifdef OMNI_ASSERTIONS_ON`), so it is now opt-in rather than deleted:

    OBPerformRuntimeChecks=1 ./GraphSketcher.app/Contents/MacOS/GraphSketcher

Expect it to crash if you turn it on.

## Fixes in this repo

- **`GraphSketcher/Model/RSNumber.h`** — removed a dead `NSAppKitVersionNumber10_4` shim. It was guarded by `#ifndef`, but the modern SDK declares that name as a `static const` *variable*, not a macro. The guard didn't fire, so the macro rewrote Apple's own header into `static const NSAppKitVersion 824 = 824;`. It was unused.

- **XIBs** — six files targeted macOS 10.5, below the 10.6 minimum ibtool now accepts. Four were in the 2009-era "archive" format and were converted with `ibtool --upgrade`; outlet counts were checked before and after.

## Fixes in the OmniGroup submodule

These live in `patches/0001-omnigroup-macos27-port.patch` (see the README for how to apply it, and why it's a patch rather than commits).

Backported from upstream OmniGroup:

- **ColorSync** — upstream replaced the dead Carbon ColorSync APIs with `ColorSyncProfileRef`. Taken wholesale; `OAColorSpace.{h,m}` became `OAColorSpaceManager.{h,m}`.
- **`OAFeatures.h`** — `OA_USE_COLOR_MANAGER` is 0 on 10.11+, where the Carbon externs are gone entirely.
- **`OFMessageQueue`** — the 2014 file used `__weak` (ARC-only) *and* manual `retain`/`release` (MRC-only) in the same file. In 2014 clang tolerated that; it now rejects it, and no compiler flag resolves the contradiction. Upstream had already converted the file to ARC.
- **`OFAlias`** — upstream replaced the Carbon `require`/`require_noerr` macros with plain `if`/`goto`.
- **`NSObject-OFExtensions`** — dropped the `-bundle` instance method. A `-bundle` on `NSObject` applies to every object in the process, including Apple's private `IFBundle`, whose `-bundle` returns `CFBundleRef` rather than `NSBundle *`. Upstream kept `+bundle` and deleted the instance method; callers use `[[self class] bundle]`.

Local, where upstream had diverged too far to backport:

- **`OBRuntimeCheck.m`** — the runtime sweep, above.
- **`OQColor.h`** — imports CoreGraphics explicitly. It relied on the types arriving transitively via `NSGeometry.h`, which they no longer do. The file's own iOS branch already imported them; the Mac branch never had to.
- **`OIInspectorRegistry.m`** — `[NSApp delegate]` is now typed `id<NSApplicationDelegate>`, and clang won't dispatch informal-protocol selectors on it. Cast to `id`, restoring the 2014 dynamic dispatch.
- **`OBUtilities.h`** — added an `OBFinishPortingWithNote` shim so backported sources compile against this vintage of OmniBase.
- **`OAContextButton` / `OAContextPopUpButton`** — `__weak` ivars in MRC files became plain assign, and their `weak` properties became `assign`.
- **`NSColor-ColorSyncExtensions.m`** — upstream's `OBPerformPosing(^{...})` block converted back to `+performPosing`; this OmniBase predates the `OBLoadAction` system.

## Known caveats

- **`FixStringsFile`'s self-test is disabled.** This is a real, pre-existing bug, not something the port introduced: modern `genstrings` emits UTF-16 with a BOM, and the tool's parser predates that, so it silently passes text through without converting it (`Ellipsis...` never becomes `Ellipsis…`). No app target uses the tool, so it doesn't affect GraphSketcher. Re-enable the script phase in its pbxproj once the parser handles a BOM.

- **The runtime checks are off** (see above). They were the only reason the app crashed, but they are a genuine diagnostic and they're now off by default.

- **Deprecation warnings are silenced wholesale**, not addressed. `-Werror` is off, so several hundred real deprecations (`SecKeychain`, `initWithContentsOfMappedFile:`, `unarchiveObjectWithData:`, and friends) are now warnings nobody reads. They're latent work.

- **The app is signed ad-hoc**, which is fine locally and insufficient for distribution.

- **Not tested beyond launch.** The app builds, launches, and shows its windows. Drawing, saving, printing, export, and LinkBack have not been exercised.
