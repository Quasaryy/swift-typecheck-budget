# typecheck-budget

One command that shows which Swift files in your project are closest to

```
the compiler is unable to type-check this expression in reasonable time
```

before the build actually breaks.

The error is not about time. Since Swift 6.2 the type checker gives up on deterministic limits,
and in SwiftUI code the one that usually runs out first is the constraint solver's memory: 512 MB
per expression in Swift 6.4. Xcode never tells you how close a file is to that limit. This tool builds
your project once with the compiler's own statistics turned on and ranks every file by how much
of that budget it used.

```
typecheck-budget 0.1.0 · Swift 6.4.2 · solver memory limit 512 MB (compiler default) · 3 files

  #  risk    solver peak  of limit      scopes  slowest body  file
  1  HIGH       424.6 MB     79.1%     611,314   8,065 ms :7  Sources/App/SettingsView.swift
  2  ok           0.6 MB      0.1%       4,703     8 ms :218  Sources/App/Features.swift
  3  ok           0.0 MB      0.0%         505       7 ms :7  Sources/App/ChildViews.swift

HIGH (>= 25% of the limit): 1 file(s); watch (5-25%): 0 file(s).

Hints:
  Sources/App/SettingsView.swift
    - 6 inline `$store.scope(state:action:)` calls: move them out of the modifier chain ...
    - 6 `.navigationDestination(item:)` calls: each one is an overload choice the solver revisits ...
```

That screen has six `.navigationDestination` modifiers and compiles, but it uses 79% of the
solver's memory, and a seventh destination breaks the build. After the fix, `typecheck-budget file`
shows the same file in seconds:

```
Sources/App/SettingsView.swift  ·  Swift 6.4.2  ·  limit 512 MB
  solver peak          0.3 MB   was 424.6 MB (scan 2026-11-17T15:32:54)
  of limit               0.1%   was 79.1% (scan 2026-11-17T15:32:54)
  scopes                  312   was 611,314 (scan 2026-11-17T15:32:54)
  risk                     ok
         35 ms  line 7  SettingsView.body.getter
```

## Quick start

You need Xcode and Python 3; the `python3` that comes with Xcode or the Command Line Tools is
enough. There is nothing to install: the tool is one script.

```bash
curl -LO https://raw.githubusercontent.com/quasaryy/swift-typecheck-budget/main/typecheck-budget
chmod +x typecheck-budget

cd path/to/YourApp
/path/to/typecheck-budget scan -- \
  -project YourApp.xcodeproj -scheme YourApp \
  -destination 'platform=iOS Simulator,name=iPhone 18 Pro'
```

Everything after `--` goes to `xcodebuild` unchanged: `-workspace` or `-project`, `-scheme`,
`-destination`, `-configuration`, build settings. Use a concrete simulator
(`xcrun simctl list devices available` lists them) and the Debug configuration. Environment
variables your build needs are set as usual, before the command.

The first scan builds the whole project, packages included, into its own DerivedData, so it takes
longer than a normal build. On an app with about 400 Swift files and 40 packages it took about
3.5 minutes on an Apple silicon Mac; a repeated scan recompiles only the project's own files and
took about 2.5 minutes. Add `.typecheck-budget/` to your `.gitignore`.

## Commands

### scan

```bash
typecheck-budget scan [--top 15] [--markdown report.md] [--fail-above PCT] -- <xcodebuild arguments>
```

Builds the project and ranks every Swift file under `--root` (the current folder by default) by
the peak memory of the constraint solver. It also lists the function bodies that took a second or
more to type-check, any expression that already fails with "unable to type-check", and hints for
common SwiftUI and TCA patterns. Each scan is saved as JSON in `.typecheck-budget/`.

### file

```bash
typecheck-budget file Sources/App/SettingsView.swift -- <the same xcodebuild arguments>
```

Re-measures one file after you edit it and compares it with the last scan or the last `file`
run. After the first scan this is an incremental build of exactly one file, so you can try a
refactoring and see whether it actually helped. On a small project it takes a few seconds; on an
app with about 400 Swift files it took about 25 seconds, most of it `xcodebuild`'s own overhead.

### compare

```bash
typecheck-budget compare .typecheck-budget/scan-A.json .typecheck-budget/scan-B.json
```

Shows which files got heavier or lighter between two scans: before and after a refactoring, on
two branches, or after an Xcode update.

### show

```bash
typecheck-budget show .typecheck-budget/latest-scan.json [--markdown]
```

Prints a saved report again, as the table or as Markdown for a pull request comment.

### Common options

| Option | Default | What it does |
| --- | --- | --- |
| `--root DIR` | current folder | Only Swift files under this folder are reported and recompiled. |
| `--work-dir DIR` | `.typecheck-budget` | Where the DerivedData, statistics, build log and reports go. |
| `--top N` | 15 | Rows to print. |
| `--limit-mb MB` | from the Swift version | The solver memory limit, for a compiler the tool does not know. |
| `--no-hints` | | Do not print the SwiftUI / TCA hints. |

## Reading the report

