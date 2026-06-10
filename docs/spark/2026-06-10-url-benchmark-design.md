# URL Benchmark Design

## Context

Issue apache/dubbo-go#3391 asks for focused benchmarks for `common.URL`.
The goal is to provide allocation and throughput visibility for URL hot paths before
later optimization work. This design only covers benchmark structure. It does not
change production behavior.

The local working tree already has `common/url_benchmark_test.go`, but it currently
contains only the Apache license header and `package common`.

## Scope

Add URL benchmarks in `common/url_benchmark_test.go`.

The benchmarks should cover:

- `URL.String()`
- `URL.Key()`
- `URL.GetCacheInvokerMapKey()`
- `URL.ServiceKey()`
- `URL.CopyParams()`
- `URL.GetParams()`
- `URL.Clone()`
- `URL.CloneWithFilter()`
- `URL.MergeURL()`
- `URL.ToMap()`
- `IsEquals()`

The benchmark parameter scale is:

- `1`
- `32`
- `256`
- `1024`

The benchmark should not include a `0` parameter scale.

## Non-Goals

- Do not change `common/url.go` or other production code.
- Do not optimize URL internals in this change.
- Do not add `b.RunParallel` benchmarks.
- Do not add PR description or baseline-output requirements to this design.
- Do not create a Cartesian product of every method, scenario, and parameter scale.

## Benchmark Helpers

Use small helper functions so individual benchmarks stay focused:

- `makeBenchmarkURL(paramCount int) *URL`
- `makeBenchmarkURLWithSubURL(paramCount int) *URL`
- `makeBenchmarkMergePair(paramCount int) (*URL, *URL)`
- `makeBenchmarkExcludeSet(paramCount int) *gxset.HashSet`
- `makeBenchmarkReserveKeys(paramCount int) []string`

The generated data should be stable and deterministic:

- parameter keys: `key0`, `key1`, `key2`, ...
- parameter values: `value0`, `value1`, `value2`, ...
- method names: `method0`, `method1`, ...

The base URL fields should remain fixed across benchmarks:

- protocol: `dubbo`
- IP: `127.0.0.1`
- port: `20000`
- path: `/com.test.BenchmarkService`
- interface: `com.test.BenchmarkService`
- group: `benchmark`
- version: `1.0.0`

## Benchmark Naming

Use stable names so later optimization PRs can compare output directly.

Base benchmark groups:

- `BenchmarkURLString`
- `BenchmarkURLKey`
- `BenchmarkURLGetCacheInvokerMapKey`
- `BenchmarkURLServiceKey`
- `BenchmarkURLCopyParams`
- `BenchmarkURLGetParams`
- `BenchmarkURLClone`
- `BenchmarkURLCloneWithFilter`
- `BenchmarkURLMergeURL`
- `BenchmarkURLToMap`
- `BenchmarkURLIsEquals`

Each base group should use sub-benchmarks named by parameter count:

- `params_1`
- `params_32`
- `params_256`
- `params_1024`

Example shape:

```go
BenchmarkURLString/params_1
BenchmarkURLString/params_32
BenchmarkURLString/params_256
BenchmarkURLString/params_1024
```

## Special Scenarios

Add a small number of targeted special cases:

- `BenchmarkURLClone/params_256_with_suburl`
- `BenchmarkURLCloneWithFilter/params_256_exclude_20_percent`
- `BenchmarkURLCloneWithFilter/params_256_reserve_20_percent`
- `BenchmarkURLMergeURL/params_256_half_overlap`
- `BenchmarkURLMergeURL/params_256_with_method_params`
- `BenchmarkURLIsEquals/params_256_with_excludes`

The special scenarios should be limited to `params_256` unless a future reviewer asks
for broader coverage. This keeps the benchmark output readable while still making the
important costs visible.

## MergeURL Data Shape

`MergeURL` should have a half-overlap scenario.

For a parameter count `N`:

- left URL has keys `key0` through `keyN-1`
- right URL has keys beginning around `keyN/2`

That shape creates a mix of existing keys and new keys, which is closer to the real
cost of `MergeURL()` than either fully disjoint or fully identical inputs.

The method-params scenario should include a small, fixed set of methods and method
configuration keys so the method-specific merge branch is exercised without creating
excessive output.

## CloneWithFilter Data Shape

`CloneWithFilter` should include:

- no-filter base cases through `BenchmarkURLClone`
- exclude roughly 20 percent of keys
- reserve roughly 20 percent of keys

The exact keys should be generated deterministically from the parameter count.

## Benchmark Safety

Each benchmark should:

- call `b.ReportAllocs()`
- build input data before `b.ResetTimer()`
- write results to package-level sink variables
- avoid random data
- avoid changing shared benchmark inputs inside the timed loop unless the method under
  test requires mutation

Useful sink variables include:

```go
var benchmarkURLStringSink string
var benchmarkURLSink *URL
var benchmarkURLParamsSink url.Values
var benchmarkURLMapSink map[string]string
var benchmarkURLBoolSink bool
```

These sinks prevent the compiler from eliminating benchmarked calls.

## Expected Outcome

The final implementation should produce focused URL benchmark coverage under
`common`, with readable benchmark names and enough scenario coverage to support later
URL optimization issues. The change should remain behavior-neutral and should not touch
production code.
