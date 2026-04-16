# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Cucumberish is a BDD test automation framework for iOS/macOS/tvOS that brings Gherkin/Cucumber-style testing to native XCTest — no Ruby, no CLI tools, no external dependencies. Tests are written as `.feature` files and step definitions in Objective-C or Swift, and they run like regular Xcode unit tests.

## Build & Test Commands

```bash
# Build via Swift Package Manager
swift build

# Build via Xcode
xcodebuild -project CucumberishLibraryTest/CucumberishLibrary.xcodeproj -scheme Cucumberish

# Run all tests
xcodebuild -project CucumberishLibraryTest/CucumberishLibrary.xcodeproj test

# In Xcode: CMD+U runs all tests
```

The test project has three targets:
- `Cucumberish` — the library itself
- `CucumberishTests` — unit tests for the library
- `CucumberishFeatureDefinition` — BDD tests demonstrating framework usage with `.feature` files

## Architecture

The framework flow: **parse `.feature` files → dynamically create XCTest classes → match steps to registered definitions → execute**

### Core Components

**`Cucumberish.m/h`** — Singleton main entry point. Orchestrates feature parsing, dynamic XCTest class creation (one class per feature, one test case per scenario), tag filtering, and lifecycle hooks. Users call `[Cucumberish executeFeaturesInDirectory:...]` at the end of `+load` or `setUp`.

**`CCIFeaturesManager`** (`Core/Managers/`) — Parses `.feature` files from the app bundle, applies `includeTags`/`excludeTags` filters, and returns `CCIFeature` model objects.

**`CCIStepsManager`** (`Core/Managers/`) — Singleton step definition registry. Stores step definitions as regex patterns in LIFO order. Matches step text against patterns and executes the associated block.

**`CCIBlockDefinitions.h`** — The public API. Defines all C functions used in step definitions and hooks: `Given()`, `When()`, `Then()`, `And()`, `But()`, `MatchAll()`, `Match()`, `step()`, `before()`, `after()`, `beforeTagged()`, `afterTagged()`, `around()`, `aroundTagged()`, `beforeStart()`, `afterFinish()`, `CCIAssert()`, etc.

**`Core/Models/`** — Data model objects: `CCIFeature`, `CCIScenarioDefinition`, `CCIStep`, `CCIExample`, `CCIBackground`, `CCIArgument`, `CCILocation`, `CCIStepDefinition`, `CCIHock`, `CCIAroundHock`, `CCIJSONDumper`.

**`Dependencies/Gherkin/`** — Gherkin parser with multi-language support via `gherkin-languages.json`.

### Step Definition Blocks

Step body blocks receive:
- `args` — array of regex capture groups from the step text
- `userInfo` — dictionary with keys `kDataTableKey`, `kDocStringKey`, `kXCTestCaseKey`

Step matching is **LIFO** — last registered definition wins when multiple patterns match.

### Hook Execution Order

- `before()` hooks run **FIFO** before each scenario
- `after()` hooks run **LIFO** after each scenario (always run, even on failure)
- `around()` hooks run **FIFO**, nesting scenarios inside

### Dry Run

Setting `cucumberish.dryRun = YES` scans features without executing steps, reporting undefined steps. Useful for validating step coverage in CI.

## Package Manager Support

| Manager | Config |
|---------|--------|
| Swift Package Manager | `Package.swift` (Swift 5.6+, iOS 9+, macOS 10.10+, tvOS 9+) |
| CocoaPods | `Cucumberish.podspec` |
| Carthage | Via git checkout |

The SPM target includes public headers in `include/` and header search paths for `Core/Managers`, `Core/Models`, `Dependencies/Gherkin`, and `Utils`.

## Known Quirks

- Running a single scenario from the Xcode test navigator may cause other tests to disappear; use `fixMissingLastScenario = YES` or run all tests with CMD+U.
- `prettyNamesAllowed = YES` (allows spaces in test names) conflicts with XCTool.
- JSON results export via `CCIJSONDumper` is available for CI/CD integration.
