---
# Post title - will be auto-generated from filename if not changed
title: "Your AI Environment as an Artifact: NVIDIA AICR with Private Data Packs"

# Publication date - automatically set to current date/time
date: 2026-09-08T00:00:00Z

# Author name - replace with your name
author: "Satyam Bhardwaj"

# keywords - replace with your keywords for SEO and post listings
keywords:
  - kubernetes
  - NVIDIA AICR
  - GPU Operator
  - k0s
  - H200
  - OCI artifact
  - SBOM
  - in-toto attestation
  - cluster validation
  - GPU fleet

# Tags for categorizing content (e.g., automation, mlops, devops, aiops)
tags: ["kubernetes", "gpu", "gitops", "validation", "supply-chain"]

# Categories for broader grouping (e.g., engineering, operations, tutorials)
categories: ["engineering", "operations"]

# Set to false when ready to publish
draft: false

# Brief description/summary of the post (recommended for SEO and post listings)
description: "Capture your NVIDIA GPU stack as a named, versioned AICR data pack, converge clusters onto it declaratively with the nvidia-aicr Helm chart, and keep signed evidence that the platform matches the contract."

# URL slug (optional) - overrides the filename for the URL
slug: "nvidia-aicr"

# Featured image path (optional) - place images in assets/images/<your-post-name>/
image: "images/2026/nvidia-aicr-data-packs/header.png"
---

Standing up an NVIDIA AI stack on Kubernetes takes more than installing a
driver. Before a single GPU pod runs, several components need to be in place:

- **GPU driver** - the kernel driver, whether host-installed or container-delivered
- **NVIDIA GPU Operator** - orchestrates driver, device plugin, and validation
- **Container toolkit** - wires the `nvidia` runtime into the container engine
- **DRA driver** - Dynamic Resource Allocation for fine-grained GPU requests
- **Scheduler extensions** - topology- and MIG-aware placement
- **Node Feature Discovery (NFD)** - labels nodes by GPU model and capability
- **Monitoring** - DCGM exporter and the metrics pipeline that reads it

Each has its own version and its own ordering quirks, and they tend to fail
silently when a sibling is off by one minor release: pods sit `Pending`,
NCCL runs at a fraction of fabric bandwidth, or DCGM reports nothing because
the driver root doesn't match - all while the cluster itself reports healthy.

Even when everything works, the setup is hard to reproduce. The reasons
behind a specific toolkit env var or driver setting usually live in a
runbook, or in the head of whoever did the install.

