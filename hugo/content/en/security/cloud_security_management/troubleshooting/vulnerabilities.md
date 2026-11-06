---
title: Troubleshooting Cloud Security Vulnerabilities
aliases:
  - /security/vulnerabilities/troubleshooting/
further_reading:
- link: "/infrastructure/containers/container_images/#enable-sbom-collection"
  tag: "Documentation"
  text: "Enable SBOM collection in Cloud Security Vulnerabilities"
- link: "/security/cloud_security_management/setup/csm_enterprise/?tab=aws#hosts"
  tag: "Documentation"
  text: "Setting up host vulnerabilities"
- link: "https://www.datadoghq.com/blog/datadog-container-image-view/"
  tag: "Blog"
  text: "Enhance your troubleshooting workflow with Container Images in Datadog Container Monitoring"
---

## Overview

If you experience issues with Cloud Security Vulnerabilities, use the following troubleshooting guidelines. If you need further assistance, contact [Datadog support][1].

## Error messages

### Disk space requirements

Before each container image scan, the Agent checks that its root file system has at least the free space set by `sbom.container_image.min_available_disk` (10 MB by default). If it doesn't, the scan fails with an error similar to the following:

```sh
failed to check current disk usage: not enough disk space to safely collect sbom, 5242880 available, 10485760 required
```

Sometimes the Agent can't read an image's layers in place and exports the image to a temporary file instead. Before the export, it checks that the file system of its temporary directory has 20% more free space than the size of the image. If it doesn't, the scan fails with an error similar to the following:

```sh
not enough disk space to export the image, 1073741824 available, 1288490188 required
```

Scans that read the image layers in place skip this second check.

Workaround:

- Increase the free disk space on the Agent's root file system. For exports, the file system of the Agent's temporary directory needs 20% more free space than your largest container image.
- To lower the minimum free space required before every scan, set `sbom.container_image.min_available_disk`, or the `DD_SBOM_CONTAINER_IMAGE_MIN_AVAILABLE_DISK` environment variable.
- To avoid exports, mount the container runtime's storage into the Agent container so the Agent can read the image layers in place. See [Enable SBOM collection][6].

### Uncompressed container image layers

The SBOM scan only works with uncompressed container image layers. Certain Kubernetes distributions (such as AWS EKS, minikube, and kind), configure their container runtime to discard the uncompressed layers, causing the scan to fail.

The resulting error appears as:

```sh
ERROR | (pkg/workloadmeta/collectors/internal/containerd/image_sbom_trivy.go:80 in func2) | Failed to generate SBOM for containerd image: unable to marshal report to sbom format, err: analyze error: failed to analyze layer:  : unable to get uncompressed layer
```

The workaround for this issue is to set the configuration option:
- For containerd: set `discard_unpacked_layers=false` in the containerd configuration file.
- For Helm: set `datadog.sbom.containerImage.uncompressedLayersSupport: true` in your `values.yaml` file.
- For Datadog Operator: set `features.sbom.containerImage.uncompressedLayersSupport` to `true` in your DatadogAgent CRD.

### GKE image streaming

Datadog doesn't support image streaming with Google Kubernetes Engine (GKE). If you have that option enabled in GKE, your Agent can't generate container SBOMs.

The resulting error appears as:

```sh
unable to mount containerd image, err: unable to scan image named: {image-name}, image is not unpacked
```

The workaround for this issue is to disable image streaming in GKE. For more information, see the [Disable Image streaming][5] section of the GKE docs.

## Disable Cloud Security Vulnerabilities

In the `datadog-values.yaml` file for the Agent, set the following configuration settings to `false`:

```
# datadog-values.yaml file
datadog:
  sbom:
    containerImage:
      enabled: false

      # Uncomment the following line if you are using Google Kubernetes Engine (GKE) or Amazon Elastic Kubernetes (EKS)
      # uncompressedLayersSupport: true

    # Enables Host Vulnerability Management
    host:
      enabled: false

    # Enables Container Vulnerability Management
    # Image collection is enabled by default with Datadog Helm version `>= 3.46.0`
      containerImageCollection:
        enabled: false
```

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /help/
[2]: /security/cloud_security_management/setup/csm_enterprise?tab=aws#configure-the-agent-for-vulnerabilities
[3]: https://app.datadoghq.com/security/configuration/csm/setup
[4]: https://app.datadoghq.com/metric/summary
[5]: https://cloud.google.com/kubernetes-engine/docs/how-to/image-streaming#disable
[6]: /containers/monitoring/container_images/#enable-sbom-collection
