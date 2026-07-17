# GraphSketcher

A fast, simple graph drawing and data plotting app for OS X and iPad. 

## About this fork

This is a fork of [graphsketcher/GraphSketcher](https://github.com/graphsketcher/GraphSketcher) that gets the **Mac app building and running on macOS 27 with Xcode 27, natively on Apple Silicon**. The upstream project targets OS X 10.8 and Xcode 5.1, and no longer builds on a modern toolchain.

The iPad app has **not** been ported and is untouched by this work.

See [PORTING.md](PORTING.md) for what was changed and why, how to build, and the known caveats.

## Download 

If you aren't interested in building from source and contributing to the project, head over to the [The Releases Page](https://github.com/graphsketcher/GraphSketcher/releases) to download GraphSketcher for Mac. (Those releases are upstream's, and predate this fork.)

## Introduction

GraphSketcher is a simple, elegant tool for quickly sketching graphs and plotting data — but you don’t even need data to get started. It’s perfect for reports, presentations, and problem sets where you need to produce sharp-looking graphs on the fly.

## Setting it Free

Graph Sketcher was created by Robin Stewart in 2007. The Omni Group further developed OmniGraphSketcher for Mac and brought it to the iPad in 2010. All GraphSketcher-related source code was open-sourced in 2014.

## What’s Inside

The Mac app is located inside the `App` folder; the iPad source is in the `iPad` folder. Shared code exists in `Model` and `OmniStyle`.

## How to Build (Mac, macOS 27)

### Checking out the source

    git clone --recursive https://github.com/billcstickers/GraphSketcher

Note the `https://` — upstream's README uses a `git://` URL, a protocol GitHub disabled in 2021.

### Applying the OmniGroup patch

The OmniGroup frameworks are a submodule pinned to a 2014 commit of a repository this fork does not control, so the framework fixes cannot be committed there. They ship as a patch instead, and **must be applied before the first build**:

    cd OmniGroup
    git apply ../patches/0001-omnigroup-macos27-port.patch
    cd ..

Re-apply this after any `git submodule update`, which will reset the submodule to the unpatched 2014 commit.

### Prerequisites

Xcode 27 (or the matching beta). Xcode's command line tools alone are not enough — if `xcodebuild` reports it "requires Xcode", point `DEVELOPER_DIR` at a full Xcode install as shown below.

### Building

    DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcodebuild \
      -workspace GraphSketcher-Mac.xcworkspace -scheme ALL -configuration Debug \
      MACOSX_DEPLOYMENT_TARGET=12.0 ARCHS=arm64 VALID_ARCHS=arm64 ONLY_ACTIVE_ARCH=YES \
      GCC_TREAT_WARNINGS_AS_ERRORS=NO OTHER_CFLAGS='$(inherited) -Wno-macro-redefined' \
      CODE_SIGN_IDENTITY="-" OMNI_MAC_CODE_SIGN_IDENTITY="-" build

These settings have deliberately not been baked into the `.xcconfig` files, so building from the Xcode UI will not work as-is. See [PORTING.md](PORTING.md#build-settings) for what each one does and why.

### Code signing

GraphSketcher for Mac is sandboxed. Sandboxed apps must be signed, but they do **not** need a Developer ID for local use: `CODE_SIGN_IDENTITY="-"` above signs ad-hoc, which is enough to run on your own machine. Distributing the app still requires a real signing identity — see Apple's [Code Signing Guide](https://developer.apple.com/library/mac/documentation/Security/Conceptual/CodeSigningGuide/Procedures/Procedures.html).

Do not disable signing entirely (`CODE_SIGNING_ALLOWED=NO`). That strips the sandbox entitlement and the app crashes on launch.

### Building GraphSketcher-iPad

Not ported. Upstream's instructions were to open `GraphSketcher-iPad.xcworkspace` and build the `GraphSketcher-iPad` scheme with an iOS signing identity; that has not been attempted against a modern toolchain.

## License

MIT-style Omni Source License 2007.

See OmniSourceLicense.html in this package.

Enjoy!
