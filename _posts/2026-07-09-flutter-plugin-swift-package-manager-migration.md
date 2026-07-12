---
title: "Migrating a Flutter Plugin to Swift Package Manager (and the Web Bug I Found on the Way)"
date: 2026-07-09 00:00:00 +0600
categories: [Flutter, Development]
tags: [flutter, dart, swift-package-manager, ios, cocoapods, js-interop, open-source, plugin]
description: Flutter is moving plugins off CocoaPods to SPM. Here's the exact layout it demands, how to keep both toolchains working at once, and the one-word class-name typo that was silently breaking flutter build web for everyone using my package.
---

I maintain [`open_file_safe_plus`](https://pub.dev/packages/open_file_safe_plus) — a Flutter plugin that opens a local file with whatever the platform thinks should open it. `Intent` on Android, `UIDocumentInteractionController` on iOS, the native shell on desktop, JS interop on web. It's a fork of the venerable `open_file`, kept alive because the original went quiet and people still need to open PDFs.

An issue came in asking for **Swift Package Manager** support. Flutter is migrating iOS plugins off CocoaPods, and eventually "eventually" arrives.

I sat down expecting a fiddly afternoon of build configuration. What I got was a fiddly *twenty minutes* of build configuration — and then the discovery that my package had been **completely broken on web** for every consumer, for months, because of a missing word in a class name.

Both halves of that are worth writing down.

---

## Part 1: What SPM Actually Demands

If you've only ever consumed Swift packages, the migration sounds abstract. It isn't. It is almost entirely about **where your files physically live.**

CocoaPods doesn't care. You point a glob at your sources and it finds them:

```ruby
s.source_files = 'Classes/**/*'
s.public_header_files = 'Classes/**/*.h'
```

`Classes/`. Anything. Whatever. The podspec is a treasure map, and as long as the map is right, the treasure can be buried anywhere.

SPM does **not** work like that. SPM infers your module from the directory structure, and the structure is not a suggestion. For a Flutter plugin, it must be:

```
ios/
└── <plugin_name>/                          ← the SPM package root
    ├── Package.swift
    └── Sources/
        └── <plugin_name>/                  ← the target, named like the module
            ├── OpenFilePlugin.m
            └── include/
                └── <plugin_name>/          ← public headers, nested AGAIN
                    └── OpenFilePlugin.h
```

Note the plugin name appears **three times**, and the `include/` directory contains another directory *also* named after the plugin. That last nesting is the one that trips everybody, and it isn't arbitrary: it's what makes `#import <open_file_safe_plus/OpenFilePlugin.h>` resolve as a **module-qualified** import rather than a bare file path.

So the migration is, at its core, two `git mv`s:

```
ios/Classes/OpenFilePlugin.m
  → ios/open_file_safe_plus/Sources/open_file_safe_plus/OpenFilePlugin.m

ios/Classes/OpenFilePlugin.h
  → ios/open_file_safe_plus/Sources/open_file_safe_plus/include/open_file_safe_plus/OpenFilePlugin.h
```

Git records those as 100%-similarity renames — not a single byte of Objective-C changed. That's a good sign in a migration: if your *code* has to change to move build systems, something has gone wrong.

Then the manifest:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "open_file_safe_plus",
    platforms: [
        .iOS("12.0")
    ],
    products: [
        .library(name: "open-file-safe-plus", targets: ["open_file_safe_plus"])
    ],
    targets: [
        .target(
            name: "open_file_safe_plus",
            dependencies: [],
            cSettings: [
                .headerSearchPath("include/open_file_safe_plus")   // ← the nested include dir
            ]
        )
    ]
)
```

That `headerSearchPath` is what closes the loop with the doubly-nested `include/` folder above.

One gotcha that cost me a build: the implementation file's own import had to change, because the header is now two directories deeper than the `.m` file:

```diff
- #import "OpenFilePlugin.h"
+ #import "./include/open_file_safe_plus/OpenFilePlugin.h"
```

And two lines in `ios/.gitignore`, because SPM leaves litter:

```
.build/
Package.resolved
```

---

## The Part Everyone Gets Wrong: Don't Delete the Podspec

Here is the mistake I very nearly made: migrating to SPM and *removing CocoaPods support*.

Do not do that. Not yet.

Flutter is **in transition**. Some of your users are on a Flutter version with SPM enabled. Many are not. If you ship an SPM-only plugin, every consumer on a CocoaPods project — which today is still most of them — gets a build failure and a GitHub issue with your name on it.

The plugin has to build under **both** toolchains, from **the same files**, for the whole transition period.

Which turns out to be easy, because the podspec was always just a treasure map. Repoint it:

```ruby
# Before
s.source_files        = 'Classes/**/*'
s.public_header_files = 'Classes/**/*.h'

