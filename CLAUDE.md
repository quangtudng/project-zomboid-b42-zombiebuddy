# ZombieBuddy — internals

Fork of `zed-0xff/ZombieBuddy`, used as a submodule of `project-zomboid-b42-stable`. This file is how the code works, for developing it. How to *use* it (mod.info keys, `@Patch` bindings, agent args, Lua API) is in `doc/*.md` and README; don't repeat that here. Paths below are relative to `java/src/main/java/me/zed_0xff/zombie_buddy/` unless rooted.

## Big picture

One fat JAR (`ZombieBuddy.jar`, shadowJar) is a Java agent (`Premain-Class: Agent`, can redefine and retransform classes). It patches game classes with ByteBuddy and loads other mods' JARs, which declare patches with ZB's own annotations. Flow:

1. `Agent.premain`: parses args, locks policy, picks approval frontend (lazily), exposes ZB's own `@Exposer.LuaClass` classes, loads the bundled `patches.jar` (+ `experimental.jar` if `experimental`), any `patches_jar=` JARs, then `Loader.initConfig()` + `Loader.preloadMods()`. All of this runs in phase `PREMAIN`, before game `main`.
2. Bundled patches hook the game lifecycle (below). The key one: `Patch_ZomboidFileSystem` wraps `ZomboidFileSystem.loadMods(List)`. On enter it calls `Loader.maybeReorderMods` then `Loader.loadMods(toLoad)`. So mod JARs load when the game loads its mod list: main menu on client, startup on server. That's phase `MAIN`.
3. `Loader.loadMods` → for each approved JAR: `loadJar` → `Pipeline.transformPatchJar` (rewrite ZB annotations into ByteBuddy ones) → `ModJarInjector` (define classes) → `ApplyPatchesFromPackage` (run `Main.main`, expose Lua classes, `PatchEngine.applyPatches`).

## Bundled patches (`patches/`, always on)

Built as their own source set → `patches.jar`, embedded as a resource inside the agent JAR and loaded via `Loader.loadResourceJar`. They are ZB's game hooks; most just fire `Callbacks.*`:

| Patch | Target | Does |
|---|---|---|
| `Patch_ZomboidFileSystem` | `ZomboidFileSystem.loadMods(String)` / `loadMods(List)` / `getAllModFoldersAux` | track mod-set id; **trigger Java mod loading**; reorder mods; B41 `41/` subdir support |
| `Patch_GameWindow` | `GameWindow.init` exit, `DoLoadingText` | `onGameInitComplete` (client); flag that loading text works |
| `Patch_GameServer` | `GameServer.startServer` enter | `onGameInitComplete` (dedicated server) |
| `Patch_Display` | `lwjglx Display.create` exit | `onDisplayCreate` (Shift-key check for forced approval dialog) |
| `Patch_Exposer` | `LuaManager$Exposer.exposeAll` exit | `afterExposeAll` → `Exposer` pushes queued Lua classes |
| `Patch_Core` | `Core.EndFrameUI` enter | `onEndFrameUI` (watermark, ImGui approval dialog) |
| `Patch_GameLoadingState` | `GameLoadingState.exit` | optional suppression of sandbox-options log spam |

`Callbacks` = ZB's internal event bus (`ONCE` / `MANY` / `FREQUENT`), separate from PZ `Events`. Errors in listeners are caught and logged (capped at 50).

`patches/experimental/` → `experimental.jar`, only with the `experimental` arg. Its `PreMain` starts `HttpServer` (runs Lua sent over HTTP, queued onto the game thread; bind `lua_server_host` default `127.0.0.1`, `lua_server_port`, `random` writes the port to `<cache>/zbLuaAPI.txt`; ZBSpec test framework uses it), `WatchesAPI`, `JavaStateDumper`. Also: Kahlua/Lua event and packet logging, log overlay, macOS Retina fix, `Patch_hide_dirs` (`DirHider`: hides `.git`/`tmp`/IDE dirs from mod folder scans), symlink fix for `ZomboidFileSystem.getRelativeFile`. **Remote Lua execution: never enable `lua_server_*` on a reachable host.**

## Mod discovery and approval (`Loader.loadMods`)

