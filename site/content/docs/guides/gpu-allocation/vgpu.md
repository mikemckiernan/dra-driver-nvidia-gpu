---
title: Advertise vGPU profiles
linkTitle: vGPU advertisement
weight: 65
description: Enable alpha NVIDIA vGPU profile advertisement and verify the devices published in ResourceSlices.
---

The DRA Driver for NVIDIA GPUs can advertise NVIDIA virtual GPU (vGPU) profiles as devices in Kubernetes `ResourceSlice` objects.
Cluster administrators can use this alpha feature to evaluate discovery and scheduler-visible capacity on hosts running NVIDIA vGPU Manager.

{{% alert color="warning" title="Advertisement only" %}}
The driver does not create, prepare, or dispose of vGPU instances.
A claim allocated a vGPU partition fails during `NodePrepareResources` with the message "vGPU device lifecycle is not implemented yet."
Pods and virtual machines cannot use these advertised partitions through the driver.
{{% /alert %}}

## How advertisement works

The `VGPUSupport` feature gate is Alpha and disabled by default.
When enabled, the GPU kubelet plugin queries NVIDIA Management Library (NVML) for the vGPU types that each GPU can currently create.
It advertises only types whose full names appear in the administrator's `vgpu.profiles` allowlist.
An empty allowlist, the default, advertises no vGPU partitions.

Each advertised device represents a possible instance of one profile, rather than an existing vGPU.
The plugin publishes as many slot devices for a profile as NVML reports through its maximum-instance count.
A slot number distinguishes devices; it does not identify a virtual function or hardware placement.
The chart creates the `vgpu.gpu.nvidia.com` DeviceClass whenever the GPU resource plugin is enabled, even when `VGPUSupport` is disabled.
The presence of this DeviceClass alone does not indicate that vGPU devices are available.

### Capacity and profile compatibility

The driver uses Kubernetes DRA partitionable devices to describe the relationship between a physical GPU and its possible partitions:

- Each GPU with advertised vGPU profiles has a shared counter set named `vgpu-<minor>-counter-set`.
  Its `framebuffer` counter contains the GPU's total memory in bytes.
- Each vGPU slot consumes its profile's framebuffer size from that counter set.
  The scheduler limits allocations by both the available slot devices and the framebuffer budget.
- The full GPU remains advertised and consumes the entire counter set when allocated.
  This prevents the scheduler from allocating a full GPU together with a vGPU partition on the same GPU.

The optional `DRADeviceCompatibilityGroups` driver gate adds profile compatibility constraints to the same counter consumption entries.
The driver publishes a `vgpu-<shortProfileName>` group for each vGPU slot and a `gpu-full` group for the full GPU.
With the corresponding Kubernetes gate enabled, the scheduler permits slots with the same group to share a counter set and rejects combinations with different groups.
For example, `12q` and `24q` use different groups, even though both names end in `q`.
Without compatibility groups, framebuffer accounting alone does not prevent mixing profiles on the same GPU.

Enable the driver gate only when `DRADeviceCompatibilityGroups` is enabled on both the kube-apiserver and kube-scheduler.
The driver cannot discover the cluster's gate settings.
If the scheduler has this gate disabled, it ignores devices that declare compatibility groups.
Refer to the [feature gates reference](../../reference/feature-gates.md) for Kubernetes version requirements.

## Prerequisites

Before enabling advertisement, prepare the following environment:

- An evaluation cluster that meets the [driver prerequisites](../../prerequisites.md), with a Helm installation of the GPU resource plugin.
- A driver image and matching source checkout that include `VGPUSupport`.
  Configure the installed release to use that image, and run the commands below from the checkout's root directory.
  The procedure uses the local Helm chart rather than a published chart version.
- NVIDIA vGPU Manager installed on the GPU hosts, with GPUs that report creatable vGPU types through NVML.
  The driver does not install or configure vGPU Manager.
- MIG mode disabled on the GPUs you want to inspect for vGPU profiles.
  The driver skips vGPU discovery on GPUs with MIG mode enabled; MIG-backed vGPU advertisement is not implemented.
- Kubernetes `DRAPartitionableDevices` enabled on the kube-apiserver and kube-scheduler.
  On Kubernetes v1.34 and v1.35, enable it explicitly; v1.36 and later enable it by default.
  If you also want compatibility groups, use Kubernetes v1.37 or later and enable its `DRADeviceCompatibilityGroups` gate on both components.
  Refer to [Kubernetes feature gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/).
- `kubectl`, Helm, and `jq` on your administration workstation, with permission to update the driver release and list DeviceClasses and ResourceSlices.

`VGPUSupport` cannot be enabled together with `DynamicMIG` or `PassthroughSupport`.
The driver rejects these combinations at startup.

## Enable and verify advertisement

This procedure updates an existing Helm release and restarts its GPU kubelet plugin pods.
It assumes that the release and namespace are both named `dra-driver-nvidia-gpu`; replace these names if your installation uses different ones.