# After
s.source_files        = 'open_file_safe_plus/Sources/open_file_safe_plus/**/*.{h,m}'
s.public_header_files = 'open_file_safe_plus/Sources/open_file_safe_plus/include/**/*.h'
```

Two globs. That's the entire CocoaPods side of the migration.

Now `Package.swift` and the `.podspec` describe **the same physical files** in the same location. SPM finds them by structure, CocoaPods finds them by glob, and neither knows or cares that the other exists. No duplicated sources, no `#ifdef`, no second copy to keep in sync.

The `ios/Classes/` directory is gone entirely — not duplicated, *moved*. There is exactly one copy of the truth.

I also added a throwaway `example/` app and actually ran it on the iOS Simulator, because "it compiles" and "it works" are different claims, and a plugin migration that hasn't been run on a device is a hypothesis.

---

## Part 2: The Bug That Was Waiting for Me

While I was in there, I ran the pub.dev static analysis to clean up warnings before publishing. And it surfaced something that had nothing to do with SPM.

The plugin's public API is `OpenFileSafePlus`. It's the name in the README, in the examples, in every doc, in the example app. That's the class you call:

```dart
final result = await OpenFileSafePlus.open('/path/to/file.pdf');
```

Platform selection happens with a conditional export — standard Dart:

```dart
library open_file_safe_plus;

export 'src/common/open_result.dart';
export 'src/platform/open_file_safe_plus.dart'
    if (dart.library.html) 'src/web/open_file_safe_plus.dart';
```

Non-web builds get `src/platform/...`. Web builds get `src/web/...`.

And in `src/web/open_file_safe_plus.dart`, the class was named:

```dart
class OpenFilePlus {          // ← not OpenFileSafePlus
  OpenFilePlus._();
  static Future<OpenResult> open(String? filePath, {...}) async { ... }
}
```

**`OpenFilePlus`.** Missing the `Safe`. A leftover from when the fork was renamed.

Sit with what that means.

On Android, iOS, macOS, Windows, Linux — the conditional export resolves to `src/platform/...`, which defines `OpenFileSafePlus`, and everything works. Every test passes. Every example runs. The package looks perfectly healthy.

The moment anyone targets **web**, the conditional export resolves to the *other* file. And that file does not define `OpenFileSafePlus`. It defines a class with a different name.

So `flutter build web` fails. Not with a subtle runtime bug — with a **compile error**, on the package's own public API, for **every single consumer targeting web**. The package simply did not work on web, at all, and had not for some time.

The fix is a rename:

```diff
-class OpenFilePlus {
-  OpenFilePlus._();
+class OpenFileSafePlus {
+  OpenFileSafePlus._();
```

Two lines. Months of a completely broken platform.

**Conditional exports are a hole in your type checker.** Dart only ever compiles *one* branch of that `if`. When you build for Android, the web file is not analysed, not type-checked, not even parsed for symbol resolution — it may as well not exist. So the two branches can drift arbitrarily far apart, and your entire CI can be green while one of them is nonsense.

If your package has a conditional export, **you must build every branch of it in CI**, or you don't know that they agree. There is no other way to find out. `dart analyze` on your dev machine will not tell you. It cannot.

---

## Part 3: While I'm Here, `dart:html` Is Dying

The same file had a second problem: it used `dart:html`, which is deprecated and does not work under Dart's WASM compilation target. Web plugins have to move to `dart:js_interop`.

The old code was pleasant, because `dart:html` gave you a typed, Future-returning API:

```dart
// ignore: avoid_web_libraries_in_flutter
import 'dart:html';

Future<bool> open(String uri) async {
  return window
      .resolveLocalFileSystemUrl(uri)
      .then((_) => true)
      .catchError((e) => false);
}
```

`dart:js_interop` gives you none of that. It gives you a way to declare that a JS function exists, and then you're on your own:

