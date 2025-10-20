#### 1.1.0 October 20 2025 ####

**Critical Compatibility Fix: Support for Both .NET 9 Built-in Metrics and OpenTelemetry.Instrumentation.Runtime**

This release fixes a critical compatibility issue discovered after the initial 1.0.0 release where the dashboard only displayed data for .NET 9+ applications, leaving .NET 6/7/8 users with blank dashboards.

## The Problem

The 1.0.0 dashboard was created using .NET 9's newly introduced built-in runtime metrics (released November 2024), which emit metrics with the `dotnet_*` naming convention (e.g., `dotnet_gc_collections_total`). However, users running .NET 6, 7, or 8 rely on the `OpenTelemetry.Instrumentation.Runtime` package, which emits metrics with a different naming convention: `process_runtime_dotnet_*` (e.g., `process_runtime_dotnet_gc_collections_count_total`).

This naming inconsistency caused the dashboard to show no data for .NET 6/7/8 applications, rendering it unusable for a significant portion of the .NET community who haven't yet upgraded to .NET 9.

## Root Cause

.NET runtime metrics can come from two completely different sources:

1. **.NET 9+ Built-in Metrics** (introduced in .NET 9, November 2024)
   - Automatically available in all .NET 9+ applications
   - No additional packages required
   - Emitted by the `System.Runtime` meter
   - Uses `dotnet.*` naming convention
   - Example: `dotnet_gc_collections_total`

2. **OpenTelemetry.Instrumentation.Runtime Package** (community package)
   - Requires explicit installation: `OpenTelemetry.Instrumentation.Runtime`
   - Must call `AddRuntimeInstrumentation()` in OpenTelemetry setup
   - Works with .NET 6, 7, 8, and 9
   - Uses OpenTelemetry semantic conventions
   - Uses `process.runtime.dotnet.*` naming convention
   - Example: `process_runtime_dotnet_gc_collections_count_total`

The original dashboard was built after .NET 9 adoption in Petabridge's testlab environment, where all services had already migrated to .NET 9 and were using built-in metrics. This led to the dashboard being designed exclusively around the `dotnet_*` naming convention.

## The Solution

This release implements a comprehensive compatibility layer that allows the dashboard to work seamlessly with both metric sources:

### 1. Query Pattern Matching (15 panels updated)

All panel queries now use PromQL regex patterns to match both naming conventions simultaneously:

**Before:**
```promql
dotnet_gc_collections_total{job=~"$job"}
```

**After:**
```promql
{__name__=~"dotnet_gc_collections_total|process_runtime_dotnet_gc_collections_count_total", job=~"$job"}
```

### 2. Unit Conversion for Time Metrics (Critical Bug Fix)

Discovered during testing that two metrics have different units between the sources:
- .NET 9 built-in: Reports in **seconds**
- OpenTelemetry package: Reports in **nanoseconds**

Fixed with proper unit conversion:

**GC Pause Time:**
```promql
sum by (job) (irate(dotnet_gc_pause_time_seconds_total{job=~"$job"}[$__range]))
or
sum by (job) (irate(process_runtime_dotnet_gc_duration_nanoseconds_total{job=~"$job"}[$__range]) / 1e9)
```

**JIT Compilation Time:**
```promql
sum by(job) (irate(dotnet_jit_compilation_time_seconds_total{job=~"$job"}[$__range]))
or
sum by(job) (irate(process_runtime_dotnet_jit_compilation_time_nanoseconds_total{job=~"$job"}[$__range]) / 1e9)
```

### 3. Variable Queries Updated

Dashboard variables (job and instance selectors) now query both metric sources:
```promql
label_values({__name__=~"dotnet_assembly_count|process_runtime_dotnet_assemblies_count"},job)
```

This ensures the dropdown menus populate with services from either metric source.

### 4. CPU Metrics Documentation

CPU metrics (`dotnet.process.cpu.time` and `dotnet.process.cpu.count`) are **only available in .NET 9+ built-in metrics**. The OpenTelemetry.Instrumentation.Runtime package does not provide these metrics for .NET 6/7/8.

Added warning descriptions to CPU panels:
> ⚠️ Requires .NET 9+ built-in metrics (not available in OpenTelemetry.Instrumentation.Runtime package for .NET 6/7/8).

## Compatibility Matrix

| Metric Category | .NET 9+ Built-in | OpenTelemetry.Instrumentation.Runtime |
|----------------|------------------|---------------------------------------|
| Exceptions | ✅ | ✅ |
| Assemblies | ✅ | ✅ |
| Garbage Collection | ✅ | ✅ |
| ThreadPool | ✅ | ✅ |
| Timers | ✅ | ✅ |
| JIT Compilation | ✅ | ✅ |
| **CPU Metrics** | ✅ | ❌ Not available |

## Changes in This Release

- [#16](https://github.com/petabridge/dotnet-grafana-dashboards/pull/16) Support both .NET 9 built-in and OpenTelemetry.Instrumentation.Runtime metrics
- Updated 15 panel queries with regex pattern matching for dual metric source support
- Fixed GC pause time metric name and unit conversion (nanoseconds → seconds)
- Fixed JIT compilation time metric name and unit conversion (nanoseconds → seconds)
- Updated 2 dashboard variable queries (job/instance selectors) for dual metric source support
- Added warning descriptions to CPU panels documenting .NET 9+ requirement
- Comprehensive README documentation covering both metric sources
- Incremented dashboard version from 46 → 48

## Impact on Users

✅ **Non-breaking change** - Existing .NET 9 users will see no difference in functionality

✅ **Backward compatible** - Dashboard now works automatically for:
- .NET 9+ applications using built-in `System.Runtime` meter
- .NET 6/7/8 applications using `OpenTelemetry.Instrumentation.Runtime` package
- Mixed environments with both metric sources

✅ **No configuration required** - The dashboard automatically detects and displays data from whichever metric source is available

⚠️ **CPU metrics limitation** - Users on .NET 6/7/8 will see empty CPU panels (documented with warnings)

## Migration Guidance

### For .NET 6/7/8 Users
Continue using `OpenTelemetry.Instrumentation.Runtime` package:
```csharp
services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddRuntimeInstrumentation());
```

### For .NET 9+ Users
Built-in metrics are automatically available. You can optionally remove the `OpenTelemetry.Instrumentation.Runtime` package dependency:
```csharp
// No additional setup needed - metrics are automatic in .NET 9+
services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("System.Runtime")); // Optional: explicitly subscribe
```

## References

- [.NET 9 Built-in Runtime Metrics Documentation](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-runtime)
- [OpenTelemetry.Instrumentation.Runtime Package](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Instrumentation.Runtime)
- [OpenTelemetry Issue #2071](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/issues/2071) - Discussion on .NET 9 compatibility

---

#### 1.0.0 April 2nd 2025 ####

Initial release
