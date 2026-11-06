---
title: APM Basic with Single Step Instrumentation on Kubernetes
is_beta: true
private: true
description: Use the Datadog Cluster Agent to inject the C tracer into selected Kubernetes pods so they report request, error, and duration metrics without sending traces.
further_reading:
- link: /tracing/trace_collection/single-step-apm/kubernetes/
  tag: "Documentation"
  text: "Single Step APM Instrumentation on Kubernetes"
- link: /containers/cluster_agent/commands/
  tag: "Documentation"
  text: "Cluster Agent commands and options"
---

<div class="alert alert-info">APM Basic is in Preview and is disabled by default.</div>

## Overview

APM Basic reports request, error, and duration metrics (RED metrics) for selected Kubernetes workloads without sending traces. Use it to get service health metrics for a workload without instrumenting it with a language-specific Datadog SDK.

APM Basic is part of [Single Step Instrumentation][1] (SSI) on Kubernetes. When it's enabled, the Cluster Agent injects the Datadog C tracer (`dd-trace-c`) into new pods that carry the `admission.datadoghq.com/apm-basic.enabled: "true"` label. Pods without the label aren't affected by APM Basic. APM Basic works whether or not SSI is enabled for the rest of the cluster.

## Requirements

- Datadog Cluster Agent v7.XX or later, with the [Admission Controller][2] enabled.

## Configure the Cluster Agent

APM Basic is controlled by the `apm_config.instrumentation.apm_basic.enabled` setting, which defaults to `false`. To enable it, set the `DD_APM_INSTRUMENTATION_APM_BASIC_ENABLED` environment variable on the Cluster Agent.

{{< tabs >}}
{{% tab "Datadog Operator" %}}

Merge the following configuration into the existing `DatadogAgent` resource:

```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  override:
    clusterAgent:
      env:
        - name: DD_APM_INSTRUMENTATION_APM_BASIC_ENABLED
          value: "true"
```

Apply the configuration and wait for the Cluster Agent rollout to complete.

{{% /tab %}}
{{% tab "Helm" %}}

Merge the following configuration into your existing `datadog-values.yaml` file:

```yaml
clusterAgent:
  env:
    - name: DD_APM_INSTRUMENTATION_APM_BASIC_ENABLED
      value: "true"
```

Upgrade your Helm release and wait for the rollout to complete:

```shell
helm upgrade -f datadog-values.yaml <RELEASE_NAME> datadog/datadog
```

{{% /tab %}}
{{< /tabs >}}

## Label the workloads

Add the `admission.datadoghq.com/apm-basic.enabled: "true"` label to the pod template of each workload you want to monitor with APM Basic:

```yaml
spec:
  template:
    metadata:
      labels:
        admission.datadoghq.com/apm-basic.enabled: "true"
```

Apply the resource. Existing pods aren't changed, so APM Basic applies to the pods the controller creates after the update.

The Cluster Agent skips pods labeled `admission.datadoghq.com/enabled: "false"` and pods in namespaces listed in `apm_config.instrumentation.disabled_namespaces`.

## Verify the setup

Confirm that the C tracer settings are set on a new pod:

```shell
kubectl exec <POD_NAME> -n <NAMESPACE> -- sh -c \
  'env | grep -E "^(DD_INJECT_NATIVE|DD_APM_BASIC_ENABLED)="'
```

The output includes `DD_INJECT_NATIVE=always` and `DD_APM_BASIC_ENABLED=true`. `DD_INJECT_NATIVE=always` makes the injector load the C tracer into every process in the container. `DD_APM_BASIC_ENABLED=true` makes the C tracer report RED metrics without sending traces.

## Pin the C tracer version

By default, the Cluster Agent injects the latest `0.x` release of the C tracer. To inject a specific version, set `apm_config.instrumentation.apm_basic.tracer_versions` with the `DD_APM_INSTRUMENTATION_APM_BASIC_TRACER_VERSIONS` environment variable. The value is a JSON map of tracer language to version. Add it to the Cluster Agent `env` list alongside `DD_APM_INSTRUMENTATION_APM_BASIC_ENABLED`:

```yaml
- name: DD_APM_INSTRUMENTATION_APM_BASIC_TRACER_VERSIONS
  value: '{"c": "<C_TRACER_VERSION>"}'
```

## How APM Basic interacts with other instrumentation

The Cluster Agent selects one instrumentation configuration for each pod. When a pod matches more than one, the following rules apply:

- A pod that also matches [Continuous Tracing with GPU Monitoring][3], a `DatadogInstrumentation` resource, or a remote instrumentation rule uses that configuration instead of APM Basic.
- A pod that also matches one of your SSI [targets][4], or that SSI instruments by default, uses APM Basic. The Cluster Agent injects only the C tracer into that pod, not the language SDKs.

## Further reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /tracing/trace_collection/single-step-apm/kubernetes/
[2]: /containers/cluster_agent/admission_controller/
[3]: /gpu_monitoring/tracing/
[4]: /tracing/trace_collection/single-step-apm/kubernetes/#target-specific-workloads