| Column | What it is |
| --- | --- |
| solver peak | `Sema.MaxSolverBytesAllocated`: the peak size of the solver's arena for the heaviest expression in the file. |
| of limit | That peak as a share of the compiler's memory limit: 512 MB in Swift 6.2 and 6.4, 564 MB in Swift 6.3. |
| scopes | `Sema.NumConstraintScopes`: every attempt the solver made while checking the file, a measure of total work. |
| slowest body | The slowest function body in the file and its line, from `-debug-time-function-bodies`. Start looking for the heavy expression there. The first function of every file also pays one-time work such as loading the imported modules, about 0.3-0.5 s, so a few hundred milliseconds there means nothing. |
| risk | `HIGH` from 25% of the limit, `watch` from 5%. |

Why 25% is already high: in the SwiftUI navigation chains this was calibrated on, every
additional modifier multiplied the solver's memory by two or more, and the builds failed while
the counter showed 55 to 97% of the limit. The counter does not include the memory of solutions
the solver has already found, while the limit does. A file at 25% is usually one or two chain
links away from the error. Treat it as a heuristic, not a guarantee.

The numbers are deterministic: the same compiler, dependencies and source give the same counts
from run to run, unlike timings. A new Xcode can change them, which is what `compare` is for.

## Using it in CI

```bash
typecheck-budget scan --fail-above 50 --markdown typecheck-budget.md -- \
  -workspace App.xcworkspace -scheme App -destination 'platform=iOS Simulator,name=iPhone 18 Pro'
```

Exit codes: `0` is fine, `1` means the build failed, `2` means a file reached the `--fail-above`
threshold. The Markdown file can be posted as a pull request comment.

## How it works

`scan` runs `xcodebuild build` with your arguments and adds:

- `SWIFT_ENABLE_BATCH_MODE=NO`, a hidden Xcode setting: every file is compiled by its own compiler
  process, so the statistics are per file instead of per batch of files;
- `OTHER_SWIFT_FLAGS=$(inherited) -Xfrontend -stats-output-dir ... -Xfrontend -debug-time-function-bodies`;
- `COMPILER_INDEX_STORE_ENABLE=NO`, `-skipMacroValidation`, `-skipPackagePluginValidation`;
- `-derivedDataPath .typecheck-budget/DerivedData`, so the build Xcode uses stays untouched.

Before building, `scan` bumps the modification time of every Swift file under `--root` so that
all of them are recompiled, and puts the original times back afterwards. `file` shows its build the
same times that build saw last time and bumps only the file you measure, so exactly one file is
recompiled. In both cases your files end up with their real modification times, so Xcode's own
build does not rebuild anything because of the tool. A file you edit while the tool runs keeps its
new time. Hidden folders such as `.git` or `.build` are skipped.

The compiler writes one JSON file of statistics per compiled file. The tool reads two counters
from it, takes the function timings and the "unable to type-check" errors from the build log, and
matches everything to your source files.

### Why not the usual warning flags

`-warn-long-function-bodies` and `-warn-long-expression-type-checking` measure milliseconds. They
depend on the machine and say nothing about how close the expression is to the error.
`-warn-long-expression-type-checking-scopes` and `-trail` count operations, but per expression,
and the solver resets these counters for every element of a closure body (`ConjunctionStep` in
`lib/Sema/CSStep.cpp`). In SwiftUI code almost everything is a closure, so they report a small
fraction of the real work. The memory the solver allocates is not reset, which is why the tool
ranks by it.

## Limitations

- Tested with Xcode 27.2 (Swift 6.4.2). For a compiler version the tool does not know, it assumes
  512 MB; pass `--limit-mb` to override.
- A scan is slower than a normal build: one compiler process per file, and the first scan also
  builds all packages.
- `solver peak` is about the heaviest expression in the file. To find the expression itself,
  start with the slowest function body and the hints.
- Use the Debug configuration. With whole-module optimization Xcode ignores batch mode settings
  and the statistics are per module.
- The tool sets `OTHER_SWIFT_FLAGS` on the command line with `$(inherited)`, so the flags from
  your project are kept. Do not pass your own `OTHER_SWIFT_FLAGS` after `--`.
- The hints are simple text patterns for SwiftUI and TCA navigation chains. They suggest where to
  look; they do not prove anything.

## Background

- Slava Pestov, [Roadmap for improving the type checker](https://forums.swift.org/t/roadmap-for-improving-the-type-checker/82952), Swift Forums
- Swift 6.4.2 sources: [`LangOptions.h`](https://github.com/swiftlang/swift/blob/swift-6.4.2-RELEASE/include/swift/Basic/LangOptions.h) (the limits), [`ConstraintSystem.cpp`](https://github.com/swiftlang/swift/blob/swift-6.4.2-RELEASE/lib/Sema/ConstraintSystem.cpp) (`isTooComplex`), [`CSStep.cpp`](https://github.com/swiftlang/swift/blob/swift-6.4.2-RELEASE/lib/Sema/CSStep.cpp) (the counter reset)
- The article this tool comes from: link will be added after publication.

## License

MIT, see [LICENSE](LICENSE).