1. List the creatable profiles on each GPU host:

   ```bash
   nvidia-smi vgpu -c
   ```

   Record the full profile names that you want to advertise.
   Refer to [NVIDIA vGPU performance monitoring](https://docs.nvidia.com/vgpu/19.0/grid-vgpu-user-guide/performance-monitoring-gpu.html) for the creatable-profile query.
   Names depend on the GPU and host configuration.

2. Save the following overrides as `vgpu-values.yaml` on your administration workstation:

   ```yaml
   gpuResourcesEnabledOverride: true
   resources:
     gpus:
       enabled: true
   featureGates:
     VGPUSupport: true
     DynamicMIG: false
     PassthroughSupport: false
     DRADeviceCompatibilityGroups: false
   vgpu:
     profiles: "NVIDIA L40S-12Q,NVIDIA L40S-24Q"
   ```

   Replace the example profile names with the exact names reported by your hosts.
   The value is one comma-separated string, not a YAML list.
   Names are case-sensitive; the driver trims surrounding whitespace from each entry.
   Set `DRADeviceCompatibilityGroups: true` only if you enabled the matching Kubernetes gate on both control-plane components.

3. Apply the overrides with the matching local chart:

   ```bash
   helm upgrade dra-driver-nvidia-gpu ./deployments/helm/dra-driver-nvidia-gpu \
     --namespace dra-driver-nvidia-gpu \
     --reuse-values \
     --values vgpu-values.yaml \
     --wait --timeout 5m
   ```

   Helm retains the release's existing settings, including its driver image, and applies the overrides.
   The chart rejects a nonempty `vgpu.profiles` value unless `VGPUSupport` is enabled.

4. Inspect the vGPU DeviceClass:

   ```bash
   kubectl get deviceclass vgpu.gpu.nvidia.com -o yaml
   ```

   Its selector must match driver `gpu.nvidia.com` and device attribute `type == 'vgpu'`.

5. List the advertised vGPU slots:

   ```bash
   kubectl get resourceslices -o json | jq -r '
     .items[]
     | select(.spec.driver == "gpu.nvidia.com")
     | .spec.nodeName as $node
     | .spec.devices[]?
     | (.attributes // .basic.attributes) as $attrs
     | select($attrs.type.string == "vgpu")
     | [$node, .name, $attrs.profile.string, ($attrs.slot.int | tostring)]
     | @tsv'
   ```

   Each row contains the node, device name, full profile name, and slot number.
   Device names follow `vgpu-gpu-<minor>-<shortProfileName>-<slot>`, such as `vgpu-gpu-0-12q-0`.
   Expect rows only for allowlisted profiles that the host can currently create.
   If discovery reports multiple instances for a profile, expect one row per slot.

6. Inspect the shared counters and device consumption:

   ```bash
   kubectl get resourceslices -o yaml
   ```

   In slices with `spec.driver: gpu.nvidia.com`, verify that each GPU with vGPU slots has a `spec.sharedCounters` entry for its `vgpu-<minor>-counter-set` with a `framebuffer` budget.
   Each slot's `consumesCounters` entry must reference that counter set with its profile's framebuffer cost.
   The full GPU's entry must consume the entire budget.
   On Kubernetes v1.35 and later, the driver publishes counters and devices in separate ResourceSlices in the same pool, so inspect all slices for the node.
   If you enabled compatibility groups, also verify `compatibilityGroups` on the consumption entries.

## Troubleshoot advertisement

Use these checks when the published resources differ from your expectations:

| Symptom | Check |
| --- | --- |
| The DeviceClass exists but no vGPU slots appear. | Check that `VGPUSupport` is enabled, `vgpu.profiles` is nonempty, and the full names match creatable profiles on the host. Check that MIG mode is disabled. |
| A configured profile is missing. | The allowlist does not make an unsupported profile creatable. Check NVML discovery through the GPU kubelet plugin logs; a type with a maximum-instance count below one is skipped. |
| No slots appear for a GPU with unknown memory size. | The plugin skips vGPU enumeration when it cannot determine the GPU's framebuffer budget. Check its logs for "memory size unknown; skipping vGPU partition enumeration." |
| Profiles or host settings changed, but ResourceSlices did not. | Discovery runs at GPU kubelet plugin startup. Restart the affected plugin pods to discover the updated host state. |
| Grouped devices are not considered for allocation. | Check the Kubernetes `DRADeviceCompatibilityGroups` settings on the kube-apiserver and kube-scheduler against the driver's setting. |
| A vGPU claim fails to prepare. | This is the expected limit of advertisement support. The driver does not implement the vGPU device lifecycle. |

Refer to [Troubleshooting](../troubleshooting.md) for collecting driver logs and cluster events.

## Disable advertisement

Set `vgpu.profiles: ""` and `featureGates.VGPUSupport: false` in `vgpu-values.yaml`, then repeat the Helm upgrade command above.
The restarted plugin stops advertising vGPU slots.
Repeat the slot-listing query to verify that it returns no rows.
The `vgpu.gpu.nvidia.com` DeviceClass remains while the GPU resource plugin is enabled.
