---
title: Go Feature Flags
description: Set up Datadog Feature Flags for Go applications.
further_reading:
- link: "/feature_flags/server/"
  tag: "Documentation"
  text: "Server-Side Feature Flags"
- link: "/tracing/trace_collection/dd_libraries/go/"
  tag: "Documentation"
  text: "Go Tracing"
- link: "/feature_flags/guide/server_flag_evaluation_metrics/"
  tag: "Guide"
  text: "Set Up Server-Side Flag Evaluation Metrics"
- link: "/feature_flags/guide/apm_trace_enrichment/"
  tag: "Guide"
  text: "Set Up APM Trace Enrichment for Feature Flags"
- link: "/feature_flags/concepts/flag_graphs/"
  tag: "Concept"
  text: "Feature Flag Graphs"
---

## Overview

This page describes how to instrument your Go application with the Datadog Feature Flags SDK. The Go SDK integrates with [OpenFeature][1], an open standard for feature flag management, and receives flag updates through Remote Configuration in the Datadog Go tracer (`dd-trace-go`).

This guide explains how to install and enable the SDK, create an OpenFeature client, and evaluate feature flags in your application.

## Prerequisites

Before setting up the Go Feature Flags SDK, ensure you have:

- **Datadog Agent** version 7.55 or later with [Remote Configuration][2] enabled
- **Datadog [API key][3]** configured on the Agent
- **Datadog Go SDK** `dd-trace-go` version 2.4.0 or later

Set the following environment variables:

{{< code-block lang="bash" >}}
# Required: Enable the feature flags provider
DD_EXPERIMENTAL_FLAGGING_PROVIDER_ENABLED=true

# Optional: Enable flag evaluation metrics
DD_METRICS_OTEL_ENABLED=true

# Required: Service identification
DD_SERVICE=<YOUR_SERVICE_NAME>
DD_ENV=<YOUR_ENVIRONMENT>
{{< /code-block >}}

<div class="alert alert-info">The <code>EXPERIMENTAL_</code> prefix is retained for backwards compatibility; the provider itself is stable.</div>

To configure `feature_flag.evaluations`, including the required tracer version and Agent OTLP setup, see [Set Up Server-Side Flag Evaluation Metrics][4]. For more information on available graphing, see [Feature Flag Graphs][5].

## Installation

Install the Datadog OpenFeature provider package:

{{< code-block lang="bash" >}}
go get github.com/DataDog/dd-trace-go/v2/openfeature
{{< /code-block >}}

You also need the OpenFeature Go SDK:

{{< code-block lang="bash" >}}
go get github.com/open-feature/go-sdk/openfeature
{{< /code-block >}}

## Initialize the SDK

Start the Datadog Go tracer and register the Datadog OpenFeature provider. The tracer must be started first because it enables Remote Configuration, which delivers flag configurations to your application.

Start the Datadog Agent before the application. If the application starts first, restart it after the Agent is available. See [Agent availability][6].

### Blocking initialization

Use `SetProviderAndWait` to block evaluation until the initial flag configuration is received. This ensures flags are ready before your application starts handling requests.

{{< code-block lang="go" >}}
package main

import (
    "log"

    "github.com/DataDog/dd-trace-go/v2/ddtrace/tracer"
    ddopenfeature "github.com/DataDog/dd-trace-go/v2/openfeature"
    "github.com/open-feature/go-sdk/openfeature"
)

func main() {
    // Start the Datadog tracer (enables Remote Config)
    tracer.Start()
    defer tracer.Stop()

    // Create the Datadog OpenFeature provider
    provider, err := ddopenfeature.NewDatadogProvider(ddopenfeature.ProviderConfig{})
    if err != nil {
        log.Fatalf("Failed to create provider: %v", err)
    }
    if ddProvider, ok := provider.(*ddopenfeature.DatadogProvider); ok {
        defer ddProvider.Shutdown()
    }

    // Register the provider and wait for initialization (default 30s timeout)
    if err := openfeature.SetProviderAndWait(provider); err != nil {
        log.Fatalf("Failed to set provider: %v", err)
    }

    // Create the OpenFeature client
    client := openfeature.NewClient("my-service")

    // Your application code here
}
{{< /code-block >}}

To specify a custom timeout, use `SetProviderAndWaitWithContext`:

{{< code-block lang="go" >}}
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := openfeature.SetProviderAndWaitWithContext(ctx, provider); err != nil {
    log.Fatalf("Failed to set provider: %v", err)
}
{{< /code-block >}}

### Non-blocking initialization

