# GHC 10.0 breakage inventory

A catalogue of all the non-trivial fixes (that is, not just bounds
bumps) required to upgrade Haskell code that worked with GHC 9.14 to
GHC 10.0.  The [GHC 10.0 migration
guide](https://gitlab.haskell.org/ghc/ghc/-/wikis/migration/10.0)
contains a list of changes that the GHC team anticipated ahead of time
were capable of causing breakage whereas this breakage inventory is a
report of breakage that actually happened in practice.

**GHC 10.0 has not been released yet.**  Entries marked *(anticipated)*
have not yet been seen to break a real package, so they have no
examples.  They come from the [GHC 10.0.1 release
notes](https://gitlab.haskell.org/ghc/ghc/-/blob/ghc-10.0/docs/users_guide/10.0.1-notes.rst),
the library changelogs, and a diff of the export lists that GHC's
`interface-stability` tests record for `base` and `template-haskell`
between the `ghc-9.14` and `ghc-10.0` branches.  Unless an entry says
otherwise, it was reproduced with a small program on the pre-release
GHC 10.0.0.20261109, with GHC 9.14.1 as the control.  As real breakage
turns up, entries should gain examples and lose the mark, and entries
that never cause trouble should be removed.

## `base`

### New exports clash with existing names

Several modules export new identifiers, which clash with other
definitions of the same name when the module is imported unqualified,
as `foldl'` did in GHC 9.10 and `Data.Functor.unzip` did in GHC 9.8.

* `Data.List` and `Data.List.NonEmpty` export `nubOrd` and `nubOrdBy`.
  These clash with `Data.Containers.ListUtils.nubOrd` from
  `containers`.  `extra` before version 1.8.1 fails to compile, because
  `Data.List.Extra` imports `Data.List` and defines its own `nubOrd`
  and `nubOrdBy`.
* `Data.List.NonEmpty` exports `mapMaybe`, which clashes with
  `Data.Maybe.mapMaybe` if both are imported unqualified.
* `Control.Applicative` exports `thenA` and `Control.Monad` exports
  `thenM`.
* `Control.Concurrent` exports `labelThread`.

#### Examples

[`extra`](https://github.com/ndmitchell/extra/pull/118) (`nubOrd` and
`nubOrdBy`)

#### References

* [CLC proposal #336
  (`nubOrd`)](https://github.com/haskell/core-libraries-committee/issues/336)
* [CLC proposal #337
  (`mapMaybe`)](https://github.com/haskell/core-libraries-committee/issues/337)
* [CLC proposal #351 (`thenA`,
  `thenM`)](https://github.com/haskell/core-libraries-committee/issues/351)
* [CLC proposal #376
  (`labelThread`)](https://github.com/haskell/core-libraries-committee/issues/376)

### `init` and `last` gain a warning (anticipated)

Use of `init` and `last`, whether imported from `Prelude` or from
`Data.List`, now incurs an `x-partial` warning, as `head` and `tail`
did in GHC 9.8.  Successful compilation
with `-Werror` requires either avoiding them or some form of
conditional compilation to suppress the warning.

#### References

* [CLC proposal
  #292](https://github.com/haskell/core-libraries-committee/issues/292)

### `GHC.Num.BigNat`, `GHC.Num.Integer` and `GHC.Num.Natural` removed (anticipated)

These modules are no longer exposed by `base`; they must be imported
from `ghc-bignum` instead.  This reverses GHC 9.14, which added them to
`base` (`base-4.22`), and whose `ghc-bignum-1.4` changelog advised
importing them from `base`.  Code that followed that advice will fail
with an error saying the module belongs to the hidden package
`ghc-bignum`.

#### References

* [CLC proposal
  #359](https://github.com/haskell/core-libraries-committee/issues/359)

### Big number internals in `GHC.Num` deprecated (anticipated)

`GHC.Num` still exports the internals of `Integer` and `Natural`
(`Integer(IS, IP, IN)`, `Natural(NS, NB)`, `integerToInt#`,
`integerAdd` and similar), but they are now deprecated, with the
message "The internals of big numbers will be removed in 4.24.  Use
ghc-bignum instead."  This breaks compilation with `-Werror`.

#### References

* [CLC proposal
  #360](https://github.com/haskell/core-libraries-committee/issues/360)

### `GHC.Desugar` and `GHC.JS.Prim.Internal.Build` removed (anticipated)

`GHC.JS.Prim.Internal.Build` exists only with the JavaScript backend,
so its removal was not reproduced.

#### References

* [CLC proposal
  #216](https://github.com/haskell/core-libraries-committee/issues/216)
  (`GHC.Desugar`)
* [CLC proposal
  #329](https://github.com/haskell/core-libraries-committee/issues/329)
  (`GHC.JS.Prim.Internal.Build`)

### `Semigroup` and `Monoid` instances for lazy `ST` (anticipated)

`Control.Monad.ST.Lazy.ST` now has `Semigroup` and `Monoid` instances,
which conflict with orphan instances defined elsewhere.

#### References

* [CLC proposal
  #374](https://github.com/haskell/core-libraries-committee/issues/374)

### Changes in run-time behaviour (anticipated)

These changes do not cause compilation errors but can change what a
program does.

* The `Bifunctor` instances for tuples are stricter, matching their
  `Functor` instances ([CLC proposal
  #339](https://github.com/haskell/core-libraries-committee/issues/339)).
* `Data.List.iterate'` forces each element to WHNF when the `(:)`
  containing it is forced ([CLC proposal
  #335](https://github.com/haskell/core-libraries-committee/issues/335)).
* `Data.List.sortOn` uses `(>)` instead of `compare`, which can change
  the result for types whose `Ord` instance is not lawful, such as
  `Double` with `NaN` ([CLC proposal
  #332](https://github.com/haskell/core-libraries-committee/issues/332)).
* `hIsReadable` and `hIsWritable` throw an exception for every closed
  or semi-closed handle, not only for file handles ([CLC proposal
  #371](https://github.com/haskell/core-libraries-committee/issues/371)).

## GHC

### Default language edition is `GHC2024` (anticipated)

The default language edition changed from `GHC2021` to `GHC2024`.  The
change most likely to break code is `MonoLocalBinds`, under which local
bindings without type signatures are no longer always generalised.

This affects only modules compiled without an explicit language: plain
`ghc`, `ghci` and `runghc` invocations and scripts.  Cabal passes
`-XHaskell98` to GHC when a component has no `default-language`, so
packages built by Cabal or Stack are not affected.

#### References

* [GHC proposal
  632](https://github.com/ghc-proposals/ghc-proposals/pull/632)
* [GHC issue #26039](https://gitlab.haskell.org/ghc/ghc/-/issues/26039)

### Infix promoted data constructors require `DataKinds` (anticipated)

GHC 9.10 to 9.14 erroneously accepted infix uses of promoted data
constructors, such as ``x ': xs``, without `DataKinds`.

### Multiplicity variables need kind annotations (anticipated)

Under `LinearTypes`, a multiplicity-polymorphic arrow needs its
multiplicity variable bound with a kind annotation.

```.hs
f :: Int %m -> Int                               -- no longer accepted
f :: forall (m :: Multiplicity). Int %m -> Int   -- still accepted
```

`NoModifiers` makes GHC accept the first form again.

Under `Strict`, `let %1 x:xs = ...` now parses as
`let (%1 x):xs = ...`, and GHC warns about an unrecognised modifier
(`-Wunrecognised-modifiers`).  Writing `let %1 (x:xs) = ...` avoids
the warning.  `linear-base` is a likely candidate for breakage.

### Free variables of `static` forms must be top-level (anticipated)

The free term variables of `static e` must now be bound at top level.

#### References

* [GHC proposal
  732](https://github.com/ghc-proposals/ghc-proposals/pull/732)
* [GHC issue #26556](https://gitlab.haskell.org/ghc/ghc/-/issues/26556)

### Namespace specifiers in subordinate export items are checked (anticipated)

An export item such as `module M (T (type A))` used to have its
namespace specifier ignored, and is now rejected.

#### References

* [GHC issue #12488](https://gitlab.haskell.org/ghc/ghc/-/issues/12488)

### `-fbyte-code` no longer produces object code (anticipated)

`-fbyte-code` now selects the bytecode backend unconditionally, so a
module with `{-# OPTIONS_GHC -fbyte-code #-}` produces no object code
and cannot be linked.  The migration guide mentions `lens` as a
library that used this pragma.

### Compiler output changed (anticipated)

These changes break golden tests and tools that parse GHC's output.

* `-v1` now implies `-fhide-source-paths`, so `Compiling` lines no
  longer show source and object file paths ([GHC issue
  #25345](https://gitlab.haskell.org/ghc/ghc/-/issues/25345)).  The
  release notes say this is the default, but in 10.0.0.20261109 it
  happens only when `-v1` is passed explicitly.  Plain `ghc` and
  `cabal build`, which pass no verbosity flag, still show the paths.

The release notes also list these changes, which were not reproduced:

* The final error `<no location info>: error: Compilation had errors`
  is now `Compilation had errors`.
* External interpreter trace messages go to stderr instead of stdout.
* `ZonkAny` appears in messages as `UnusedType`.

## `template-haskell`

### `Extension` has new constructors (anticipated)

`Language.Haskell.TH.LanguageExtensions.Extension` gained `Modifiers`
and `QualifiedStrings`, so exhaustive matches on it give an
incomplete-patterns warning.

### `Quasi` has a new method (anticipated)

`Quasi` gained `qAddDependentDirectory`, which is part of its `MINIMAL`
pragma.  Hand-written `Quasi` instances get a missing-methods warning.

### `Q` moved to `GHC.Internal.TH.Monad` (anticipated)

`Q`, `Quote` and `Code` moved from `GHC.Internal.TH.Syntax` to
`GHC.Internal.TH.Monad`.  Only code that imports `ghc-internal`
directly is affected.

## GHC API (`ghc`)

### Typechecker and defaulting plugins (anticipated)

`tcPluginStop` was replaced by `tcPluginPostTc` and
`tcPluginShutdown`, and `dePluginStop` by `dePluginPostTc` and
`dePluginShutdown`.  To migrate a plugin whose stop action released a
resource, move that action to `tcPluginShutdown` and set
`tcPluginPostTc = const (return ())`.  This affects every typechecker
plugin.

#### References

* [GHC issue #26839](https://gitlab.haskell.org/ghc/ghc/-/issues/26839)

### Typechecking entry points take an extra argument (anticipated)

`typecheckModule`, `hscTypecheckRename`,
`hscTypecheckRenameWithDiagnostics` and `hscTypecheckAndGetWarnings`
take an additional `TcMPluginHandling` argument that says how to start
and stop typechecker plugins, and `tcRnModule` takes a
`TcRnModuleOptions` record in place of a `Bool`.  `withTcPlugins` and
`withHoleFitPlugins` were replaced by `withTcMPlugins`.

#### References

* [GHC issue #26839](https://gitlab.haskell.org/ghc/ghc/-/issues/26839)

### `RewriterSet` renamed to `CoHoleSet` (anticipated)

For example, `emptyRewriterSet` is now `emptyCoHoleSet`.

### Unique tags are an ADT (anticipated)

Unique tags are now values of `GHC.Types.Unique.UniqueTag`.  Plugins
that used a custom tag character should use `PluginTag`.

### Syntax tree changes (anticipated)

These affect tools that traverse or construct the syntax tree, such as
`ghc-exactprint`, `ormolu`, `fourmolu` and `hlint`.

* The `HsTyLit` data type was replaced by `HsLit`, which gained
  `HsNatural` and `HsDouble` constructors ([GHC issue
  #26862](https://gitlab.haskell.org/ghc/ghc/-/issues/26862)).
* `HsMultilineString` was merged into `HsString` ([GHC issue
  #26860](https://gitlab.haskell.org/ghc/ghc/-/issues/26860)).
* `OverlapMode` and `OverlapFlag` are no longer exported from
  `GHC.Types.Basic`.  `OverlapMode` moved to
  `Language.Haskell.Syntax.Decls.Overlap` and gained an extension
  point, and `OverlapFlag` moved to `GHC.Hs.Decls.Overlap`.  Both are
  still exported from `GHC.Core.InstEnv`.  `TopLevelFlag` and
  `RuleName` moved to `Language.Haskell.Syntax.Basic`, but
  `GHC.Types.Basic` still exports them.
* The `ForeignCall` types that need no extension points moved to
  `Language.Haskell.Syntax.Decls.Foreign`.  `CCallTarget`, `CType` and
  `Header` gained extension points, and their `Bool` fields were
  replaced by specific data types.

### Other removals (anticipated)

* The backwards-compatibility pattern synonym `ModLocation` was
  removed ([GHC issue
  #24932](https://gitlab.haskell.org/ghc/ghc/-/issues/24932)).
* `SourceError` now stores the context it needs to print its
  diagnostics ([GHC issue
  #26387](https://gitlab.haskell.org/ghc/ghc/-/issues/26387)).
* The `Data` instance for `ClsInst` was removed.
* `preloadClosure` was removed from `UnitState`.
* `interpreterBackend` was renamed `bytecodeBackend`; the old name is a
  deprecated alias.

## `Cabal`

GHC 10.0 ships `Cabal-3.18.1.0` (GHC 9.14.1 shipped `Cabal-3.16.0.0`).

### `Verbosity` split into `Verbosity` and `VerbosityFlags` (anticipated)

The old `Verbosity` type is now `VerbosityFlags`, and the new
`Verbosity` adds the handles used for logging.  Flags such as
`buildVerbosity` hold a `VerbosityFlags`, which must be converted with
`mkVerbosity defaultVerbosityHandles`.  The `Ord`, `Enum` and `Bounded`
instances of `VerbosityFlags` and the function `modifyVerbosity` were
removed.  This affects custom `Setup.hs` scripts and `SetupHooks`.

### Removed modules (anticipated)

`Distribution.Make` (and `build-type: Make` for packages that declare
`cabal-version: 3.18` or later), `Distribution.Compat.MonadFail`,
`Distribution.Compat.Directory` and `Distribution.Compat.FilePath`
were removed.

#### References

* [`Cabal` 3.18.1.0 release
  notes](https://github.com/haskell/cabal/blob/Cabal-v3.18.1.0/release-notes/Cabal-3.18.1.0.md)

## Distribution

### `ghc-iserv` is no longer shipped (anticipated)

GHC builds the external interpreter on demand instead of shipping
`iserv`.  Build setups that pass `-pgmi` the path to the shipped
`ghc-iserv` binary, for example for cross-compilation, will break.

#### References

* [GHC issue #24731](https://gitlab.haskell.org/ghc/ghc/-/issues/24731)

### Settings moved to `lib/targets/default.target` (anticipated)

GHC reads target settings from `lib/targets/default.target`, and the
redundant entries were removed from `lib/settings`.  Tools that read or
patch `lib/settings` directly need to change.

#### References

* [GHC issue #24212](https://gitlab.haskell.org/ghc/ghc/-/issues/24212)

### Fewer Linux bindists (anticipated)

There are no longer bindists for Debian 9 and 10, Ubuntu 18.04 and
20.04, or Fedora 33 and 38.  There are new ones for Debian 13 and
Fedora 43.

#### References

* [GHC issue #25876](https://gitlab.haskell.org/ghc/ghc/-/issues/25876)
