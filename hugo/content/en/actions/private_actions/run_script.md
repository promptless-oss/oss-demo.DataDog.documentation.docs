---
title: Run a Script with the Private Action Runner
description: Use the private action runner to run predefined scripts in your private network, including the configuration required for ownerless, Execution-Policy-authorized runners.
further_reading:
- link: "/actions/private_actions/set_up_agent_based"
  tag: "Documentation"
  text: "Set up a private action runner in the Datadog Agent"
- link: "/actions/private_actions/execution_policies"
  tag: "Documentation"
  text: "Execution Policies"
- link: "/actions/private_actions/reference"
  tag: "Documentation"
  text: "Reference"
---

## Overview

The private action runner can run **predefined scripts**, which are shell commands, command-line tools, and scripts you declare ahead of time in a script configuration file. Only what you predefine can run, so the runner never runs arbitrary inline commands from a workflow or app.

The runner can also run scripts that Datadog writes and publishes. See the [Datadog-authored script actions](#datadog-authored-script-actions) section.

<div class="alert alert-warning">You decide which commands and binaries the runner is allowed to run. Review every command you add to the script configuration, especially those that accept parameters, grant the runner only the privileges it needs, and carefully review the permissions you share through connections. See <a href="/actions/connections/#connection-security-considerations">connection security considerations</a>.</div>

## Use cases

| Use case | Agent-based | Standalone | Notes |
|---|:---:|:---:|---|
| Running Linux binaries (`ls`, `rm`, `find`, `curl`) | {{< X >}} | {{< X >}} | For standalone runners, the relevant files must be accessible to the container. |
| Running CLIs (`aws`, `terraform`, `kubectl`) | {{< X >}} | {{< X >}} | For standalone runners, the CLI and credentials must be available in the image. For Agent-based runners, tools must be installed on the host. |
| Running bash scripts | {{< X >}} | {{< X >}} | For standalone runners, scripts can be mounted inside the container. Use the [large image](#large-image) for a Python interpreter. |
| Running PowerShell scripts | {{< X >}} | | Supported on Agent-based Windows runners only. |
| Running privileged commands (`systemctl restart`) | {{< X >}} | | For Agent-based runners, grant permissions to the runner user. Container sandboxing prevents standalone runners from privileged host access. |

## Prerequisites

**For Agent-based runners:**
- Datadog Agent 7.81.0 or later. See [Set up a private action runner in the Datadog Agent][1].
- Add `com.datadoghq.script.runPredefinedScript` (Linux) or `com.datadoghq.script.runPredefinedPowershellScript` (Windows) to the runner's actions allowlist.

**For standalone runners:**
- A standalone runner. See [Set up a standalone private action runner][2].
- For CLI tools not included in the base or [large image](#large-image), a custom Docker image. See [Custom images](#custom-images).

## Agent-based

### Configure scripts

{{< tabs >}}
{{% tab "Linux" %}}

Edit the `/etc/datadog-agent/private-action-runner/script-config.yaml` file:

```yaml
schemaId: script-credentials-v1
runPredefinedScript:
  echo:
    command: ["echo", "Hello world!"]
  echo-parametrized:
    command: ["echo", "{{ parameters.echoValue }}"]
  restart-service:
    command: ["sudo", "systemctl", "restart", "{{ parameters.service }}"]
```

{{% /tab %}}
{{% tab "Windows" %}}

Edit the `C:\ProgramData\Datadog\private-action-runner\powershell-script-config.yaml` file:

```yaml
schemaId: script-credentials-v1
runPredefinedPowershellScript:
  helloWorld:
    script: |
      Write-Output "Hello world!"
  greet:
    script: |
      Write-Output "Run script from workflow called {{ parameters.name }} !"
    parameterSchema:
      properties:
        name:
          type: string
      required:
        - name
  restartService:
    script: |
      Restart-Service -Name {{ parameters.serviceName }} -Force
    parameterSchema:
      properties:
        serviceName:
          type: string
      required:
        - serviceName
```

{{% /tab %}}
{{< /tabs >}}

In a workflow or app, reference a script by the name you defined (for example, `echo`). Use `runPredefinedScript` on Linux runners and `runPredefinedPowershellScript` on Windows runners.

### Grant permissions

{{< tabs >}}
{{% tab "Linux" %}}

The runner executes scripts as the `dd-agent` user. If your scripts require elevated permissions, grant them to the `dd-agent` user:

```bash
echo "dd-agent ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx" > /etc/sudoers.d/dd-agent
chmod 440 /etc/sudoers.d/dd-agent
```

{{% /tab %}}
{{% tab "Windows" %}}

The runner executes scripts as `ddagentuser`. If your scripts require access to certain resources, grant `ddagentuser` elevated permissions to them:

```powershell
icacls "C:\<your-file-path>" /grant "ddagentuser:(OI)(CI)RX" /T

# Verify permissions
icacls "C:\<your-file-path>"
```

{{% /tab %}}
{{< /tabs >}}

### Ownerless runner (Execution Policy-authorized)

When a runner is enrolled as ownerless and authorized by [Execution Policies][3], two things are required in addition to the steps above:

- The **Script** integration must be authorized for the runner through an Execution Policy, in addition to the predefined-script action being in the runner's actions allowlist.
- The runner reads its predefined scripts from a **fixed path**, the same path used in [Configure scripts](#configure-scripts) above:

{{< tabs >}}
{{% tab "Linux" %}}

`/etc/datadog-agent/private-action-runner/script-config.yaml`

{{% /tab %}}
{{% tab "Windows" %}}

`C:\ProgramData\Datadog\private-action-runner\powershell-script-config.yaml`

{{% /tab %}}
{{< /tabs >}}

#### Delivering the config on Kubernetes

On Kubernetes, provide the script configuration file to the runner in the Datadog Agent as a ConfigMap. Mount it into the runner container at the fixed path. The Cluster Agent runner uses the Linux path above.

First, create a ConfigMap that holds your script configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: par-script-config
  namespace: datadog
data:
  script-config.yaml: |
    schemaId: script-credentials-v1
    runPredefinedScript:
      echo:
        command: ["echo", "Hello world!"]
```

Then, on the `DatadogAgent` resource, allow the predefined-script action and mount the ConfigMap into the runner container at the fixed path:

```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
  annotations:
    agent.datadoghq.com/private-action-runner-enabled: "true"
    agent.datadoghq.com/private-action-runner-configdata: |
      private_action_runner:
        enabled: true
        api_key_only_enrollment: true
        actions_allowlist:
          - "com.datadoghq.script.runPredefinedScript"
          - "com.datadoghq.kubernetes.*"
          - "com.datadoghq.remoteaction.*"
spec:
  override:
    nodeAgent:
      volumes:
        - name: par-script-config
          configMap:
            name: par-script-config
      containers:
        private-action-runner:
          volumeMounts:
            - name: par-script-config
              mountPath: /etc/datadog-agent/private-action-runner/script-config.yaml
              subPath: script-config.yaml
              readOnly: true
```

Finally, apply the manifest:

```bash
kubectl apply -f datadog-agent.yaml
```

### Owned runner (Connection-based)

{{< tabs >}}
{{% tab "Linux" %}}

#### Configure the connection

If you selected `com.datadoghq.script.runPredefinedScript` in the runner's actions allowlist, you should already have a **Script** connection linked to your runner. Otherwise, create a connection and specify `/etc/datadog-agent/private-action-runner/script-config.yaml` as the **path to file**. For more information, see [Handling private action credentials][4].

{{% /tab %}}
{{% tab "Windows" %}}

#### Configure the connection

If you selected `com.datadoghq.script.runPredefinedPowershellScript` in the runner's actions allowlist, you should already have a **Script** connection linked to your runner. Otherwise, create a connection and specify `C:\ProgramData\Datadog\private-action-runner\powershell-script-config.yaml` as the **path to file**. For more information, see [Handling private action credentials][4].

{{% /tab %}}
{{< /tabs >}}

## Standalone

A standalone runner is always owned and authorized with [Connections][5].

{{< tabs >}}
{{% tab "Docker" %}}

1. After [setting up a runner][2], navigate to **Connections**.
1. Click **New Connection** and select **Script**.
1. Enter a connection name, and in the **Private Action Runner** dropdown, select your runner.
1. Copy the credential file template into your runner's configuration directory with the commands you want to run.
1. In **Path to file**, confirm the file path matches the path on your runner's file system (the default is sufficient in most cases).
1. Click **Next, Confirm Access**, configure permissions, then click **Create**.
1. Select this connection when using the script action in your workflows or apps.

Configure script actions through your runner's `config.yaml` file and the script connection
(`credentials/script.yaml` by default):

```yaml
# Add the script action to the allowlist (config.yaml)
actionsAllowlist:
  - com.datadoghq.script.runPredefinedScript
```

```yaml
# Configure your script connection (credentials/script.yaml)
schemaId: script-credentials-v1
runPredefinedScript:
  echo:
    command: ["echo", "Hello world"]
  echo-parametrized:
    command: ["echo", "{{ parameters.echoValue }}"]
    parameterSchema:
      properties:
        echoValue:
          type: string
      required:
        - echoValue
```

{{% /tab %}}
{{% tab "Kubernetes (Helm)" %}}

When deploying the runner with Helm, configure scripts through your `values.yaml` file:

```yaml
common:
  actionsAllowlist:
    - com.datadoghq.script.runPredefinedScript

credentials:
  script:
    schemaId: script-credentials-v1
    runPredefinedScript:
      echo:
        command: ["echo", "Hello world"]
      echo-parametrized:
        command: ["echo", "{{ parameters.echoValue }}"]
        parameterSchema:
          properties:
            echoValue:
              type: string
          required:
            - echoValue
```

Deploy or upgrade the runner:

```bash
helm upgrade --install <RELEASE_NAME> datadog/private-action-runner -f ./values.yaml
```

{{% /tab %}}
{{< /tabs >}}

### Runner image options

The following options are available for standalone runners only.

#### Large image

If you want to use tools like Python, SSH, the AWS CLI, Terraform, or the gcloud CLI, use the `gcr.io/datadoghq/private-action-runner:v{{< private-action-runner-version "private-action-runner" >}}-large` image instead of the default image.

#### Custom images

For binaries not available in the Datadog-provided images, create a custom image:

```dockerfile
FROM gcr.io/datadoghq/private-action-runner:v{{< private-action-runner-version "private-action-runner" >}}
USER root
# Change the line below to install the tool of your choice
RUN apt update && apt install -y python3
USER dog
```

You can mount complex scripts inside the runner:

```yaml
# docker-compose example
services:
  runner:
    build: . # if you are using a local Dockerfile
    volumes:
      - "./config:/etc/dd-action-runner/config" # contains credentials for actions
      - "./scripts:/etc/dd-action-runner-script/scripts" # contains dependencies for script actions
```

```yaml
# credentials/script.yaml
schemaId: script-credentials-v1
runPredefinedScript:
  python:
    command: ["python3", "/etc/dd-action-runner-script/scripts/script.py"]
  shell:
    command: ["bash", "/etc/dd-action-runner-script/scripts/script.sh"]
```

## Using the configured scripts

In your workflow or app, configure the action to use the script name you defined (for example, `echo` or `echo-parametrized`). For Linux runners, use `runPredefinedScript`. For Windows runners, use `runPredefinedPowershellScript`.

There are two levels of variable resolution: one at the workflow level and one at the action level
inside the runner.

## Datadog-authored script actions

<div class="alert alert-info">Datadog-authored script actions are in Preview.</div>

Datadog-authored script actions run scripts that Datadog writes and publishes, so you don't write or maintain a script configuration file for them. These actions use the `com.datadoghq.authoredscripts` prefix and run on Agent-based runners on Linux and Windows hosts.

When a workflow or app runs one of these actions, the runner:

1. Looks up the action's versioned package in the Datadog catalog, which the runner receives through Remote Configuration.
1. Downloads the package and keeps it in a local cache.
1. Runs the package's entrypoint in a temporary directory that the runner deletes after the run.

### Prerequisites

- An Agent-based runner on a Linux or Windows host. See [Set up a private action runner in the Datadog Agent][1]. Windows support requires Datadog Agent `<AGENT_VERSION>` or later.
- The authored-script actions in the runner's actions allowlist. Datadog-authored script actions are not part of the default allowlist.

### Allow Datadog-authored script actions

To allow every Datadog-authored script action, add `com.datadoghq.authoredscripts.*` to the `private_action_runner.actions_allowlist` section of `datadog.yaml`:

```yaml
private_action_runner:
  enabled: true
  actions_allowlist:
    - "com.datadoghq.authoredscripts.*"
```

To allow fewer actions, list each action's full name instead of the wildcard. To apply the change, restart the Agent as described in [Change the allowlist][6].

### How scripts run on Windows

On Windows, the runner applies these rules to Datadog-authored script actions:

- **Entrypoints**: The runner accepts packages whose entrypoint is a PowerShell script (`.ps1`) or a native executable (`.exe` or `.com`), and rejects any other entrypoint type.
- **PowerShell**: The runner starts `.ps1` entrypoints with Windows PowerShell from `%SYSTEMROOT%\System32\WindowsPowerShell\v1.0\powershell.exe`, using the `-NoProfile`, `-NonInteractive`, and `-ExecutionPolicy Bypass` options. The host's PowerShell execution policy doesn't block these scripts, and PowerShell profiles on the host don't run.
- **Files**: The runner caches packages in `C:\ProgramData\Datadog\run\dd-authored-script`. Each run gets its own temporary directory under `C:\ProgramData\Datadog\run`, and `USERPROFILE`, `HOME`, `APPDATA`, `LOCALAPPDATA`, `TEMP`, and `TMP` point inside that directory. Scripts can't read or keep state in the runner user's profile between runs.
- **Cancellation**: When an action times out or is canceled, the runner stops the script and every process the script started.

### Troubleshooting Datadog-authored script actions

**The host denies execution of the script**

Application control on the host can block the runner from starting the package's entrypoint. Check the following:

- On Windows, confirm that file permissions, AppLocker, and Windows Defender Application Control (WDAC) policies let the runner user run files from `C:\ProgramData\Datadog\run`, and let it start `powershell.exe`.
- On Linux, confirm that file permissions and application allowlisting tools such as `fapolicyd` let the runner user run files from its cache directory.

## Further reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /actions/private_actions/set_up_agent_based/
[2]: /actions/private_actions/set_up_standalone/
[3]: /actions/private_actions/execution_policies/
[4]: /actions/connections/private_action_credentials/
[5]: /actions/connections/
[6]: /actions/private_actions/set_up_agent_based/#change-the-allowlist
