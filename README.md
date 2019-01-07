# SourceKit-LSP

SourceKit-LSP is an implementation of the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) for Swift and C-based languages. It provides features like code-completion and jump-to-definition to editors that support LSP. SourceKit-LSP is built on top of [sourcekitd](https://github.com/apple/swift/tree/master/tools/SourceKit) and [clangd](https://clang.llvm.org/extra/clangd.html) for high-fidelity language support, and provides a powerful source code index as well as cross-language support. SourceKit-LSP supports projects that use the Swift Package Manager.


## Getting Started

SourceKit-LSP is under heavy development! The best way to try it out is to build it from source. You will also need a Swift development toolchain and an editor that supports LSP.

1. Install the `swift-DEVELOPMENT-SNAPSHOT-2019-01-05-a` toolchain snapshot from https://swift.org/download/#snapshots. Set the environment variable `SOURCEKIT_TOOLCHAIN_PATH` to the absolute path to the toolchain or otherwise configure your editor to use this toolchain. See [Toolchains](#toolchains) for more information.

2. Build the language server executable `sourcekit-lsp` using `swift build`. See [Building](#building-sourcekit-lsp) for more information.

3. Configure your editor to use the newly built `sourcekit-lsp` executable. See [Editors](Editors) for more information about editor integration.

4. Build the project you are editing with `swift build`. The language server depends on the build to provide module dependencies and to update the global index.


## Building SourceKit-LSP

SourceKit-LSP is built using the [Swift Package Manager](https://github.com/apple/swift-package-manager).

For a standard debug build:

```sh
$ swift build
```

After building, the server will be located at `.build/debug/sourcekit-lsp`, or a similar path, if you passed any custom options to `swift build`. Editors will generally need to be provided with this path in order to run the newly built server - see [Editors](Editors) for more information about configuration.

### Building on Linux

The C++ code in the index requires `libdispatch`, but unlike Swift code, it cannot find it automatically on Linux. You can work around this by adding a search path manually.

```sh
$ swift build -Xcxx -I<path_to_swift_toolchain>/usr/lib/swift
```

### Using the Generated Xcode Project

If you would like to use the Xcode project generated by `swift package generate-xcodeproj`, you will need to include the xcconfig overrides to avoid a link error.

```sh
$ swift package generate-xcodeproj --xcconfig-overrides overrides.xcconfig
$ open SourceKitLSP.xcodeproj
```

## Debugging

You can attach LLDB to SourceKit-LSP and set breakpoints to debug. You may want to instruct LLDB to wait for the sourcekit-lsp process to launch and then start your editor, which will typically launch
SourceKit-LSP as soon as you open a Swift file:


```sh
$ lldb -w -n sourcekit-lsp
```

If you are using the Xcode project, go to Debug, Attach to Process by PID or Name.

### Print SourceKit Logs

You can configure SourceKit-LSP to print log information from SourceKit to stderr by setting the following environment variable:

```sh
SOURCEKIT_LOGGING="N"
```

Where "N" configures the log verbosity and is one of the following numbers: 0 (error), 1 (warning), 2 (info), or 3 (debug).

## Toolchains

SourceKit-LSP depends on tools such as `sourcekitd` and `clangd`, which it loads at runtime from an installed toolchain.

### Recommended Toolchain

Use the `swift-DEVELOPMENT-SNAPSHOT-2019-01-05-a` toolchain snapshot from https://swift.org/download/#snapshots. SourceKit-LSP is still early in its development and we are actively adding functionality to the toolchain to support it.

### Selecting the Toolchain

After installing the toolchain, SourceKit-LSP needs to know the path to the toolchain.

* On macOS, the toolchain is installed in `/Library/Developer/Toolchains/` with an `.xctoolchain` extension. The most recently installed toolchain is symlinked as `/Library/Developer/Toolchains/swift-latest.xctoolchain`.  If you opted to install for the current user only in the installer, the same paths will be under the home directory, e.g. `~/Library/Developer/Toolchains/`.

* On Linux, the toolchain is wherever the snapshot's `.tar.gz` file was extracted.

Your editor may have a way to configure the toolchain path directly via a configuration setting, or it may allow you to override the process environment variables used when launching `sourcekit-lsp`. See [Editors](Editors) for more information.

Otherwise, the simplest way to configure the toolchain is to set the following environment variable to the absolute path of the toolchain.

```sh
SOURCEKIT_TOOLCHAIN_PATH=<toolchain>
```

## Indexing While Building

SourceKit-LSP uses a global index called [IndexStoreDB](https://github.com/apple/indexstore-db) to provide features that cross file or module boundaries, such as jump-to-definition or find-references. To efficiently create an index of your source code we use a technique called "indexing while building". When the project is compiled for debugging using `swift build`, the compiler (swiftc or clang) automatically produces additional raw index data that is read by our indexer. Producing this information during compilation saves work and ensures that any time the project is built the index is updated and fully accurate.

In the future we intend to also provide automatic background indexing so that we can update the index in between builds or to include code that's not always built like unit tests. In the meantime, building your project should bring our index up to date.

## Status

SourceKit-LSP is still in early development, so you may run into rough edges with any of the features. The following table shows the status of various features when using the [Recommended Toolchain](#recommended-toolchain). See [Caveats](#caveats) for important known issues you may run into.

| Feature | Status | Notes |
|---------|:------:|-------|
| Swift | ✅ | |
| C/C++/ObjC | ❌ | [clangd](https://clang.llvm.org/extra/clangd.html) is not available in the recommended toolchain. You can try out C/C++/ObjC support by building clangd from source and putting it in `PATH`.
| Code completion | ✅ | |
| Quick Help (Hover) | ✅ | |
| Diagnostics | ✅ | |
| Fix-its | ❌ | |
| Jump to Definition | ✅ | |
| Find References | ✅ | |
| Background Indexing | ❌ | Build project to update the index using [Indexing While Building](#indexing-while-building) |
| Workspace Symbols | ❌ | |
| Refactoring | ❌ | |
| Formatting | ❌ | |
| Folding | ✅ | |
| Syntax Highlighting | ❌ | Not currently part of LSP. |
| Document Symbols | ❌ |  |


### Caveats

* SwiftPM build settings are not updated automatically after files are added/removed.
	* **Workaround**: close and reopen the project after adding/removing files

* SourceKit-LSP does not update its global index in the background, but instead relies on indexing-while-building to provide data. This only affects global queries like find-references and jump-to-definition.
	* **Workaround**: build the project to update the index
