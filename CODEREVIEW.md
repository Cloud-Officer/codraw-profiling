# Code Review: codraw/profiling

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **H1** — Moved `codraw/tester` from `require-dev` to `require` (`^0.39`) in `composer.json`: the shipped class `Sql/SqlAssertionBuilder.php` hard-imports `Draw\Component\Tester\DataTester` with no guard, so the dependency is a genuine runtime requirement.
- **H2** — Added a `require` section with `"php": ">=8.5"` to `composer.json`, matching the constraint style used by the other codraw packages.
- **L5 (partial)** — `Sql/SqlAssertionBuilder.php`: added a `default:` branch to the `switch` in `__invoke()` that throws a `\LogicException` on an unknown assertion method, making the dispatch fail closed. Also removed the stray `"laravel"` keyword from `composer.json`. (The single-assertion-slot design itself is M3 and was left untouched.)
- `composer validate --no-check-publish` passes and `php -l` is clean on the modified PHP file.

### Validation pass (2026-07-20)

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts` resolves cleanly with the new `require` section (`php: ^8.5`, `codraw/tester: ^0.39`); no constraint adjustment was needed.
- Full PHPUnit run: 27 tests, 48 assertions, 0 failures/errors. The 6 PHPUnit notices ("no expectations were configured for the mock object" in `Tests/ProfilerCoordinatorTest.php` and `Tests/Sql/SqlProfilerTest.php`) are pre-existing PHPUnit 12 style notices — verified identical with the fixes stashed.
- PHPStan level 5 (`phpstan.dist.neon`): no errors; the baseline remains empty, so no stale entries to prune.
- markdownlint-cli2: clean (only auto-fixed trailing whitespace in this file). No test-expectation updates or code repairs were required — no existing test pinned the old silent-no-op `switch` behavior, so the new `LogicException` default branch caused no fallout.
- No additional findings were fixed in this pass: the remaining items (M1, M2, M3, L1) correspond exactly to the scenarios the coverage section below lists as untested, so none met the bar of being directly exercised by existing unit tests.

Not fixed (deliberately, to avoid behavioral changes for consumers): M1 (metric-builder reset changes what repeated sessions report), M2 (lifecycle guards would throw where code currently runs), M3 (assertion-list rework changes builder API/behavior), L1 (adding parameter types could TypeError existing callers), L2 (typing the interfaces breaks existing implementors), L3 (typing/readonly on public properties could break consumers that mutate them), L4 (`exclude-from-classmap` could break a consumer autoloading a `Tests\` class).

Reviewed: all PHP sources at package root and `Sql/`, `composer.json`, PHPStan/PHPUnit configuration, and the `Tests/` suite. The package is a very small profiling component (7 source classes/interfaces, ~200 LOC) providing a profiler coordinator plus SQL query-count metrics and PHPUnit assertion helpers.

## Overall assessment

This is a compact, well-factored component with a clear separation between profiler lifecycle (`ProfilerCoordinator`, `ProfilerInterface`), metric collection (`SqlMetricBuilder`, `SqlLog`, `SqlMetric`) and test assertions (`SqlAssertionBuilder`). Test coverage is proportionally excellent and PHPStan runs at level 5 with an empty baseline. The notable problems are all packaging/contract-level rather than algorithmic: `composer.json` declares **no `require` section at all** (no PHP version constraint, and `codraw/tester` — used by shipped production code — is only in `require-dev`), the base `ProfilerInterface` is untyped, and the SQL profiler's metric builder is never reset between profiling sessions, which makes repeated start/stop cycles accumulate stale queries. No security-sensitive surface exists in this package (no I/O, no deserialization, no user input handling).

## Findings

### High

#### **[FIXED]** H1. Production class depends on `codraw/tester`, which is only a dev dependency

`Sql/SqlAssertionBuilder.php:5` imports `Draw\Component\Tester\DataTester` and the class type-hints it (`__invoke(DataTester $tester)` at line 42). `composer.json` lists `codraw/tester` only under `require-dev` (line 19). Any consumer that installs `codraw/profiling` and uses `SqlAssertionBuilder` (a class shipped in the package's autoloaded namespace) without also depending on `codraw/tester` gets a fatal "class not found" error. Even accepting that this class is intended for test contexts, the dependency contract is wrong: it should either be in `require`, or the package should declare it in `suggest` and document the requirement. Composer cannot detect or warn about this today.

#### **[FIXED]** H2. `composer.json` has no `require` section — not even a PHP version constraint

`composer.json` (lines 18–21) contains only `require-dev`. The code uses `final public const` (`Sql/SqlProfiler.php:9`, PHP >= 8.1), typed properties and arrow functions (PHP >= 7.4). Without `"php": ">=8.1"` (or the framework's actual floor, likely `^8.2`) the package is installable on any PHP version and will produce parse errors at runtime on older interpreters. Every dependency the package actually needs must be declared for the resolver to protect consumers.

### Medium

#### M1. `SqlProfiler` never resets its metric builder — metrics leak across profiling sessions

`Sql/SqlProfiler.php:13-25`: `getMetricBuilder()` lazily creates a single `SqlMetricBuilder` and `stop()` builds a metric from it, but nothing ever clears `SqlMetricBuilder::$logs` (`Sql/SqlMetricBuilder.php:12-17` has no reset method). `ProfilerCoordinator` explicitly supports repeated `startAll()`/`stopAll()` cycles (`isStarted()` toggles back and forth), so a second session on the same profiler instance will report the first session's queries in its counts unless every concrete subclass remembers to null out `$this->metricBuilder` in `start()`. That contract is implicit and unenforced; the base class should reset the builder in `stop()` (or `start()`), or `SqlMetricBuilder` should expose a `reset()` invoked by the base class.

#### M2. `ProfilerCoordinator` lifecycle is not enforced against its own state

`ProfilerCoordinator.php:26-47`: `startAll()` can be called while already started (double-starting every profiler), `stopAll()` can be called when never started (calling `stop()` on profilers that were never started — with M1 this returns stale/empty metrics rather than failing), and `registerProfiler()` (line 49) accepts new profilers after `startAll()` without starting them, so the following `stopAll()` calls `stop()` on a profiler that never ran. Since the class tracks `$started` anyway, guarding these transitions (or at least documenting the contract) would prevent silent bad data. Additionally, `stopAll()` has only a PHPDoc return type (line 34-37) instead of a native `: \stdClass` return type.

#### M3. `SqlAssertionBuilder` holds a single assertion slot and its `assert*` naming is misleading

`Sql/SqlAssertionBuilder.php:27-40`: the three `assertCount*` methods each overwrite `$countAssertion`, so configuring both a lower and an upper bound (a natural use: "between 2 and 5 queries") silently keeps only the last one. The `assert*` prefix also suggests an immediate assertion when it is really deferred configuration; a silent overwrite plus that naming makes it easy to believe two constraints are being checked when only one is. Storing a list of assertions (and applying all of them in `__invoke()`) would fix this cheaply. Also, the setters return `void` while `create()` returns `self`, so the builder is not chainable — an odd half-fluent API.

### Low

#### L1. Inconsistent/missing parameter types on assertion setters

`Sql/SqlAssertionBuilder.php:27` and `:32`: `assertCountGreaterThanOrEqual($count)` and `assertCountLessThanOrEqual($count)` take an untyped parameter while `assertCountEquals(int $count)` (line 37) is typed. The property PHPDoc (`array{0: string, 1: int}`, lines 9-12) is therefore not guaranteed. Passing a string count would flow into PHPUnit comparisons with loose semantics.

#### L2. `ProfilerInterface` and `MetricBuilderInterface` are fully untyped

`ProfilerInterface.php:7-11` and `MetricBuilderInterface.php:7` declare `start()`, `stop()`, `getType()`, `build()` with no return types and no PHPDoc. `stop()`'s return value is what `ProfilerCoordinator::stopAll()` aggregates into the metrics object, so the most important contract in the package is undocumented (`getType(): string` and `stop(): mixed`/an interface for metrics would at least be checkable).

#### L3. Legacy untyped public properties in value objects

`Sql/SqlLog.php:7` (`public $query`) and `Sql/SqlMetric.php:10,15` (`public $count`, `public $queries`) are untyped and mutable despite the constructors enforcing types. Native typed (or `readonly`) properties would prevent external mutation from desynchronizing `count` from `queries` (nothing stops `$metric->queries[] = '...'` without updating `$metric->count`).

#### L4. `Tests/` not excluded from production autoload/classmap

`composer.json:24-28` maps the package root to `Draw\Component\Profiling\`, which also makes `Draw\Component\Profiling\Tests\*` autoloadable in consumer installs. Symfony-style root-mapped components normally add `"exclude-from-classmap": ["/Tests/"]` (and `.gitattributes` export-ignore) so test classes referencing PHPUnit never leak into production autoloading.

#### **[FIXED]** L5. Silent no-op on unknown assertion method

`Sql/SqlAssertionBuilder.php:60-70`: the `switch` on the stored method name has no `default` branch, so an unrecognized value would make `__invoke()` pass without asserting anything. All current writers are internal, but a `default: throw new \LogicException(...)` (or a `match`) would make the dispatch fail closed. The `keywords` entry `"laravel"` in `composer.json:8` also looks like a copy-paste artifact for a Draw/Symfony-ecosystem package.

## Strengths

- Clean, minimal design with good separation of concerns: coordinator vs. profiler vs. metric builder vs. metric vs. assertion helper; the SQL profiler is properly `abstract`, leaving transport-specific wiring (e.g. Doctrine) to other packages.
- Proportionally excellent test suite: every class has a dedicated test, `SqlAssertionBuilder` is exercised through data providers covering pass and fail paths for all three comparison modes, both `__invoke()` branches (nested `sql` path and direct metric), the failure-message format (including the query dump), and the "no assertion configured" exception.
- Static analysis hygiene: PHPStan level 5 with an **empty** baseline (`phpstan-baseline.neon`), so no suppressed debt.
- No security surface: no user input, filesystem, network, serialization, or query execution — the package only observes queries as strings; the failure message embedding SQL text is confined to test output.
- Modern PHPUnit 11/12 setup with attributes-based data providers and correct `source` include/exclude configuration in `phpunit.xml.dist`.

## Test coverage

Coverage is qualitatively strong for a package this size — all seven source files have corresponding tests:

- **Well covered:** `ProfilerCoordinator` lifecycle (default state, start, stop, registration, metric aggregation keyed by type); `SqlAssertionBuilder` (all three assertion modes with positive/negative cases, both invocation branches, failure-message content, missing-assertion exception); `SqlMetricBuilder`, `SqlMetric`, `SqlLog` constructors and `build()`; `SqlProfiler::getType()`, `getMetricBuilder()` lazy init, and `stop()` on an empty builder.
- **Not covered:** repeated start/stop cycles (which would expose finding M1's metric accumulation); registering a profiler after `startAll()` (M2); double-start/unstarted-stop behavior; non-int counts passed to the untyped setters (L1); overwriting one assertion with another (M3).

The uncovered scenarios track exactly the behavioral findings above — adding a two-cycle profiling test would have caught the builder-reset gap.

Overall grade: **B** — good, clean code with real but fixable packaging and lifecycle-contract issues.