Use `SetProvider` to register the provider without waiting. Flag evaluations return default values until the configuration is received.

{{< code-block lang="go" >}}
package main

import (
    "log"

    "github.com/DataDog/dd-trace-go/v2/ddtrace/tracer"
    ddopenfeature "github.com/DataDog/dd-trace-go/v2/openfeature"
    "github.com/open-feature/go-sdk/openfeature"
)

func main() {
    // Start the Datadog tracer (enables Remote Config)
    tracer.Start()
    defer tracer.Stop()

    // Create the Datadog OpenFeature provider
    provider, err := ddopenfeature.NewDatadogProvider(ddopenfeature.ProviderConfig{})
    if err != nil {
        log.Fatalf("Failed to create provider: %v", err)
    }
    if ddProvider, ok := provider.(*ddopenfeature.DatadogProvider); ok {
        defer ddProvider.Shutdown()
    }

    // Register the provider without waiting
    openfeature.SetProvider(provider)

    // Create the OpenFeature client
    client := openfeature.NewClient("my-service")

    // Your application code here
    // Flag evaluations return defaults until configuration is received
}
{{< /code-block >}}

## Create a client

Create an OpenFeature client to evaluate flags. You can create multiple clients with different names for different parts of your application:

{{< code-block lang="go" >}}
// Create a client for your application
client := openfeature.NewClient("my-service")
{{< /code-block >}}

## Set the evaluation context

Define an evaluation context that identifies the user or entity for flag targeting. The evaluation context includes attributes used to determine which flag variations should be returned:

<div class="alert alert-warning">Datadog Feature Flags requires evaluation context attributes to be flat primitive values: strings, numbers, and Booleans. Do not pass nested objects or arrays; they are not supported and can cause exposure data to be dropped.</div>

{{< code-block lang="go" >}}
evalCtx := openfeature.NewEvaluationContext(
    "user-123", // Targeting key (typically user ID)
    map[string]interface{}{
        "email":   "user@example.com",
        "country": "US",
        "tier":    "premium",
        "age":     25,
    },
)
{{< /code-block >}}

The targeting key is used for consistent traffic distribution (percentage rollouts). Additional attributes enable targeting rules, such as "enable for users in the US" or "enable for premium tier users" in the example above.

## Evaluate flags

After setting up the provider and creating a client, you can evaluate flags throughout your application. Flag evaluation is local and fast—the SDK uses locally cached configuration data, so no network requests occur during evaluation.

Each flag is identified by a key (a unique string) and can be evaluated with a typed method that returns a value of the expected type. If the flag doesn't exist or cannot be evaluated, the SDK returns the provided default value.

### Boolean flags

Use `BooleanValue` for flags that represent on/off or true/false conditions:

{{< code-block lang="go" >}}
ctx := context.Background()

enabled, err := client.BooleanValue(ctx, "new-checkout-flow", false, evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}

if enabled {
    showNewCheckout()
} else {
    showLegacyCheckout()
}
{{< /code-block >}}

### String flags

Use `StringValue` for flags that select between multiple variants or configuration strings:

{{< code-block lang="go" >}}
theme, err := client.StringValue(ctx, "ui-theme", "light", evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}

switch theme {
case "dark":
    setDarkTheme()
case "light":
    setLightTheme()
default:
    setLightTheme()
}
{{< /code-block >}}

### Numeric flags

For numeric flags, use `IntValue` or `FloatValue`. These are appropriate when a feature depends on a numeric parameter such as a limit, percentage, or multiplier:

{{< code-block lang="go" >}}
maxItems, err := client.IntValue(ctx, "cart-max-items", 20, evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}

discountRate, err := client.FloatValue(ctx, "discount-rate", 0.0, evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}
{{< /code-block >}}

### Object flags

For structured data, use `ObjectValue`. This returns a value that can be type-asserted to maps or other complex types:

{{< code-block lang="go" >}}
config, err := client.ObjectValue(ctx, "feature-config", map[string]interface{}{
    "maxRetries": 3,
    "timeout":    30,
}, evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}

// Type assert to access the configuration
if configMap, ok := config.(map[string]interface{}); ok {
    maxRetries := configMap["maxRetries"]
    timeout := configMap["timeout"]
    // Use configuration values
}
{{< /code-block >}}

### Flag evaluation details

When you need more than just the flag value, use the `*ValueDetails` methods. These return both the evaluated value and metadata explaining the evaluation:

{{< code-block lang="go" >}}
details, err := client.BooleanValueDetails(ctx, "new-feature", false, evalCtx)
if err != nil {
    log.Printf("Error evaluating flag: %v", err)
}

