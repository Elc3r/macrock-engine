# Building macrock-engine

## Build host

You need an Apple Silicon Mac with macOS 27, its Command Line Tools and
[Homebrew](https://brew.sh). `packaging/macos/bootstrap.sh` installs the rest, and it is safe to
run again:

- the Homebrew formulae that provide headers and build tools;
- llvm-mingw, the compiler for the Windows half of Wine;
- an archive of Gcenx's macOS Wine build, which provides the x86_64 GnuTLS and FreeType
  libraries;
- the Rust toolchain that xodus-service needs.

Every download is pinned to a version and a SHA-256 in `packaging/macos/pins.sh`.

## Building the runtime

`packaging/macos/build.sh` builds everything into `build.noindex/dist/macrock-engine-<version>`.
The steps are:

1. **Wine's build tools**, natively for arm64, so that the build runs no x86_64 tools under
   Rosetta.
2. **Wine**, for x86_64. The Unix half is built with Apple clang, the Windows half with
   llvm-mingw. Vulkan is left out: graphics come from D3DMetal or DXMT, never from wined3d.
3. **GnuTLS and FreeType**, copied next to Wine from the Gcenx archive.
4. **xgameruntime**, with CMake, llvm-mingw and Wine's IDL compiler.
5. **xodus-service**, with cargo, for arm64. It keeps the Microsoft account's credentials in the
   Keychain. `xodus-cli`, a development tool kept out of the runtime, goes to `build.noindex/bin`:
   its `login` signs an account in for the tests.

The result can be moved: Wine finds its files relative to its own location.

First of all, `build.sh` generates the patched Wine tree with `tools/engine import wine` (see
[patches.md](patches.md)). Its arguments are passed to that command: for example,
`build.sh --without fps-log` builds without one topic. When the patches have not changed, the
tree is left alone and the build is incremental.

| Variable | Default | Meaning |
| --- | --- | --- |
| `JOBS` | 4 | parallel jobs; Macs with 8 GB should stay at 4–6 |
| `ENGINE_WORK` | `build.noindex` | where trees, downloads, build directories and output go |
| `LLVM_MINGW` | the one bootstrap.sh downloads | an llvm-mingw you already have |
| `GCENX_ARCHIVE` | the one bootstrap.sh downloads | a Gcenx archive you already have; its SHA-256 is checked |
| `SIGNING_IDENTITY` | ad hoc | the identity that signs the built xodus-service |
| `RECONFIGURE` | unset | run Wine's `configure` again |
| `REBUILD_TOOLS` | unset | build the arm64 tools again |

A service signed with a real `SIGNING_IDENTITY` serves only a launcher signed with the same
certificate, through the launcher channel ([runtime.md](runtime.md#the-launcher-channel)). Started
any other way, it exits with status 78, so the tests need the ad hoc default. The Keychain trusts
each build of the service on its own: a new build asks for the password again, once per item.

## Releases

`packaging/macos/package.sh` turns the build into the release files, in `build.noindex/packages`.
[runtime.md](runtime.md) describes them. The bundle leaves out development files such as headers,
import libraries and man pages. Every library in it must have its license texts in
`packaging/macos/licenses`, or packaging fails.

`package.sh` refuses uncommitted changes, and patch trees that do not match their series. With
`--dev` it packages them anyway and marks the release as dirty.

`tests/bundle/check.sh` checks the result as a launcher will receive it. Then
`packaging/macos/release.sh` uploads it to a draft GitHub release named `v<version>`. The commit
must already be on GitHub, and the release needs the GitHub CLI, signed in. A draft stays private
until it is published.

## Tests

Most tests take the runtime to check as an optional argument. The default is the build in
`build.noindex/dist`, so the same tests also run on a runtime extracted from a release. Each
script's header explains its options.

| Command | What it checks | Needs |
| --- | --- | --- |
| `tests/wine-tests/run.sh <dll>` | Wine's own tests of one DLL | the Wine build tree |
| `tests/afunix/run.sh` | AF_UNIX sockets between a Windows program and macOS, both ways | |
| `tests/tls/run.sh` | WinHTTP rejects bad server certificates | network |
| `tests/window-probe/run.sh <gptk>/lib` | a D3D12 swapchain fills its window: windowed, resized, fullscreen | GPTK, Screen Recording permission |
| `tests/xgameruntime/run.sh` | xgameruntime's unit tests and its connection to the service | |
| `tests/xodus/run.sh` | xodus-service's protocol, and xgameruntime talking to it | the built xodus-service, signed ad hoc |
| `tests/xuser-live/run.sh` | sign-in, Xbox Live tokens and the game license with a real account | network, `XODUS_SOCKET` with an account signed in by `xodus-cli login`, `GAME_CONFIG` |
| `tests/bundle/check.sh` | a release: checksums, seal, manifest, Wine, the service | |
| `tests/game/run.sh` | plays the game | `GAME`, `GPTK_LIB`, `XODUS_SOCKET`, a prefix macrock prepared or `GAMEINPUT_SCRIPT` |

The runtime's own logs never contain tokens. Raw Wine traces (`WINEDEBUG`) can, so do not share
them.

`tests/game/run.sh` can play in a prefix of the macrock launcher, which already has GameInput:
macrock keeps one per account in `~/Library/Application Support/macrock/prefixes`. Quit macrock
first. Any other prefix needs `GAMEINPUT_SCRIPT`, an installer of GameInput from the game's own
redistributable.

The tests of `tools/engine` run with `/usr/bin/python3 -m unittest discover -s tools/tests`.

The Rust tests of xodus-service and xodus run in the tree that `build-xodus.sh` generates, with the
toolchain `bootstrap.sh` installs:

```sh
cd build.noindex/src/xodus
cargo +1.98.0 test --locked -p xodus-service -p xodus -- \
  --skip test_minecraft_win_auth --skip test_get_xbox_live_dev_token
```

The two skipped tests come from upstream and sign in to Microsoft's servers.
