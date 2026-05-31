# Flutter/Dart SLOC reference

Concrete, reusable levers and gotchas for simplifying a Flutter/Dart codebase. Pair with the `goal-sloc` SKILL.md (honesty + preflight rules apply).

## Structural levers (genuine simplification)

### StatefulWidget boilerplate → flutter_hooks
`StatefulWidget` + `State` + `createState`/`initState`/`dispose` + manual listener/controller lifecycle is pure ceremony. `flutter_hooks` `HookWidget` collapses it:
- `useState`, `useEffect` (with a cleanup return for listeners/timers/subscriptions), `useAnimationController`, `useTextEditingController`, `useFocusNode`, `useListenable`/`useValueListenable`, `useMemoized`/`useRef`, `useOnAppLifecycleStateChange`.
- **Caveat: roughly LOC-neutral** (hook ceremony ≈ the State boilerplate it removes) — the win is clarity + fewer lifecycle bugs, not the counter. Do it for quality.
- **Caveat: mutating a hook's `.value` during build notifies mid-build.** For "mutate-during-build" flags use `useRef` + bump a `useState` tick only from callbacks.
- **Blocker:** a widget whose `State` is reached via `GlobalKey<MyState>()` for imperative calls can't trivially become a HookWidget — fix that first (next item).

### GlobalKey<State> imperative calls → reactive controllers
`someKey.currentState!.doThing()` is an antipattern (and blocks hooks). Replace with a small controller passed in:
- A `ChangeNotifier` "signal" (e.g. `FlashController { void flash(int dir){...notifyListeners();} }`) consumed via `useListenable`/`useEffect`; or
- A handle the widget registers its impl onto (`class FooController { Future<void> Function()? _impl; ... }`, widget does `useEffect(() { controller._impl = _doIt; return () => controller._impl = null; })`).
- Net: kills the antipattern, enables HookWidget, keeps behavior. LOC ~neutral.

### Redundant provider/ChangeNotifier layering
A `ChangeNotifier` "provider" that only **mirrors** another controller's state (re-exposes its getters, forwards its methods) is a redundant layer. Make the controller itself the `ChangeNotifier` the UI watches (`ChangeNotifierProvider<TheController>`), and let any platform adapter (e.g. an `audio_service` handler) **observe** the same controller rather than the UI watching a copy. Real win (we cut ~400 lines this way).
- **God-class trap:** when you collapse the layer, don't dump the absorbed concerns (UI-notification, a stream, etc.) permanently into the core class — extract cohesive collaborators (e.g. a `Telemetry`, a `SpectrumSource`) so you don't recreate the smell.

### sealed classes for closed hierarchies
Config/variant hierarchies → `sealed class` enables exhaustive `switch` (compiler-checked, no default branch, less boilerplate) and lets you collapse `fromJson`/`copyWith` with expression bodies.

### Data-driven UI
Repetitive widget builders (rows of label/value/onTap; settings lists) → a declarative list of small spec records + one renderer. Collapses N near-identical builder methods into data.

### Delegate hand-rolled code to a library
- Logging: hand-rolled in-memory ring/`LoggingService` → `package:logging` (`Logger('app.area')`, configure `Logger.root.onRecord` once).
- Marquee/scrolling text → `marquee`; etc.
- Moving code into a dep removes it from your counted tree **and** is usually better — but only when the lib is genuinely the better choice; never add deps just to dodge the counter.

## Dead-weight finds (Flutter-specific)
- **Placeholder subsystems:** e.g. an EQ feature whose settings row is hardcoded `'unavailable'` and whose platform-channel methods are logged no-ops — full plumbing, zero function. Remove end-to-end (model + persistence + channel methods + native handler + UI row).
- **Debug/automation harness in `lib/`:** VM-service driving extensions, test overlays, fakes, and the integration-test entrypoint should not ship in `lib/`. Relocate to a `dev/` area behind a thin seam file in `lib/` (e.g. `lib/debug_hooks.dart` holding nullable callbacks/objects the app populates and the harness reads). Add a debug entrypoint `dev/main_debug.dart` that installs the harness then runs the real app (`flutter run -t dev/main_debug.dart`). Move test doubles to `test/`. (Production code must not `import` the harness.)
- **Unused widgets/providers** left after refactors — find with DCM `check-unused-code`/`check-unused-files`, not grep.

## The platform floor (don't over- or under-estimate it)
`tool/sloc.sh --app`-style metrics often count `android/ macos/ linux/`. Reality:
- **Mostly a floor:** generated Xcode `project.pbxproj`, `MainMenu.xib`, CMake, manifests, gradle, and **binary icons (PNG newline bytes counted as "lines")**. A fresh `flutter create --platforms=...` gives the baseline size — compare to it.
- **Not all floor:** real custom native (Kotlin/Swift `MainActivity`, services, channels) **is** reducible like any code (dead methods, condensable handlers). Don't dismiss it (mistake we made) — but don't expect to gut generated files either.
- Dropping a platform target sheds its whole dir but is a *feature/scope* decision — escalate.

## dart format gotcha
- No CI format-enforcement is common; the repo may not be `dart format`-clean. The default 80-col "tall" style (Dart ≥3.7) can **inflate** code authored denser → sub-agents running `dart format` ad hoc silently re-expand lines and muddy your deltas.
- Pin a consistent format decision up front; **measure format-normalized deltas** (`dart format -l <N>` on a copy) to separate real reduction from formatting noise. Widening line length *purely* to pack lines is gaming (§2).

## Tooling
- **`flutter analyze`** — lints + types; dead-code detection is limited (only unused imports + unused *private* members within a library; misses unused public exports and whole dead files). Necessary, not sufficient.
- **`dart_code_metrics` (DCM)** — `metrics check-unused-files`, `metrics check-unused-code`, `metrics analyze --cyclomatic-complexity=N`. Semantic + reliable; the right tool for dead code + complexity hotspots. (OSS 5.7.x still installs via `dart pub global activate dart_code_metrics`.)
- **Code knowledge-graph tools (e.g. GitNexus)** — great for architecture navigation (dependency graph, clusters, fan-in/out, impact). But their Dart **call-graphs can be incomplete** (miss tear-offs, cross-file, string-interpolated calls) → their "uncalled function" lists have false positives; cross-check before deleting. (Needs a modern Node + the `tree-sitter-dart` native grammar built.)
- A custom AST/import-graph script (resolve relative + `package:` imports) reliably finds orphan files + coupling and avoids grep's relative-import blindness.

## App-driving feedback loop (essential)
- Register debug-only VM-service extensions (`developer.registerExtension('ext.app.*', ...)`) gated by `kDebugMode`; drive them from a script over the Dart VM Service WebSocket (the `drive.py` pattern: inspect state, play/tap/nav, screenshot via a `RepaintBoundary` on desktop / `adb screencap` on device).
- Verify on **both** a mobile target (emulator/device) **and** a desktop target (`-d linux`/`-d macos`) — desktop bypasses ADB flakiness (useful in WSL2; needs cmake/ninja/clang/GTK for `-d linux`). Note headless desktop may have no audio device (SoLoud runs a null/real-time clock) — judge playback by state advancing, not sound.
- Run `integration_test/` on a device for end-to-end flows. Ensure the test entrypoint builds the **real themed app** (registering theme extensions etc.), or first-frame `Theme.of(context).extension<...>()!` will null-crash.
