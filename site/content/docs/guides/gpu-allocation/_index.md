---
title: GPU workloads
linkTitle: GPU workloads
weight: 10
description: Task-oriented walkthroughs for running GPU workloads.
---

Start with [GPU allocation](../../concepts/gpu-allocation.md) to choose a base
resource type: full GPU, MIG, or VFIO passthrough. Then use the corresponding
guide to request that resource.

Kubernetes DRA consumable capacity enables independent `ResourceClaim` objects to share a full GPU or MIG device with scheduler-managed capacity accounting.
Refer to [Consumable capacity](consumable-capacity.md) to configure memory-based, unlimited, or fixed-count shares.

Time-slicing and Multi-Process Service are optional in-claim sharing strategies.
Fabric Manager partitioning is an optional topology layer for full-GPU and
VFIO devices.
Refer to
[Fabric Manager partitioning](fabric-manager-partitioning.md) to select and
activate one complete NVSwitch partition.

For example manifests, refer to
[`demo/`](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/tree/{{< param driver_release_tag >}}/demo)
in the repository.

For alpha vGPU discovery, refer to [Advertise vGPU profiles](vgpu.md).
This feature publishes possible vGPU partitions for inspection and scheduling evaluation; the driver cannot prepare them for workloads.
