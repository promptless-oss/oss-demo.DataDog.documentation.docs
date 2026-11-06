---
title: Private Action Runner Reference
description: Reference tables for private action runner configuration settings, supported actions and integrations, credential file formats, and named runner credentials.
further_reading:
- link: "/actions/private_actions/"
  tag: "Documentation"
  text: "Private Actions Overview"
- link: "/actions/private_actions/set_up_agent_based/"
  tag: "Documentation"
  text: "Set up a private action runner"
- link: "/actions/private_actions/execution_policies/"
  tag: "Documentation"
  text: "Execution Policies"
- link: "/actions/connections/private_action_credentials/"
  tag: "Documentation"
  text: "Handling private action credentials"
---

## Overview

This page covers private action runner configuration settings, the actions and integrations each runner supports, credential file formats, and named runner credentials. To learn more about Private Actions concepts and setup, see [Private Actions Overview][1].

## Runner configuration

The runner reads its settings from the `private_action_runner` section of its [configuration][2].

For how the enrollment settings (`self_enroll` and `api_key_only_enrollment`) fit into runner enrollment, see [Enrollment and ownership][3].

The same setting is named differently depending on how you install the runner. Host installs use environment variables, Helm uses camelCase keys under `privateActionRunner`, and the Datadog Operator uses snake_case keys under `private_action_runner`. Use this table to translate the common settings across install methods.

| Setting | Host (environment variable) | Helm (`privateActionRunner.*`) | Operator (`private_action_runner.*`) |
|---|---|---|---|
| Enable | `DD_PRIVATE_ACTION_RUNNER_ENABLED` | `enabled` | `enabled` |
| Self-enroll | `DD_PRIVATE_ACTION_RUNNER_SELF_ENROLL` | `selfEnroll` | `self_enroll` |
| Actions allowlist | `DD_PRIVATE_ACTION_RUNNER_ACTIONS_ALLOWLIST` (comma-separated) | `actionsAllowlist` (list) | `actions_allowlist` (list) |

## Supported actions and integrations

This matrix shows, for each integration, its availability in each runner type and whether it is authorizable through [Execution Policies][4].

<div class="alert alert-info">Availability in the Agent and authorization through Execution Policies are independent. An integration can run in the Agent without being authorizable through an Execution Policy.</div>

| Integration | Runner in the Agent | Authorizable through<br>Execution Policies | Standalone runner |
|---|:---:|:---:|:---:|
| Kubernetes | {{< X >}} | {{< X >}} | {{< X >}} |
| Remote Action (for example, rshell) | {{< X >}} | {{< X >}} | {{< X >}} |
| Script | {{< X >}} | {{< X >}} | {{< X >}} |
| HTTP | {{< X >}} |  | {{< X >}} |
| GitLab | {{< X >}} |  | {{< X >}} |
| Jenkins | {{< X >}} |  | {{< X >}} |
| MongoDB | {{< X >}} |  | {{< X >}} |
| PostgreSQL |  |  | {{< X >}} |
| Temporal | {{< X >}} |  | {{< X >}} |

- **Remote Action** is the integration family under the `com.datadoghq.remoteaction` prefix. It includes the rshell bundle, whose `runCommand` action runs shell commands through a restricted shell. See [Agent Restricted Shell (rshell)][8]
- **Script** actions are limited to *predefined* scripts declared in the runner's `script-config.yaml`. To configure script actions (`runPredefinedScript` for Linux or `runPredefinedPowershellScript` for Windows), see [Run a script with a private action runner][5].

{{% collapse-content title="Available actions by runner type" level="h3" %}}

{{< partial name="actions/private_actions_allowlist.html" >}}

{{% /collapse-content %}}

## Credential file formats

Some integrations, such as HTTP, Jenkins, PostgreSQL, MongoDB, and Temporal, require credentials to run. Credentials are supplied to the runner as JSON files that you reference from a [connection][6]. Each integration has its own credential file structure and supported authentication methods.

For the full set of credential file formats and examples, see [Handling private action credentials][7].

## Named runner credentials

The runner in the Agent can also read credentials from the Agent configuration. Define each credential by name under `private_action_runner.credentials.values`. A connection token that references a runner credential uses this name, and the runner substitutes the value when it prepares the task. To keep the value out of the configuration file, reference a secret with the `ENC[<secret_id>]` notation from [Agent secrets management][9].

Restrict each credential to the actions and HTTP destinations that need it. Otherwise, any allowed action whose connection references the credential name can use it.

{{< code-block lang="yaml" filename="datadog.yaml" disable_copy="false" >}}
private_action_runner:
  credentials:
    values:
      api_token:
        value: ENC[api_token]
        allowed_actions:
          - com.datadoghq.http.*
        allowed_http_domains:
          - api.example.com
      mongodb_password:
        value: ENC[mongodb_password]
        allowed_actions:
          - com.datadoghq.mongodb.*
{{< /code-block >}}

In this example, `api_token` works only for HTTP actions that send requests to `api.example.com`, and `mongodb_password` works only for MongoDB actions.

| Field | Required | Description |
|---|---|---|
| `value` | Yes | The credential value. Supports `ENC[<secret_id>]` secret references. |
| `allowed_actions` | No | Actions that can use the credential. Each entry is a fully qualified action name, such as `com.datadoghq.http.request`, or a trailing wildcard for an integration or sub-bundle, such as `com.datadoghq.mongodb.*`. Entries are case-sensitive. A wildcard for the whole `com.datadoghq` namespace isn't accepted. |
| `allowed_http_domains` | No | HTTP destinations the credential can be sent to. Each entry is a hostname, an IP address, or a subdomain pattern such as `*.example.com`. Entries are case-insensitive and can't include a scheme, port, or path. A subdomain pattern matches `api.example.com` but not `example.com`. Setting this field limits the credential to HTTP actions. |

### How restrictions are applied

- **Omitted, empty, and null lists**: Omit a restriction field to leave that dimension unrestricted. Set it to an empty list (`[]`) to block every action or destination. A `null` value or a non-list value is invalid configuration.
- **Multiple credentials**: When a connection references several named credentials, every referenced credential must allow the action and the destination.
- **Actions allowlist**: Credential restrictions narrow access and never expand it. The runner rejects actions that aren't in its `actions_allowlist` before it resolves any credentials.
- **HTTP destination**: The runner checks the hostname of the URL in the task (`http` or `https` only) for the `request` and `getChecksumFromURL` actions. For `testConnection`, it checks the connection's base URL combined with its test path. Hostnames are compared as written, without DNS resolution.
- **Failed checks**: If a check fails, the task fails before the runner sends any request with the credential.

<div class="alert alert-warning">Restrictions are checked when the runner resolves credentials for a task. <code>allowed_http_domains</code> applies to the initial request destination only, not to redirects the action follows. Restrictions don't change how the Agent loads or refreshes secrets.</div>

## Further reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /actions/private_actions/
[2]: https://github.com/DataDog/datadog-agent/blob/main/pkg/config/schema/yaml/private_action_runner.yaml
[3]: /actions/private_actions/enroll_runner/
[4]: /actions/private_actions/execution_policies/
[5]: /actions/private_actions/run_script/
[6]: /actions/connections/
[7]: /actions/connections/private_action_credentials/
[8]: /agent/guide/rshell/
[9]: /agent/configuration/secrets-management/