- For each mod id (preload ids first, then the game's list, deduped): `ChooseGameInfo.getAvailableModDetails`. On B42, `JavaModInfo` tries the version-dir `mod.info`, then the common-dir one, then common `mod.info` + version-dir JAR (`parseMerged`). B41 = `mod.getDir()`.
- `JavaModInfo` requires both `javaJarFile` and `javaPkgName`. It skips `media/java/client/` JARs on a server and `media/java/server/` JARs on a client (`Utils.isServer()` = `LuaManager.GlobalObject.isServer`). It takes the Workshop id from the path `.../content/108600/<id>/...`.
- Structural skips: ZB's own package, which is ZB itself. That's where `SelfUpdater` runs (below). Also skipped: a package also declared by a *later* mod in the list (last wins).
- For every JAR: sha256, a Steam Workshop details fetch (ban status, uploader; `SteamWorkshop`, HTTP with a cache), a `KnownAuthors` fetch (signed `authors.json` from upstream GitHub master, cached in `config_dir`), and a ZBS check.
- **Network on every mod load** unless `policy=allow-all` (allow-all skips ZBS entirely). Timeouts: `http_client_timeout` (5 s); in-memory cache `http_cache_ttl`.
- Decision order per JAR in `isJarAllowedByPolicy`: stored decision for this hash (allow/deny) → `allow-all` (stores allow) → `deny-new` (block) → `prompt` (decided earlier in batch).
- `prompt`: collect undecided / banned / forced entries into one batch → `approvalFrontend().approvePendingMods` → `applyBatchApprovalLines` (session table + optional persist + trust-author + preload). Signed JAR from a trusted author, not banned → auto-allow for the session only (not persisted per hash). Shift held at load → force the dialog even for decided JARs.
- Then the block reasons, in order: banned → ZBS invalid → policy. Load state is recorded in `g_jarLoadStatus` (`ZombieBuddy.getActiveJavaMods()` for Lua).
- Approvals saved to `<config_dir>/mod_approvals.json` (`ModApprovalsStore`, migrates the legacy `java_mod_approvals.txt`). Other state in `<config_dir>/config.json` (`Config` record: `preload_mods`, `auto_fix_mod_order`, `fix_approval_dialog_cursor`, trusted authors). `config_dir` default `~/.zombie_buddy`.
- `loadJar` re-hashes just before loading and aborts if the hash differs from the approved one (TOCTOU guard).

### Frontends (`frontend/`)

`ModApprovalFrontends.resolve`: `auto` → ImGui if the game window exists (`ImguiModApprovalFrontend`, drawn from `onEndFrameUI`); else console if `GameServer.server` (dedicated); else Swing. Console reads `y/n` from stdin and **blocks** (null stdin = deny). Swing runs a child JVM (`SwingApprovalMain`, `-Djava.awt.headless=false`) speaking `JarBatchApprovalProtocol`. TinyFD = native per-JAR dialogs.

### ZBS signatures (`ZBSVerifier`, `KnownAuthors`, `doc/ModSigning.md`)

`<jar>.zbs` sidecar = SteamID64 + Ed25519 signature over the JAR sha256. The public key is scraped from the author's Steam profile (`JavaModZBS:<hex>`). For Workshop installs the signer must equal the Workshop uploader. Result flags `ModFlags` (`MF_SIGNED`, `MF_VALID`, `MF_TRUST_AUTHOR`, `MF_PERSIST`, `MF_PRELOAD`, `MF_ACTIVE`; some are sticky across reloads). A missing `.zbs` is allowed unless `allow_unsigned_mods=false`; an invalid one is always blocked. `authors.json` is signed with a hardcoded upstream key (`KnownAuthors.AUTHORS_PUBKEY_HEX`): we can't re-sign our own list without changing that constant.

### Preload

`javaPreload=true` in mod.info **and** `ZB-Preload: true` in the JAR manifest **and** a valid ZBS **and** user approval → stored in `config.json`. On the next launch `Agent.premain` → `Loader.preloadMods` loads it in phase `PREMAIN` (runs `PreMain.premain`), before the game. It's removed automatically when the mod leaves the `default` mod set, or when its checks fail.

## Transform pipeline (`transformers/`)

Mod JARs aren't loaded as-is. `Pipeline.transformPatchJar` reads every class in memory and runs the default transformers in order (ASM tree or ByteBuddy):

| id | Class | Does |
|---|---|---|
| `compat` | `ZB2Compat` | upgrade ZB 2.x annotation forms |
| `resolve` | `asmtree/Resolver` | resolve multi-name/alternative names in annotations (`@Patch.Field({"a","b"})` etc.) |
| `convert` | `asmtree/Converter` | append the equivalent ByteBuddy annotation next to each `@Patch.*` that carries `@Internal.Meta` (ZB annotations are 1:1 façades over ByteBuddy's `Advice.*` / `bind.annotation.*`) |
| `bind` | `bytebuddy/Unshadow` | `@Shadow(className=…)` stub types → real target types |
| `shadow` | `asmtree/ShadowRewrite` | direct field/method/`new` on shadow targets → inlined `VarHandle`/`MethodHandle` calls (`ShadowHandles`) |
| `pub-cond` | `Publicizer` (conditional) | make members public if earlier steps converted anything |

Output `TransformedJar`: class bytes, resources, sorted `@Patch` class names *in `javaPkgName`* (patches outside the package are ignored), `Main` / `PreMain` names. To debug a transform: verbosity ≥ TRACE (2) dumps each step via `jardump/AsmDump`; `jardump.sh` / `jardump/Main` is a standalone CLI for it. `PatchAnnotationProcessor` (javac SPI, `META-INF/services`) validates patches at compile time. It's enabled for `patches`, `experimental` and `testpatches`.

## Class loading (`ModJarInjector`)

- Default (mod JARs and bundled JARs) = `injectMerged`: classes are defined with `ClassInjector.UsingUnsafe` **directly into ZB's own class loader** (app/system loader), so patch static state is single. A class name that already exists there is silently skipped (first definition wins), so packages must be unique.
- Only a file named `testpatches.jar` gets an isolated `ByteArrayClassLoader.ChildFirst` (tests isolation path).
- If a target class lives in another loader, `exposePatchesInClassLoader` injects the patch classes there too, at transform time.

## Applying patches (`PatchEngine.applyPatches`)

- Refuses targets under `me.zed_0xff.`. Groups by (class, method): any number of **Advice** patches per method (they stack, one `.visit` each), one **MethodDelegation** per method (the last one wins, with a warning).
- Targets already loaded → `RETRANSFORMATION` + `disableClassFormatChanges` (method bodies only: no new fields/methods). Not yet loaded → woven on first define (format changes allowed). Handles targets that load between scan and install ("slip-in") and the mixed case (forces deferred targets to load). `warmUp=true` forces the target to load first.
- `PatchTransformer.preparePatch` (per target) fills `@Patch.VarHandle` / `@Patch.MethodHandle` static fields and returns null (patch dropped) if a non-optional handle can't be resolved. This is how patches survive game renames.
- **Overload matching** (`buildAdviceMethodMatcher`): an advice param without an annotation counts as a positional target argument, and its type narrows which overloads match. `@Argument(n)` sets a minimum arity. `@AllArguments` / no params match every overload unless `strictMatch`. Most `PatchedTestOverloadedMethods*` tests cover this. Change it carefully.
- Constructor delegation (`methodName="<init>"`) replaces the ctor with `Object()` + delegate. Field initializers still run.
- `ApplyPatchesFromPackage` runs `Main`/`PreMain` once per class, and applies Lua exposure and patches only in the **first** phase that loads a package.

## Lua side

- `Exposer`: classes with `@Exposer.LuaClass` (name may be dotted → nested tables) and static `@LuaMethod(global=true)` are queued until the game's `exposeAll`, then registered. Mods loaded later are exposed right away. `expose_classes=` adds by name.
- ZB's own Lua API: `ZombieBuddy` (version, `getActiveJavaMods`, closure introspection), `EventsAPI`, `WatchesAPI` (experimental), `zbUtils` dev helpers (`doc/DevDebugFunctions.md`), `LuaJSON`.
- Workshop mod Lua (`42/media/lua/client/`): `ZombieBuddy.lua` shows an install-instructions dialog at the main menu if the agent isn't running; `ZombieBuddy_Options.lua` = mod options (watermark opacity, etc.). `Watermark` = in-game ZB badge plus status line ("restart recommended", etc.).

## Self-update (`SelfUpdater`) — matters for the fork

While handling the ZombieBuddy mod's own entry, `getExclusionReasonSuffix` checks the Workshop copy `libs/ZombieBuddy.jar`. If it's fully jar-signed by the **upstream** cert (`EXPECTED_FINGERPRINT`) and its `Implementation-Version` is newer, it **overwrites the running agent JAR** (`.bak` backup; `.new` if locked). Consequences:
- If the upstream Workshop ZombieBuddy is enabled next to our fork build, upstream silently replaces our JAR on the next mod load whenever its version is higher than `java/VERSION`.
- Our builds can't self-update anyone (not signed with that cert). To deploy the fork, install the JAR ourselves and don't depend on Workshop item `3619862853`. Alternatively, patch `SelfUpdater` to be inert.

## Build and tests (`java/build.gradle`)

- Source sets: `main` (excludes `patches/**`), `patches` → `patches.jar`, `experimental` → `experimental.jar` (both shadow-relocate gson to `zb.com.google.gson` and are embedded in the shadowJar), `test`, `testVanilla`, `testPatched`, `testjar`, `testpatches`.
- Shaded deps: byte-buddy + agent 1.18.8, asm-tree 9.9.1 (**keep matching byte-buddy's ASM version**), bcprov (→ `zb.org.bouncycastle`), gson. `compileOnly` game classpath from `-PgameClasspath=a,b` / `PZ_CLASSPATH` / `-Dpz.classpath`. Compile is skipped if classes already exist and no classpath is given.
- `-PjavaVersion=17` (shipped, game JRE compat) or default 25 → `build/jdk<N>/`. Repo-root `libs` symlinks to `build/jdk17/libs`, which is where mod.info `javaJarFile=../libs/ZombieBuddy.jar` points.
- ⚠️ `build` is finalized by `copyNativeDll`, which **throws if `c/windows/zbNative.dll` is missing**, and that file is gitignored (only `zbNative.c` + `Makefile` are in the repo). Build the DLL (`c/windows/Makefile`, needs a MinGW cross-compiler), or run `gradle shadowJar` instead of `build`. Signing: `signJar` (jarsigner via macOS `KeychainStore`, only if `signing.storeType=KeychainStore` in `gradle.properties`, which is set). Without the upstream key it fails on non-macOS / without that alias, so override `-Psigning.storeType=none` or drop the line. `zb-gradle-plugin` adds `signJarZBS` (needs `zbsPrivateKeyFile` / `zbsSteamID64` in `~/.gradle/gradle.properties`).
- Tests:
  - `unitTest` (`test/`): no agent; game jar on the classpath.
  - `test_vanilla` (`test_vanilla/`): target classes without the agent (baseline).
  - `test_patched` (`test_patched/`): JVM started with `-javaagent:<shadowJar>=patches_jar=<testpatches.jar>:me.zed_0xff.zombie_buddy.testpatches,config_dir=.config,prop_prefix=zb`, plus gson/bcprov on the boot classpath. Targets live in `testjar/` (`testjar.*`), patches in `testpatches/`.
  - Recipe for a new patch feature: target method in `testjar/src/testjar/`, `@Patch` class in `testpatches/`, assertion in `test_patched/PatchedTest<Feature>.java`, and a vanilla baseline if behavior differs.
- `Rakefile` / `lib/tasks/*.rake`: upstream author's macOS-local wrappers (`PROJECT_ROOT ~/projects/zomboid/versions/{unstable,42.12}/java`). `rake test TESTS=Name` picks the suite from the name prefix (`Patched*` / `Vanilla*` / else unit).

## Other parts

- `installer/` (Go): Windows installer. Finds the game via Steam `libraryfolders.vdf`, copies the JAR + DLL, and patches `ProjectZomboid64.json` `vmArgs`, Steam launch options and/or the alternate `.bat` launcher. Has preview, uninstall, and fixtures in `testdata/launchers/<ver>/{vanilla,patched}`. `go test ./...`.
- `c/windows/zbNative.c`: `-agentlib:zbNative` shim. Adds `jre64\bin` to the DLL search path, then proxies to `instrument.dll` so `-javaagent` works on Windows.
- `authors.json` + `authors/` + `lib/tasks/authors.rake`: known Java-mod authors list, signed by upstream's key and fetched at runtime from upstream master.
- `Reflect` (cached reflective / `MethodHandle` access, used everywhere), `Logger` (levels -2..2, `ZB_VERBOSITY`), `Utils` (sha256, version compare, `isServer`, paths).

## Fork rules

- Keep changes minimal and isolated (new files over edits) so `git fetch upstream && git merge upstream/master` stays clean. Note fork-only behavior in this file.
- Bump `java/VERSION` for fork builds in a way that sorts correctly against upstream (`Utils.isVersionNewer`), or neutralize `SelfUpdater` (see above).
- `.workshopignore` excludes `CLAUDE.md`, `AGENTS.md`, `.claude*` from Workshop uploads.
