## How to run operator-sdk scorecard in disconnected environment

#### Build scorecard-test using digests for docker.io/busybox:1.33.0 and registry.access.redhat.com/ubi8/ubi:8.4

Clone scorecard repo.

```console
$ git clone https://github.com/operator-framework/operator-sdk.git

```

Apply the following diff.

```diff
$ git diff
diff --git a/Makefile b/Makefile
index 3e8524b0..39235ada 100644
--- a/Makefile
+++ b/Makefile
@@ -93,7 +93,7 @@ DOCKER_PROGRESS = --progress plain
 endif
 image/%: export DOCKER_CLI_EXPERIMENTAL = enabled
 image/%:
-       docker buildx build $(DOCKER_PROGRESS) -t $(BUILD_IMAGE_REPO)/$*:dev -f ./images/$*/Dockerfile --load .
+       docker build $(DOCKER_PROGRESS) -t $(BUILD_IMAGE_REPO)/$*:dev -f ./images/$*/Dockerfile .
 
 ##@ Release
 
diff --git a/internal/scorecard/storage.go b/internal/scorecard/storage.go
index 3ec2e537..a57d6ddd 100644
--- a/internal/scorecard/storage.go
+++ b/internal/scorecard/storage.go
@@ -149,7 +149,7 @@ func addStorageToPod(podDef *v1.Pod, mountPath string) {
        // add the storage sidecar container
        storageContainer := v1.Container{
                Name:            StorageSidecarContainer,
-               Image:           "busybox",
+               Image:           "busybox@sha256:c71cb4f7e8ececaffb34037c2637dc86820e4185100e18b4d02d613a9bd772af",
                ImagePullPolicy: v1.PullIfNotPresent,
                Args: []string{
                        "/bin/sh",
diff --git a/internal/scorecard/testpod.go b/internal/scorecard/testpod.go
index 5862a83c..82b9388f 100644
--- a/internal/scorecard/testpod.go
+++ b/internal/scorecard/testpod.go
@@ -33,7 +33,7 @@ const (
 
        // The image used to untar bundles prior to running tests within a runner Pod.
        // This image tag should always be pinned to a specific version.
-       scorecardUntarImage = "registry.access.redhat.com/ubi8/ubi:8.4"
+       scorecardUntarImage = "registry.access.redhat.com/ubi8@sha256:910f6bc0b5ae9b555eb91b88d28d568099b060088616eba2867b07ab6ea457c7"
 )
```

Build the image, tag it and push.

```console
$ make image-build BUILD_IMAGE_REPO=my-private-registry/tkrishtop IMAGE_TARGET_LIST=scorecard-test
...
STEP 1: FROM golang:1.16 AS builder
✔ docker.io/library/golang:1.16
...
Successfully tagged my-private-registry/tkrishtop/scorecard-test:dev
e18fd5d07887af362f0b431deabe1bad02851f9cf1ec0de94dfe8f7e4ae19443

$ podman push my-private-registry/tkrishtop/scorecard-test:dev
```

#### How to run operator-sdk scorecard in disconnected environment

Now use previously built scorecard-test image in your scorecard config.

```
---
apiVersion: scorecard.operatorframework.io/v1alpha3
kind: Configuration
metadata:
  name: config
stages:
- parallel: true
  tests:
  - entrypoint:
    - scorecard-test
    - basic-check-spec
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: basic
      test: basic-check-spec-test
  - entrypoint:
    - scorecard-test
    - olm-bundle-validation
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: olm
      test: olm-bundle-validation-test
  - entrypoint:
    - scorecard-test
    - olm-crds-have-validation
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: olm
      test: olm-crds-have-validation-test
  - entrypoint:
    - scorecard-test
    - olm-crds-have-resources
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: olm
      test: olm-crds-have-resources-test
  - entrypoint:
    - scorecard-test
    - olm-spec-descriptors
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: olm
      test: olm-spec-descriptors-test
  - entrypoint:
    - scorecard-test
    - olm-status-descriptors
    image: my-private-registry/tkrishtop/scorecard-test:dev
    labels:
      suite: olm
      test: olm-status-descriptors-test
```

Mirror busybox and ubi8 in local registry

```console
$ skopeo copy --all \
    --remove-signatures \
    --authfile path/to/registry/auth \
    --dest-tls-verify=false \
    docker://registry.access.redhat.com/ubi8@sha256:910f6bc0b5ae9b555eb91b88d28d568099b060088616eba2867b07ab6ea457c7 \
    docker://my-private-registry/ubi8

$ skopeo copy --all \
    --remove-signatures \
    --authfile path/to/registry/auth \
    --dest-tls-verify=false \
    docker://docker.io/library/busybox@sha256:f7ca5a32c10d51aeda3b4d01c61c6061f497893d7f6628b92f822f7117182a57 \
    docker://my-private-registry/library/busybox
```

and setup imageContentSourcePolicy

```console
$ cat /path/to/imageContentSourcePolicy.yaml 
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
...
spec:
  repositoryDigestMirrors:
  - mirrors:
    - my-private-repository/ubi8
    source: registry.access.redhat.com/ubi8
  - mirrors:
    - my-private-repository/library/busybox
    source: docker.io/library/busybox
```

Now you're ready to run `operator-sdk scorecard` in disconnected environment. 
