You can embed git information in your Docker image using build arguments with environment variables, image labels, or image annotations.

###### Using build arguments and environment variables

1. Add the following lines to your application's Dockerfile:

   ```dockerfile
   ARG DD_GIT_REPOSITORY_URL
   ARG DD_GIT_COMMIT_SHA
   ENV DD_GIT_REPOSITORY_URL=${DD_GIT_REPOSITORY_URL} 
   ENV DD_GIT_COMMIT_SHA=${DD_GIT_COMMIT_SHA}
   ```

1. Add the following arguments to your Docker build command:

   ```shell
   docker build . \
    -t my-application \
    --build-arg DD_GIT_REPOSITORY_URL=<git-provider.example/me/my-repo> \
    --build-arg DD_GIT_COMMIT_SHA=$(git rev-parse HEAD)
   ```

###### Using image labels

This approach requires Docker, or containerd >= 1.5.6. It doesn't support containers running on AWS Fargate.

Datadog can extract source code information directly from your images' Docker labels. During build time, follow the [Open Containers standard][101] to add the git commit SHA and repository URL as Docker labels:

```shell
docker build . \
  -t my-application \
  --label org.opencontainers.image.revision=$(git rev-parse HEAD) \
  --label org.opencontainers.image.source=$(git config --get remote.origin.url)
```

###### Using image annotations

This approach requires containerd or CRI-O as the container runtime, and Datadog Agent v7.XX.0 or later (**writer: confirm the version**). It doesn't support containers running on AWS Fargate.

Datadog can extract source code information directly from your images' OCI annotations, without any Agent configuration. During build time, follow the [Open Containers standard][101] to add the git commit SHA and repository URL as image annotations:

```shell
docker build . \
  -t my-application \
  --annotation org.opencontainers.image.revision=$(git rev-parse HEAD) \
  --annotation org.opencontainers.image.source=$(git config --get remote.origin.url)
```

The Agent adds the `git.commit.sha` and `git.repository_url` tags to the image and to every container that uses it.

[101]: https://github.com/opencontainers/image-spec/blob/main/annotations.md#pre-defined-annotation-keys