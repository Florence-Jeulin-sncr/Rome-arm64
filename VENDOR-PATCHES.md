# Vendored dependency patches

`vendor/` holds two upstream Hackage packages, unmodified except where noted
below. Both are MPL-2.0 and keep their original `LICENSE` files.

They are vendored because Rome pins `amazonka` 1.6.1 (2018), which does not
compile under GHC 9.2 — and GHC 9.2 is the *minimum* for a native arm64 macOS
binary:

* GHC 8.8 (upstream Rome's `lts-15.11`) has no `aarch64-darwin` target at all,
  which is why every upstream Rome release is x86_64-only.
* GHC 8.10.7 added `aarch64-darwin` but only via the LLVM backend, needing
  llvm 9-12 — no longer packaged by Homebrew.
* GHC 9.2 introduced the native AArch64 code generator. It ships base-4.16,
  which removed `Data.Semigroup.Option`, so aeson 1.x cannot be built there
  either. aeson 2.x is therefore mandatory, and that is what amazonka 1.6.1
  predates.

## `vendor/amazonka-core-1.6.1`

* `src/Network/AWS/Data/JSON.hs` — aeson 2 changed `Object` from
  `HashMap Text Value` to `KeyMap Value`; `(.:>)` and `(.?>)` now look up via
  `Data.Aeson.KeyMap` with a `Key`.
* `src/Network/AWS/Data/Map.hs` — the `Object` `IsList` items are `(Key, Value)`
  under aeson 2; convert with `Key.toText` / `Key.fromText`.
* `src/Network/AWS/Data/Body.hs` — `ToHashedBody (HashMap Text Value)` converts
  through `KeyMap.fromHashMapText` before applying the `Object` constructor.
* `src/Network/AWS/Data/ByteString.hs`, `src/Network/AWS/Data/Log.hs` —
  `Data.ByteString.Lazy.Builder` was removed in bytestring 0.11; use
  `Data.ByteString.Builder`.

## `vendor/amazonka-1.6.1`

* `src/Network/AWS/Internal/Logger.hs` — same bytestring module rename.
* `src/Control/Monad/Trans/AWS.hs` — unliftio-core 0.2 dropped `askUnliftIO`
  as a class method; the `MonadUnliftIO` instance is written with
  `withRunInIO` instead.

## Rebuilding

    stack build                              # arm64, native
    stack install --local-bin-path ./dist

For a universal binary, build the x86_64 slice with an x86_64 Stack under
Rosetta (`arch -x86_64 <x86_64-stack> install --local-bin-path ./dist-x86_64`),
then:

    lipo -create dist-x86_64/rome dist/rome -output rome
    codesign --force --sign - rome          # lipo invalidates the signature

`pkg-config` must be installed (`brew install pkg-config`) — the `digest`
dependency requires it.
