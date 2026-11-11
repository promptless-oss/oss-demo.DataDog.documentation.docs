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

This page describes how to instrument your Go application with the Datadog Feature Flags SDK. The Go SDK integrates with [OpenFeature][1], an open standard for feature flag management, and is part of the Datadog Go tracer (`dd-trace-go`).

The SDK evaluates flags locally from flag configuration. By default, it fetches that configuration directly from Datadog over HTTPS, so no Datadog Agent is required. To receive configuration from the Agent instead, see [Receive configuration from the Agent](#receive-configuration-from-the-agent). For a comparison of the two sources, see [Server SDK Configuration Sources][6].

This guide explains how to install and enable the SDK, create an OpenFeature client, and evaluate feature flags in your application.

## Prerequisites

Before setting up the Go Feature Flags SDK, ensure you have:

- **Datadog Go SDK** `dd-trace-go` version 2.11.0 or later
- **Datadog [API key][3]** available to the application process

Set the following environment variables in the application process:

{{< code-block lang="bash" >}}
# Required: Agentless flag configuration and event delivery
DD_API_KEY=<DATADOG_API_KEY>
DD_SITE={{< region-param key="dd_site" code="true" >}}

# Required: Service identification
DD_SERVICE=<YOUR_SERVICE_NAME>
DD_ENV=<YOUR_ENVIRONMENT>

# Optional: Enable flag evaluation metrics
DD_METRICS_OTEL_ENABLED=true
{{< /code-block >}}

No Feature Flags enablement or source setting is required. The SDK starts fetching flag configuration when your application creates the Datadog OpenFeature provider. Starting the tracer alone does not fetch flag configuration.

Store `DD_API_KEY` in a secret manager and expose it only to the application process.

To configure `feature_flag.evaluations`, including the required tracer version and Agent OTLP setup, see [Set Up Server-Side Flag Evaluation Metrics][4]. For more information on available graphing, see [Feature Flag Graphs][5].

### Receive configuration from the Agent

To receive flag configuration from the Datadog Agent through Remote Configuration, you need:

- **Datadog Agent** version 7.55 or later with [Remote Configuration][2] enabled
- **Datadog [API key][3]** configured on the Agent
- **Datadog Go SDK** `dd-trace-go` version 2.4.0 or later

Select the source in the application process:

{{< code-block lang="bash" >}}
# dd-trace-go 2.11.0 or later
DD_FEATURE_FLAGS_CONFIGURATION_SOURCE=remote_config

# dd-trace-go 2.4.0 through 2.10.x
DD_EXPERIMENTAL_FLAGGING_PROVIDER_ENABLED=true
{{< /code-block >}}

`DD_EXPERIMENTAL_FLAGGING_PROVIDER_ENABLED` is deprecated beginning with `dd-trace-go` 2.11.0. If it is set to `true` and `DD_FEATURE_FLAGS_CONFIGURATION_SOURCE` is not set, the SDK stays on Remote Configuration. To move off the deprecated setting, see [Migrate an existing Remote Configuration setup][7].

### Feature flag event delivery

The provider sends exposure events and aggregated flag evaluation events to Datadog. The route depends on the configuration source:

- **Agentless (default)**: The SDK calls `GET /info` on the configured Agent URL to discover a compatible Agent Event Platform Proxy (EVP) endpoint. If it finds one, it sends events through the Agent. Otherwise, it sends events directly to `event-platform-intake.<DD_SITE>` over HTTPS and authenticates with `DD_API_KEY`. If the Agent later becomes unreachable or stops accepting these events, the SDK switches to direct delivery.
- **Remote Configuration**: The SDK sends events through the Datadog Agent only.

If no compatible Agent is reachable and `DD_API_KEY` is not set, the SDK cannot send events. It retries Agent discovery after 30 seconds. Flag evaluation is not affected.

To stop sending flag evaluation events while keeping exposure events, set `DD_FLAGGING_EVALUATION_COUNTS_ENABLED=false`.

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

Start the Datadog Go tracer, then create and register the Datadog OpenFeature provider. Starting the tracer first is recommended but not required. With Remote Configuration, the provider reuses the tracer's Remote Configuration subscription when the tracer is already running.

If you receive configuration from the Agent through Remote Configuration, start the Datadog Agent before the application. If the application starts first, restart it after the Agent is available. See [Agent availability][8].

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
    // Start the Datadog tracer (recommended before creating the provider)
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
    // Start the Datadog tracer (recommended before creating the provider)
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
[6]: /feature_flags/concepts/configuration_sources/
[7]: /feature_flags/concepts/configuration_sources/#migrate-an-existing-remote-configuration-setup
[8]: /feature_flags/concepts/configuration_sources/#agent-availability

## Further reading

{{< partial name="whats-next/whats-next.html" >}}
