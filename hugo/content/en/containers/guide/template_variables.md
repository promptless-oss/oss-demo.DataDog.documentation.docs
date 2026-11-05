---
title: Autodiscovery Template Variables
description: Reference guide for template variables available in Autodiscovery integration configuration for dynamic container environments
aliases:
 - /agent/autodiscovery/template_variables
 - /agent/faq/template_variables
 - /agent/guide/template_variables
further_reading:
- link: "/containers/kubernetes/integrations/"
  tag: "Documentation"
  text: "Configure integrations with Autodiscovery on Kubernetes"
- link: "/containers/docker/integrations/"
  tag: "Documentation"
  text: "Configure integrations with Autodiscovery on Docker"
- link: "/agent/guide/ad_identifiers/"
  tag: "Documentation"
  text: "Match a container with the corresponding integration template"
- link: "/agent/guide/autodiscovery-management/"
  tag: "Documentation"
  text: "Manage which container to include in the Agent Autodiscovery"
---

[Autodiscovery][1] enables you to set static configurations for dynamic resources like containers.

You can use the following template variables to dynamically assign your container's values:

| Template Variable | Description |
| --------------------------  | ---    |
| `"%%host%%"`                | The container's network IP. |
| `"%%host_<NETWORK NAME>%%"` | When the container is attached to multiple networks, returns the network name to use. |
| `"%%port%%"`                | The highest exposed port **sorted numerically and in ascending order**.<br>For example, returns `8443` for a container that exposes ports `80`, `443`, and `8443`. |
| `"%%port_<NUMBER_X>%%"`     | The `<NUMBER_X>` port **sorted numerically and in ascending order**.<br>For example, if a container exposes ports `80`, `443`, and `8443`, `"%%port_0%%` refers to port `80`, and `"%%port_1%%"` refers to `443`. |
| `"%%port_<NAME>%%"`     | The port associated with the port name `<NAME>`. |
| `"%%pid%%"`                 | The container process ID, as returned by `docker inspect --format '{{.State.Pid}}' <CONTAINER_NAME>`. |
| `"%%hostname%%"`            | The `hostname` value from the container configuration. Only use this variable if the `"%%host%%"` variable cannot fetch a reliable IP (for example, in [ECS awsvpc mode][2]).                                       |
| `"%%env_<ENV_VAR>%%"`       | The contents of the `$<ENV_VAR>` environment variable **as seen by the Agent process**. Environment variables bound to Agent settings are not resolved by default. See the [Environment variable restrictions](#environment-variable-restrictions) section. |
| `"%%kube_namespace%%"`      | The Kubernetes namespace. |
| `"%%kube_pod_name%%"`       | The Kubernetes pod name.  |
| `"%%kube_pod_uid%%"`        | The Kubernetes pod UID.   |

**Fall back**:

* For the `"%%host%%"` template variable: in case the Agent is not able to find the IP, this template variable falls back to the `bridge` network IP.
* For the `"%%host_<NETWORK NAME>%%"`: if the `<NETWORK_NAME>` specified is not found, this template variable behaves like `"%%host%%"`.

Depending on your platform, not all template variables are supported:

| Platform    | Auto-discovery identifiers  | Host | Port | Tag | Pid | Env | Hostname | Kube Namespace | Pod Name | Pod UID |
| ----------- | ---                         | ---  | ---  | --- | --- | --- | ---      | ---            | ---      | ---     |
| Docker      | ✅                          | ✅   | ✅   | ✅  | ✅  | ✅  | ✅      | ❌      | ❌      | ❌      |
| ECS Fargate | ✅                          | ✅   | ❌   | ✅  | ❌  | ✅  | ❌      | ❌      | ❌      | ❌      |
| Kubernetes  | ✅                          | ✅   | ✅   | ✅  | ❌  | ✅  | ❌      | ✅      | ✅      | ✅      |

## Environment variable restrictions

The `%%env_<ENV_VAR>%%` template variable reads environment variables from the Agent process. Workload owners can supply Autodiscovery templates through sources such as pod annotations or container labels. To keep those templates from reading Agent credentials, the Agent limits which environment variables a template can resolve.

Starting with Agent v7.XX, `%%env_<ENV_VAR>%%` resolves any environment variable except those bound to Agent settings, such as `DD_API_KEY`, `DD_SITE`, and `DD_CLUSTER_AGENT_AUTH_TOKEN`. `DD_CLUSTER_NAME` and `DD_ENV` are the exceptions and resolve by default. This applies to both check and log configurations. When a template references a blocked environment variable, the Agent doesn't schedule the configuration for that container and reports an error like the following:

```text
envvar DD_API_KEY is not allowed in check configs, skipping service <SERVICE_ID>
```

Control which environment variables templates can resolve with these Agent options:

| Option | Environment variable | Description |
| --- | --- | --- |
| `ad_allowed_agent_env_vars` | `DD_AD_ALLOWED_AGENT_ENV_VARS` | Environment variables bound to Agent settings that templates can resolve. Defaults to `DD_CLUSTER_NAME` and `DD_ENV`. Setting this list replaces the default, so include `DD_CLUSTER_NAME` and `DD_ENV` if you still need them. Ignored when `ad_allowed_env_vars` is set. |
| `ad_allowed_env_vars` | `DD_AD_ALLOWED_ENV_VARS` | When set, templates can resolve only the listed environment variables, including any bound to Agent settings. Defaults to an empty list. |
| `ad_disable_env_var_resolution` | `DD_AD_DISABLE_ENV_VAR_RESOLUTION` | Set to `true` to turn off `%%env_<ENV_VAR>%%` resolution entirely. Defaults to `false`. |

For example, to keep resolving `%%env_DD_SITE%%` along with the defaults, add the following to your `datadog.yaml` file:

```yaml
ad_allowed_agent_env_vars:
  - DD_CLUSTER_NAME
  - DD_ENV
  - DD_SITE
```

To set the same list as an environment variable, separate the names with spaces:

```shell
DD_AD_ALLOWED_AGENT_ENV_VARS="DD_CLUSTER_NAME DD_ENV DD_SITE"
```

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /getting_started/containers/autodiscovery
[2]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html