fmt.Printf("Value: %v\n", details.Value)
fmt.Printf("Variant: %s\n", details.Variant)
fmt.Printf("Reason: %s\n", details.Reason)
fmt.Printf("Error: %v\n", details.Error())
{{< /code-block >}}

Flag details help you debug evaluation behavior and understand why a user received a given value.

## Exposures

For flags whose allocation logs exposures, such as experiment flags, the Datadog provider records an exposure after each successful evaluation. It sends the exposure to Datadog from an OpenFeature provider hook that runs when the evaluation finishes. An evaluation that returns an error, including one that an application `After` hook rejects, is not an exposure.

For each flag and subject, the provider remembers the last exposure it recorded. It records another exposure only when the allocation, variant, or split serial ID changes, so repeated evaluations for the same subject do not send duplicates.

Provider hooks run only when you evaluate flags through an OpenFeature client. Register the Datadog provider directly with OpenFeature, as shown in the [Initialize the SDK](#initialize-the-sdk) section. Inside an OpenFeature `MultiProvider` or a custom provider wrapper, flags still evaluate. However, the provider logs a warning and records no exposures.

<div class="alert alert-warning">Earlier versions of <code>dd-trace-go</code> attempted to send exposures from inside a <code>MultiProvider</code>. Starting with version <strong>X.Y.Z</strong>, the provider sends no exposures there. To keep sending exposures, register the Datadog provider directly.</div>

To reject a flag result in an application `After` hook, return an error instead of panicking. When an `After` hook panics, OpenFeature still completes the evaluation hooks, so the provider can record an exposure for a result your application rejected.

### Record exposures in your own data store

To record exposures in your own data store or analytics pipeline in addition to Datadog, register a hook created with `NewExposureHook`. The hook receives every completed evaluation, including errors, non-exposures, and repeats, along with the same exposure decision that the provider uses for Datadog. `NewExposureHook` requires `dd-trace-go` version **X.Y.Z** or later.

The following example copies each new exposure to a channel that a background goroutine writes to the data store. `exposureRecord` stands for your own record type:

{{< code-block lang="go" >}}
package main

import (
    ddopenfeature "github.com/DataDog/dd-trace-go/v2/openfeature"
    "github.com/open-feature/go-sdk/openfeature"
)

type exposureRecord struct {
    FlagKey         string
    Variant         string
    Subject         string
    Allocation      string
    HoldoutKey      any
    AssignmentGroup any
}

func registerExposureHook(client *openfeature.Client, records chan<- exposureRecord) {
    client.AddHooks(ddopenfeature.NewExposureHook(func(e ddopenfeature.ExposureEvaluation) {
        // Skip evaluations that are not new exposures
        if !e.ShouldSend() {
            return
        }
        allocation, _ := e.AllocationKey()
        features := e.Features()
        record := exposureRecord{
            FlagKey:         e.Details.FlagKey,
            Variant:         e.Details.Variant,
            Subject:         e.Context.TargetingKey(),
            Allocation:      allocation,
            HoldoutKey:      features["holdout.key"],
            AssignmentGroup: features["holdout.assignment_group"],
        }
        // The callback runs on the evaluating goroutine: hand off the record and return
        select {
        case records <- record:
        default:
            // Queue is full: drop the record or count it
        }
    }))
}
{{< /code-block >}}

You can also register the hook globally with `openfeature.AddHooks`, or for a single evaluation with `openfeature.WithHooks`. Create the hook once and reuse it.

The callback runs on the goroutine that evaluates the flag, and several evaluations can call it at the same time. Do not write to the data store from the callback. Copy the data you need and hand it to a background queue. Treat `Details.Value` as read-only, because object flag values can share state with the provider. If the callback panics, the hook recovers, logs a warning, and returns the flag result unchanged.

Each `ExposureEvaluation` that the callback receives provides these fields and methods:

| Field or method | Returns |
|---|---|
| `ShouldSend()` | `true` for a new or changed exposure. The evaluation succeeded, its allocation logs exposures, and the subject was not yet exposed to the same allocation, variant, and split serial ID. |
| `IsExposure()` | `true` for every successful evaluation of an allocation that logs exposures, including repeats. |
| `IsCacheHit()` | `true` when the exposure repeats one already recorded for the subject, with the same allocation, variant, and split serial ID. |
| `Features()` | The features of the selected split that Datadog routes to hooks. Values are a `string`, `bool`, or `float64`. Returns `nil` when the split has no such features. |
| `AllocationKey()` | The selected allocation key, and whether it is present. |
| `SplitSerialID()` | The selected split's serial ID, and whether it is present. A serial ID of `0` is different from an absent serial ID. |
| `Details` | The OpenFeature `InterfaceEvaluationDetails` for the evaluation. |
| `Context` | The evaluation context. |

Every exposure hook receives the same decision, so one hook does not consume an exposure before another. `ShouldSend()` reports what the provider recorded, not whether your pipeline delivered it. Your application owns queueing, retries, and flushing on shutdown. If you need delivery guarantees independent of the provider, use `IsExposure()` and deduplicate exposures in your own pipeline.

Splits in a holdout carry these features:

| Feature | Value |
|---|---|
| `holdout.key` | The holdout key |
| `holdout.assignment_group` | The holdout assignment group of the subject |

### Send exposures only to your own data store

To stop sending exposures to Datadog while you record them with an exposure hook, set the following environment variable before the application starts:

{{< code-block lang="bash" >}}
DD_FEATURE_FLAGS_EXPOSURES_DATADOG_LOGGING_ENABLED=false
{{< /code-block >}}

The default is `true`. The provider reads this setting when it is created. With the setting set to `false`, exposure hooks and exposure deduplication keep working, and flag values, flag evaluation metrics, and APM trace enrichment are unchanged.

## Testing

You can test against a dedicated Datadog test environment with the real `DatadogProvider`, or swap it for OpenFeature's in-memory provider to control flag values directly in test code. This section shows the in-memory approach, which keeps tests hermetic and offline. The in-memory provider ships in the upstream `go-sdk` module under `github.com/open-feature/go-sdk/openfeature/memprovider`, so no additional dependency is required.

Register the in-memory provider under a **named client** rather than the default global provider. The default provider is process-global, which breaks `t.Parallel()` and leaks flag state between tests. A named client scopes the provider to each test.

{{< code-block lang="go" >}}
package checkout

import (
    "context"
    "testing"

    "github.com/open-feature/go-sdk/openfeature"
    "github.com/open-feature/go-sdk/openfeature/memprovider"
)

func TestNewCheckoutFlow(t *testing.T) {
    cases := []struct {
        name string
        tier string
        want bool
    }{
        {"premium user sees new flow", "premium", true},
        {"free user sees legacy", "free", false},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            evalByTier := func(flag memprovider.InMemoryFlag, flatCtx openfeature.FlattenedContext) (any, openfeature.ProviderResolutionDetail) {
                if flatCtx["tier"] == "premium" {
                    return flag.Variants["on"], openfeature.ProviderResolutionDetail{Reason: openfeature.TargetingMatchReason, Variant: "on"}
                }
                return flag.Variants[flag.DefaultVariant], openfeature.ProviderResolutionDetail{Reason: openfeature.DefaultReason, Variant: flag.DefaultVariant}
            }

            provider := memprovider.NewInMemoryProvider(map[string]memprovider.InMemoryFlag{
                "new-checkout-flow": {
                    State:            memprovider.Enabled,
                    DefaultVariant:   "off",
                    Variants:         map[string]any{"on": true, "off": false},
                    ContextEvaluator: &evalByTier,
                },
            })

            name := "test-" + t.Name()
            if err := openfeature.SetNamedProviderAndWait(name, provider); err != nil {
                t.Fatal(err)
            }
            t.Cleanup(func() {
                _ = openfeature.SetNamedProviderAndWait(name, openfeature.NoopProvider{})
            })

            client := openfeature.NewClient(name)
            got, err := client.BooleanValue(context.Background(), "new-checkout-flow", false, openfeature.NewEvaluationContext("user-1", map[string]any{"tier": tc.tier}))
            if err != nil {
                t.Fatal(err)
            }
            if got != tc.want {
                t.Errorf("got %v, want %v", got, tc.want)
            }
        })
    }
}
{{< /code-block >}}

`ContextEvaluator` is defined as `*func(...)` — a pointer to a function. Define the evaluator in a local variable and pass its address with `&`, as shown above. Omit `ContextEvaluator` entirely to always return `DefaultVariant`.

[1]: https://openfeature.dev/
[2]: /agent/remote_config/
[3]: /account_management/api-app-keys/#api-keys
[4]: /feature_flags/guide/server_flag_evaluation_metrics/
[5]: /feature_flags/concepts/flag_graphs/
[6]: /feature_flags/concepts/configuration_sources/#agent-availability

## Further reading

{{< partial name="whats-next/whats-next.html" >}}
