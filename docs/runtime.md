# The runtime for launchers

This document covers what a release contains, how a launcher gets and checks it, what a Wine
prefix needs from it, and the environment variables it reads. Then the account service: how a
launcher talks to it, how it installs the game, and how it stops.

## Release files

A GitHub release, `v<version>`, has these files:

| File | Contents |
| --- | --- |
| `macrock-engine-<version>.aar` | the runtime: a sealed macOS bundle, in an Apple Archive |
| `xodus-service-<version>-arm64.aar` | the account service, its licenses, its sources (`SOURCES.md`) and its launcher channel's messages (`proto/`), for the launcher's app bundle |
| `index.json` | size and SHA-256 of both archives, the bundle's code directory hash, the service protocol, the requirements, the upstream commits |
| `*.aar.sha256` | the checksum of each archive |
| `SOURCES.md` | the upstream commits and the patches the runtime was built from |

A launcher keeps xodus-service in its app bundle and signs it again with its own identity: the
service serves only a launcher signed like itself (see
[The launcher channel](#the-launcher-channel)). The Keychain trusts the service by its signature,
and each build of it by its code directory hash. Signing the same release again keeps that hash.
A new service build makes macOS ask for the user's password once per account; "Always Allow"
lets that build in for good.

A launcher downloads only the runtime, so it pairs a runtime with the service it carries. To
install a runtime, a launcher:

1. reads `https://github.com/vincenzopio/macrock-engine/releases/latest/download/index.json`
   (`schema` 2);
2. checks `requires`: the macOS version, Rosetta 2, and the Game Porting Toolkit major version;
3. checks `service_protocol`. It must equal the protocol of the service the launcher carries,
   taken from the index of the release that service came from. A release with another protocol
   needs another build of the launcher;
4. downloads the runtime archive over HTTPS and checks its size and SHA-256 against the index;
5. extracts it with AppleArchive;
6. checks the bundle with `codesign --verify --strict`, which detects any changed, missing or
   added file, and its code directory hash against `bundle.cdhash`.

To look for updates, a launcher reads the same `index.json` again: a newer `version` with the
same `service_protocol` can replace the installed runtime. `tests/bundle/check.sh` does the
checks of steps 4 to 6.

## The bundle

```
macrock-engine-<version>.bundle/Contents/
  Info.plist                      dev.macrock.engine, the version and the build number
  Resources/manifest.json         the SHA-256 of every runtime file, requirements, the service
                                  protocol, upstream commits
  Resources/runtime/
    bin/wine, bin/wineserver      Wine, x86_64
    lib/                          Wine, GnuTLS and FreeType,
                                  macrock-engine/x86_64-windows/xgameruntime.dll
    share/macrock-engine/prefix/  what a Wine prefix needs from this runtime
    share/doc/macrock-engine/     licenses and SOURCES.md
  _CodeSignature/                 an ad hoc seal over all of it
```

The runtime can live anywhere: Wine finds its files relative to its own location.

## Prefix setup

`share/macrock-engine/prefix/setup.json` says what a Wine prefix needs from this runtime. The
launcher applies it before each start. `tests/game/prefix-setup.py` does the same for the tests.

| Field | Meaning |
| --- | --- |
| `schema`, `revision` | the format, and the version of the steps |
| `dll_overrides` | the runtime's entries for `WINEDLLOVERRIDES`; the graphics backend adds its own |
| `game_drive` | the drive letter that must map to the game folder, `g:` (registry entries name it) |
| `steps` | what to do, in order; each step has an `id` and one action |

The actions are:

- `copy` a runtime file to a path in the prefix, given by `to`. This installs `xgameruntime.dll`
  in `system32`, where Gaming Services puts it on Windows.
- `reg` imports a registry file with the runtime's `reg.exe`. This registers the Windows App SDK
  classes the game activates, because the SDK's own bootstrapper fails under Wine.

Paths are relative to the runtime, and `to` is relative to the prefix. A step runs again when its
file changes, so a new runtime updates existing prefixes. To know which steps a prefix has, keep
the SHA-256 of each applied step's file: `prefix-setup.py` keeps them in
`PREFIX/.macrock-engine.json`.

Setup that is specific to the game belongs to the launcher: the game drive, GameInput and the
graphics environment.

## Environment variables

These are read by the runtime, in addition to Wine's own:

| Variable | Read by | Effect |
| --- | --- | --- |
| `WINEDLLPATH_PREPEND` | Wine (msync) | DLL directories searched before the runtime's own; this is how D3DMetal or DXMT replace Wine's `d3d*` and `dxgi` |
| `CX_APPLEGPTK_LIBD3DSHARED_PATH` | Wine (msync) | path of D3DMetal's `libd3dshared.dylib` |
| `WINEMSYNC` | Wine (msync) | `1` turns on msync, synchronization based on Mach primitives |
| `CX_ACTIVE_GRAPHICS_BACKEND` | Wine (d3dmetal) | `d3dmetal` selects the D3DMetal window bridge; any other value, DXMT's |
| `MACROCK_FPS_LOG` | Wine (fps-log) | a file that receives one line per second: fps, average and worst frame time, frames over 33.3 ms |
| `MACROCK_WINDOW_TITLE_SUFFIX` | Wine (window-title) | text appended to every non-empty window title, to tell sessions apart |
| `XODUS_SOCKET` | xgameruntime, xodus-service | absolute path of the service's socket, at most 103 bytes; default `/tmp/xodus.sock` |
| `XODUS_KEYCHAIN_SERVICE` | xodus-service | the Keychain service that holds its credentials; default `Xodus Service` |
| `XODUS_LAUNCHER_CHANNEL` | xodus-service | `stdin`: standard input is the launcher's channel, a socket; a signed service needs it |
| `XGAMERUNTIME_LOG_LEVEL` | xgameruntime | `err`, `warn`, `fixme` (the default) or `trace` |
| `XGAMERUNTIME_LOG_FILE` | xgameruntime | a file for its log, instead of standard output |

Run one xodus-service per signed-in account, each with its own socket and Keychain service.
xgameruntime never logs tokens. `trace` is verbose: it writes gigabytes in minutes.

D3DMetal itself reads `D3DM_MTL4`, `1` for its Metal 4 backend and `0` for Metal 3, and
`D3DM_SUPPORT_DXR`.

## The account service

xodus-service holds one account's credentials, in one Keychain item named `credentials` under its
Keychain service. It reads the item once.

At start, the service renews the device's token if the device is registered, then listens on its
socket. It never registers the device itself: signing in does. Until then, token requests get an
empty answer and license requests a failure, as they do while Microsoft can't be reached. While it
runs, the service renews the device's token before it expires.

The service logs to standard error, in colour only on a terminal. `XODUS_LOG` filters the log;
`xodus_service=info` keeps tokens out, and `debug` adds the raw bytes of each request.

### The launcher channel

The Keychain trusts any process that has the service's signature, so a signed service serves only
the launcher:

1. The launcher passes one end of a socket pair as the service's standard input, and sets
   `XODUS_LAUNCHER_CHANNEL=stdin`.
2. Before the service touches the Keychain, it checks that the socket's peer satisfies
   `identifier "dev.macrock.launcher" and certificate leaf = H"..."`. The leaf is the service's
   own certificate, so only the holder of its key can sign such a launcher.

A service signed ad hoc also runs without the channel, for development and tests. The Keychain
trusts it with nothing a release created.

A frame on the channel carries the protobuf magic `0x58445350`, a type of `XodusMessageType`
(`proto/common.proto` in the service's archive) and a message of `proto/launcher.proto`. The
service answers one request at a time, and an answer's type is the request's plus one. `INSTALL`
gets several answers; the last one is its outcome. A type the service does not know gets an empty
answer. A frame holds at most 65,535 bytes, so a launcher sends only the fields of `ServerData`
that `SignIn` names.

| Request | Type | Message | Answer |
| --- | --- | --- | --- |
| `LAUNCHER_HELLO` | 200 | `Hello` | `Hello`: the channel protocol's version, now 2 |
| `LAUNCHER_SIGN_IN` | 202 | `SignIn`: the sign-in page's `ServerData`, and whether the page is the one an `inline_auth_url` named | `SignInResult`: signed in, or another page to show first (`inline_auth_url`) |
| `LAUNCHER_SIGN_OUT` | 204 | `SignOut`: whether to forget the device too | `SignOutResult` |
| `LAUNCHER_ACCOUNT_STATUS` | 206 | none | `AccountStatusResult`: no device, a device, or a signed-in user; and whether the user must sign in again because Microsoft refused their token |
| `LAUNCHER_PROFILE` | 208 | none | `ProfileResult`: the user's gamertag |
| `LAUNCHER_PACKAGE` | 210 | `Package`: a game's Store ID and market | `PackageResult`: the version of the package Microsoft serves now, and the size of its download |
| `LAUNCHER_INSTALL` | 212 | `Install`: a game's Store ID and market, and the absolute path of a staging folder | `InstallUpdate` frames: the progress of the download and of the extraction, then the installed version |

Every answer but `Hello` holds either its result or a `Failure`. The failure's reason is one of:

- the data is not what a sign-in page posts;
- Microsoft can't be reached;
- Microsoft refused, with its status when it gives one;
- the Keychain refused;
- no user is signed in;
- the user does not own the game;
- there is too little room, with the bytes the install needs;
- the package keeps arriving damaged, or can't be extracted;
- the staging folder failed.

Signing in registers the device with Microsoft the first time. Signing out with `device` set
forgets the device too and removes the account's item, as removing the account does.

### Installing the game

`INSTALL` first gets the key of the package's content, so that a user who does not own the game
downloads nothing. Then:

- The package downloads to `<version>.msixvc.part` in the staging folder. That folder must be on
  the volume the game goes to, with room for the package and the game. Downloads of other
  versions there are removed.
- The game is extracted into the staging folder's `game`, replacing what is there. The package
  is then removed, and the launcher moves `game` to where it keeps the game.
- Closing the channel stops the install, and the service with it. The next `INSTALL` resumes
  from the pages already downloaded.

Microsoft's CDNs serve the package over plain HTTP, with no hash to check it against, and
Microsoft signs the package's header with a key only it has. The service checks every page
against the package's own hash tree as it arrives. This catches a damaged or cut download, but
not a package made to pass the check, so a launcher asks the user before it installs.

### Stopping

SIGINT, SIGTERM or the closing of the launcher channel stops the service at any time, even while
a Keychain prompt waits for the user. The service then removes its socket. Its exit status says
why it stopped:

| Status | Meaning |
| --- | --- |
| 0 | stopped by SIGINT, SIGTERM or the closing of the launcher channel |
| 73 | the socket could not be created, for example because another service is using it |
| 77 | the Keychain refused access to the credentials |
| 78 | a signed service without the launcher channel, or a peer that is not the launcher |