[NVIDIA AI Cluster Runtime (AICR)](https://github.com/NVIDIA/aicr) addresses
both problems. Its validated recipes cover the first: the stack assembles
from a tested, version-locked combination. Its extension mechanism covers
the second: your environment's specifics become a **named, versioned, private
data pack** that AICR resolves like any built-in recipe. This post is about
that second capability and how to deliver the whole loop declaratively
with the [nvidia-aicr](https://github.com/Mirantis/nvidia-aicr-chart) Helm
chart.

## What AICR Gives You

AICR's central idea is the **recipe**: a version-locked, dependency-ordered
description of every component that should land on a cluster to support a
given GPU workload. You state intent across five criteria dimensions -
`service`, `accelerator`, `intent`, `os`, `platform` - and AICR resolves it
against a catalog of validated combinations.

AICR's capabilities are:

1. **A validated matrix.** One invocation converges a cluster on
   NVIDIA's tested combination of driver, operators, scheduler, and
   monitoring. The recipe describes the target state; the rendered bundle
   is what actually gets deployed.
2. **Recipes as data, not code.** A `--data` extension pack can register its
   own criteria values.
3. **Validation and evidence.** `aicr validate` runs assertive checks against
   the live cluster and emits a CTRF verdict; `--emit-attestation` produces
   an in-toto evidence bundle with a CycloneDX SBOM that can be signed and
   pushed to an OCI registry.

AICR itself is a CLI, but fleets are usually managed by reconcilers that
speak Helm, watch git, and converge clusters toward declared state. The
[nvidia-aicr](https://github.com/Mirantis/nvidia-aicr-chart) chart packages
the CLI workflow as a single Kubernetes Job that:

- downloads a pinned, checksum-verified `aicr` binary
- pulls your data pack
- resolves the recipe
- optionally deploys the bundle
- runs validation
- captures every result to ConfigMaps
- optionally publishes the evidence bundle

## The Gap a Pack Closes

Here is a situation the built-in catalog cannot describe, taken from the
machine we'll use for the rest of this post: a bare-metal k0s cluster with
H200 GPUs. Upstream AICR has no `service` value for on-prem bare-metal
Kubernetes. On top of that, `intent: training` upstream
requires a concrete managed service - of `{aks, bcm, eks, lke, ocp, gke,
oke}`, none describe this machine.

## Authoring the Pack

A pack is a directory with a required `registry.yaml` (component additions -
an empty stub if you add none) and an `overlays/` directory. The one shipped
in the chart repository, [`packs/k0s-h200-training/`](https://github.com/Mirantis/nvidia-aicr-chart/tree/main/packs/k0s-h200-training),
registers a private `service: k0s` value and pins every environment fact to
it. The overlay's skeleton:

```yaml
apiVersion: aicr.run/v1alpha2
kind: RecipeMetadata
metadata:
  name: k0s-h200-training
spec:
  criteria:
    service: k0s          # private value - registers on load
    accelerator: h200     # upstream-embedded value
    intent: training      # valid because THIS overlay declares the chain
  constraints:
    - name: K8s.server.version
      value: ">= 1.34"
  componentRefs:
    - name: gpu-operator
      type: Helm
      overrides:
        driver:
          enabled: false            # host carries the driver
        toolkit:
          enabled: true             # ...but NOT the toolkit
          env:
            - name: CONTAINERD_CONFIG
              value: /etc/k0s/containerd.d/nvidia.toml
            - name: CONTAINERD_SOCKET
              value: /run/k0s/containerd.sock
            - name: CONTAINERD_RUNTIME_CLASS
              value: nvidia
    - name: nvidia-dra-driver-gpu
      type: Helm
      overrides:
        nvidiaDriverRoot: /         # host-installed driver userspace
    - name: nvsentinel
      type: Helm
      overrides:
        labeler:
          assumeDriverInstalled: true
```

k0s runs its *own* containerd. Without the three toolkit env vars,
the container toolkit configures the default containerd and reports success,
but GPU pods fail with `no runtime for "nvidia" is configured`. The
driver-ownership trio (`driver.enabled: false`, `nvidiaDriverRoot: /`,
`assumeDriverInstalled: true`) must move together. AICR's coherence checks
fail closed if only one side is set.

And `registry.yaml`, when you add no components of your own:

```yaml
apiVersion: aicr.run/v1alpha2
kind: ComponentRegistry
components: []
```

Publish the directory as an OCI artifact:

```bash
( cd packs/k0s-h200-training && \
  oras push ghcr.io/<org>/aicr-packs/k0s-h200-training:0.1.0 . )
```

From here on, `service: k0s` works as a first-class criteria value on every
cluster that loads the pack, and the environment's specifics are versioned
and distributable instead of sitting in a runbook.

## Running It

```yaml
# values-k0s-h200.yaml
service: k0s
accelerator: h200
intent: training
dataPack: "ghcr.io/<org>/aicr-packs/k0s-h200-training:0.1.0"
dataPackSecret: "aicr-pack-pull"   # existing dockerconfigjson Secret

deploy:
  enabled: true
validate:
  enabled: true
  phases: [deployment]
```

```bash
helm install nvidia-aicr oci://ghcr.io/mirantis/charts/nvidia-aicr \
  --version 0.2.0 \
  --namespace aicr --create-namespace \
  -f values-k0s-h200.yaml
```

Because the chart is a normal OCI Helm chart, any reconciler that speaks
Helm delivers any of these - a Flux `HelmRelease`, an Argo CD `Application`,
a k0rdent `MultiClusterService` - all consuming the same values block. The
pack keeps the values block small, and the reconciler applies it across the
fleet.

When the Job completes, read what actually resolved. This output is from
the real run on our H200 box.

```bash
kubectl get cm aicr-recipe -n aicr -o jsonpath='{.data.summary\.txt}'
```

```text
criteria: service=k0s accelerator=h200 os= intent=training platform=
dataPack: ghcr.io/ramessesii2/aicr-packs/k0s-h200-training:0.1.0
recipe generation completed: output=./recipe.yaml components=11 componentNames=cert-manager, gpu-operator, k8s-ephemeral-storage-metrics, kai-scheduler, kube-prometheus-stack, nfd, nodewright-operator, nvidia-dra-driver-gpu, nvsentinel, prometheus-adapter, prometheus-operator-crds overlays=4
```

This ConfigMap matters because AICR resolves exactly what you state - leave
a criterion out and you get a thinner stack with no warning. The summary
shows the criteria used, the overlay count, and the sorted component list,
so there is a record of what actually resolved.

## Validation and Evidence

The same run captured a verdict, because `validate.enabled: true` ran the
`deployment` phase against the live cluster - is every recipe component
present, healthy, and versioned as the contract says?

```bash
kubectl get cm aicr-validate-result -n aicr -o jsonpath='{.data.ctrf\.json}' \
  | jq '.results.summary, (.results.tests[] | {name, status})'
```

```json
{"tests": 4, "passed": 4, "failed": 0, "skipped": 0, "pending": 0, "other": 0}
{"name": "operator-health",     "status": "passed"}
{"name": "expected-resources",  "status": "passed"}
{"name": "gpu-operator-version","status": "passed"}
{"name": "check-nvidia-smi",    "status": "passed"}
```

Four checks, four passes: the GPU operator is running, every one of the 11
recipe components exists and holds its health assertions stable for a minute,
the operator version matches the recipe constraint, and `nvidia-smi`
answers correctly on the GPU node itself.

A failed validation is captured either way. By default the Job still
succeeds (validation is informative); set `validate.failOnError: true` to
make the verdict gate the Job.

For runs you may need to prove later, enable evidence:

```yaml
validate:
  emitEvidence: true
  publish:
    enabled: true
    mode: unsigned
    ref: ghcr.io/<org>/aicr-evidence
    registrySecret: "aicr-pack-pull"
```

In `unsigned` mode, identity never enters the cluster. The bundle - CTRF
verdict, recipe, snapshot, CycloneDX SBOM, in-toto statement - is pushed
with an empty signer block and signed later from a trusted host with
`aicr evidence sign <pointer> --relocate`. In `signed` mode a short-lived
OIDC token drives in-cluster Sigstore keyless signing. Both are one flag
apart; the chart's values document the trade-offs plainly (the token lives
in a cluster-admin-bound pod, and it must still be valid when the publish
step finally runs). One robot token, one Secret, can serve the pack pull,
the validator image pulls, and the evidence push.

The run above did exactly that. The pointer the chart captured, the durable
locator for the pushed bundle, reads:

```bash
kubectl get cm aicr-evidence-bundle -n aicr -o jsonpath='{.data.pointer\.yaml}'
```

```yaml
attestations:
  - attestedAt: 2026-08-31T10:29:48Z
    bundle:
      digest: sha256:bfc722368c6917ccd59eb9e1b8f43863969a98e2f37a93ec4535903103ebfee1
      oci: ghcr.io/ramessesii2/aicr-evidence:h200-k0s-training-fd111e56291b
      predicateType: https://aicr.run/recipe-evidence/v1
recipe: h200-k0s-training
schemaVersion: 1.0.0
```

Because the reference is digest-pinned, anyone with pull access can fetch
the bundle, verify the manifest hash chain, and, once it's signed from a
trusted host, verify the signer identity - without touching the cluster
that produced it.

## Supply Chain

The Job verifies the pinned release tarball's sha256 against the release's
checksums file before executing anything; `aicrSha256.<arch>` optionally
anchors trust in reviewed chart values instead of runtime-fetched assets.
Before executing the rendered bundle, the Job runs `aicr verify` - AICR's
closed-world checksum gate: any file added, removed, or modified between
bundle and deploy fails the run. The pod runs nonroot (UID 65532) with a
read-only root filesystem. The pack pull is fail-closed: a private registry
with no credential is an error.

## Conclusion

At the end of this loop, the GPU stack is described by a named, versioned
artifact: AICR resolves the recipe, the chart deploys it, validation
confirms the running platform matches it, and the evidence bundle records
that verdict in a registry. The environment's specifics are written down in
the pack rather than carried as tribal knowledge, and the same values block
reproduces the setup on any cluster in the fleet.

What this does not cover is the hardware itself: whether every GPU, NVLink,
and fabric path holds up under hours of sustained load, and which node is at
fault when one doesn't. Platform validation and hardware burn-in are
separate problems with separate tooling, and we'll look at the burn-in side
in a follow-up post.

## Try It

The chart is [published as an OCI artifact](https://github.com/Mirantis/nvidia-aicr-chart).
Start with the README's day-1 example, then author a pack for your org - the
[`k0s-h200-training` pack](https://github.com/Mirantis/nvidia-aicr-chart/tree/main/packs/k0s-h200-training)
is a complete, annotated template. From there, delivery works through
whatever reconciler you already run.