```dart
import 'dart:js_interop';

@JS('window.resolveLocalFileSystemURL')
external void _resolveLocalFileSystemUrl(
  String url,
  JSFunction successCallback,
  JSFunction errorCallback,
);

Future<bool> open(String uri) {
  final completer = Completer<bool>();
  _resolveLocalFileSystemUrl(
    uri,
    ((JSAny entry) => completer.complete(true)).toJS,
    ((JSAny error) => completer.complete(false)).toJS,
  );
  return completer.future;
}
```

Three things changed, and each is a general lesson for this migration:

**The real browser API is callback-style, not Future-style.** `dart:html` was *wrapping* it for you. Strip that wrapper away and you're back to success/error callbacks, so you bridge to a `Future` yourself with a `Completer`. Expect to write a lot of `Completer`s.

**`.toJS` is mandatory.** A Dart closure is not a JS function. `((JSAny entry) => ...).toJS` converts it into a real `JSFunction` that can cross the interop boundary. Forget it and you get an error that will not obviously tell you this is the problem.

**The name in `@JS()` is the *browser's* name.** Look closely: `resolveLocalFileSystemURL` — capital `URL`. The `dart:html` binding spelled it `resolveLocalFileSystemUrl`, in Dart's camelCase house style. `dart:js_interop` doesn't rename anything for you; you are naming the actual property on `window`, and if you carry over Dart's spelling you'll bind to `undefined` and fail silently at runtime.

This also forces the SDK floor up, because `external` + `@JS` + `.toJS` need Dart 3.3:

```diff
- sdk: '>=2.17.0 <4.0.0'
+ sdk: '>=3.3.0 <4.0.0'
```

---

## A Small Note on Maintaining a Fork

One last thing that made me smile.

The Android package in this plugin is still `com.crazecoder.openfile`. The FileProvider authority is still `${applicationId}.fileProvider.com.crazecoder.openfile`. The Dart API was renamed to `OpenFileSafePlus`, the pub package is `open_file_safe_plus`, but the *native Android namespace still carries the original author's name*, years later.

That's not sloppiness — it's **deliberate**. Renaming an Android package in a published plugin changes the FileProvider authority, which is declared in the consuming app's merged manifest. Change it and you break file-opening for every app that upgrades. The cost of tidiness is a broken release for your users, so the original author's name lives on in the namespace forever.

Forks inherit their parent's decisions, and some of them are load-bearing.

The Android side of this plugin is, incidentally, a nice fossil record of platform churn: scoped storage forcing `READ_EXTERNAL_STORAGE` to be capped at `maxSdkVersion="32"` and replaced with granular `READ_MEDIA_IMAGES`/`VIDEO`/`AUDIO`; AGP 8 making `namespace` mandatory and needing a `hasProperty` guard so older AGP still builds; `minSdkVersion` dragged from 16 to 19 by a Flutter engine bump. Maintaining a plugin means chasing the platform forever, and never quite catching it.

---

## What I'd Tell You

**1. SPM cares where your files are; CocoaPods doesn't.** The migration is mostly `git mv` into `<plugin>/Sources/<plugin>/`, with public headers nested again under `include/<plugin>/`. If your Objective-C had to change, you've done something more than a migration.

**2. Keep the podspec working.** Repoint its globs at the new SPM tree so both toolchains resolve the same physical files. Flutter's SPM transition is not over, and an SPM-only plugin breaks every consumer still on CocoaPods.

**3. Conditional exports are unchecked code.** Dart compiles exactly one branch. The others are not analysed at all, so they can rot for months while CI is green. **Build every branch in CI** — `flutter build web` *and* `flutter build apk` — or you are shipping code nobody has ever compiled.

**4. `dart:html` → `dart:js_interop` is a real port, not a find-and-replace.** You lose the typed Future-returning wrappers and get raw callbacks. Budget for `Completer`s, remember `.toJS`, and use the **browser's** exact property name in `@JS()`, not Dart's camelCase version of it.

**5. Run the example app.** "It compiles" is not "it works." A plugin migration verified only by a green build is a guess.

---

## The Point

I set out to add Swift Package Manager support, which is a build-system chore, and I came away having fixed a bug that made the package **unusable on an entire platform** — a bug that had been sitting there in plain sight, in a file that no compiler on my machine had looked at in months.

The SPM migration didn't find that bug. Running the analyzer over *every* file, including the one my own platform never compiles, is what found it.

Your CI is only checking the branches it builds. Go count how many branches you have.

---

**Tags:** #Flutter #Dart #SwiftPackageManager #iOS #CocoaPods #JSInterop #OpenSource
