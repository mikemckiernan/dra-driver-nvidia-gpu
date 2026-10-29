---
title: Run virtual machines with vGPUs
linkTitle: vGPU workloads
weight: 65
description: Configure vGPU profiles, request a vGPU for a KubeVirt virtual machine, and verify GPU access in the guest.
---

Use NVIDIA virtual GPU (vGPU) software with the DRA Driver for NVIDIA GPUs to share a physical GPU across virtual machines (VMs).
Cluster administrators choose the vGPU profiles available on GPU nodes; workload authors request a profile through a Kubernetes `ResourceClaim` and attach the allocation to a KubeVirt VM.
This guide covers configuration, VM startup, verification, and cleanup.

{{% alert title="Feature status" %}}
vGPU support is an Alpha feature controlled by the `VGPUSupport` feature gate, which is disabled by default.
{{% /alert %}}

## When to consider vGPU

In a supported virtualization deployment, NVIDIA vGPU software enables multiple VMs to share one physical GPU, with a separate vGPU device assigned to each VM.
Sharing a GPU across VMs can improve utilization when individual workloads leave much of a full GPU idle.
Profiles define framebuffer (GPU memory) sizes so administrators can assign GPU memory according to workload needs.
For example, several development or light inference VMs might share a GPU instead of each reserving a full device.
Refer to the [NVIDIA vGPU for Compute overview](https://docs.nvidia.com/ai-enterprise/release-7/latest/infra-software/vgpu/overview.html) for architecture and sharing modes.

Choose an approach according to the workload's requirements:

| Requirement | Approach to evaluate |
| --- | --- |
| Share a GPU across independent VMs whose workloads meet performance requirements with shared compute time. | Time-sliced vGPU can increase the number of VMs served by one GPU. Measure performance with the expected number of active VMs. |
| Give a workload exclusive access to the full GPU's resources. | Exclusive full-GPU allocation reserves the GPU's compute resources for one workload. Refer to [GPU allocation](../../concepts/gpu-allocation.md) for container access and VFIO passthrough options. |
| Give container workloads dedicated compute and memory partitions on supported hardware. | [MIG](mig.md) provides hardware partitioning. NVIDIA vGPU software also supports MIG-backed configurations, but this guide covers time-sliced vGPU profiles on GPUs with MIG mode disabled. |

A time-sliced vGPU profile's framebuffer size does not imply dedicated compute engines or MIG-style performance isolation.
Workloads share GPU execution time, so evaluate throughput and latency under contention before choosing profile sizes and VM density.

Adoption also requires a supported combination of GPU, hypervisor, guest operating system, vGPU Manager, and guest driver, plus the applicable software license.
Check the [NVIDIA vGPU software documentation](https://docs.nvidia.com/vgpu/latest/) and [licensing requirements](https://docs.nvidia.com/vgpu/latest/grid-licensing-user-guide/intro-to-grid-licensing.html) for your deployment.
Choose a profile that meets the application's memory and feature requirements before creating the VM.
A profile name alone does not establish compatibility with the application or guest operating system.

## How vGPU allocation works

With `VGPUSupport` enabled, the GPU kubelet plugin queries NVIDIA Management Library (NVML) for the vGPU types that each GPU can currently create.
The plugin advertises only types whose full names appear in the administrator's `vgpu.profiles` allowlist.
An empty allowlist, the default, advertises no vGPU partitions.

Each advertised device is a vGPU slot that represents capacity to create one instance of a specific vGPU profile on a physical GPU.
A vGPU slot does not represent an existing vGPU instance.
The plugin publishes as many slot devices for a profile as the maximum-instance count that NVML reports for that profile.
A slot number distinguishes devices; it does not identify a virtual function or hardware placement.
The chart creates the `vgpu.gpu.nvidia.com` DeviceClass whenever the GPU resource plugin is enabled, even when `VGPUSupport` is disabled.
The presence of this DeviceClass alone does not indicate that vGPU devices are available.

After Kubernetes allocates a slot to a claim, the GPU kubelet plugin creates and prepares the vGPU instance on the selected node.
KubeVirt attaches the prepared device to the VM, where the NVIDIA vGPU guest driver provides GPU access to applications.
Deleting the consuming VM and releasing its claim returns the allocation's capacity for subsequent workloads.

### Capacity and profile compatibility

The driver uses Kubernetes DRA partitionable devices to describe how vGPU slots share a physical GPU's capacity:

- Each GPU with advertised vGPU profiles has a shared counter set named `vgpu-<minor>-counter-set`.
  The counter set's `framebuffer` counter contains the GPU's total memory in bytes.
- Each vGPU slot consumes its profile's framebuffer size from that counter set.
  The scheduler limits allocations by both the available slot devices and the framebuffer budget.
- The full GPU remains advertised and consumes the entire counter set when allocated.
  This prevents the scheduler from allocating a full GPU together with a vGPU partition on the same GPU.

The optional `DRADeviceCompatibilityGroups` driver gate adds profile compatibility constraints to the same counter consumption entries.
The driver publishes a `vgpu-<shortProfileName>` group for each vGPU slot and a `gpu-full` group for the full GPU.
With the corresponding Kubernetes gate enabled, the scheduler permits slots with the same group to share a counter set and rejects combinations with different groups.
For example, `12q` and `24q` use different groups, even though both names end in `q`.
For compatibility groups, set `DRADeviceCompatibilityGroups=true` on both the kube-apiserver and kube-scheduler, then set `featureGates.DRADeviceCompatibilityGroups=true` in the driver's Helm values.
For clusters using framebuffer capacity tracking alone, configure one profile in `vgpu.profiles`.
Kubernetes tracks GPU memory usage, and the single-profile configuration keeps allocations compatible.
Refer to the [feature gates reference](../../reference/feature-gates.md) for Kubernetes version requirements.

## Prerequisites

Before configuring vGPU workloads, prepare the following environment:

- A cluster that meets the [driver prerequisites](../../prerequisites.md), with the DRA Driver's GPU resource plugin installed.
  Follow the [installation guide](../../install.md) for your driver deployment's installation method.
- NVIDIA vGPU Manager on the GPU nodes, with the configured vGPU profiles reported as available to create.
  The driver does not install or configure vGPU Manager.
- A supported combination of GPU, host operating system, NVIDIA vGPU Manager, and guest driver, with the applicable NVIDIA software license.
  Prepare a Linux VM disk image containing the matching vGPU guest driver and the tools needed to verify licensing.
  Refer to the [NVIDIA vGPU software documentation](https://docs.nvidia.com/vgpu/latest/) for compatibility and setup requirements.
- MIG mode disabled on the physical GPUs used for this workflow.
- Kubernetes v1.34.2 or later with `DRAPartitionableDevices` enabled on the kube-apiserver and kube-scheduler.
  On v1.34 and v1.35, enable the feature explicitly; v1.36 and later enable the feature by default.
  The `DRAPartitionableDevices` feature enables Kubernetes to track how vGPU allocations consume a physical GPU's shared capacity.
  For optional profile compatibility groups, use Kubernetes v1.37 or later and enable `DRADeviceCompatibilityGroups` on both components.
- KubeVirt v1.8.0 or later with `GPUsWithDRA` enabled.
  KubeVirt v1.8.x requires explicit enablement; v1.9.0 and later enable the gate by default.
  Refer to [Assigning GPUs with DRA](https://kubevirt.io/user-guide/compute/dra_gpu/) for the KubeVirt configuration.
- `kubectl`, Helm, `jq`, and `virtctl` on your administration workstation.
  You need permission to update the driver configuration, list DeviceClasses and ResourceSlices, and create claims and VMs in the workload namespace.

`VGPUSupport` cannot be enabled together with `DynamicMIG` or `PassthroughSupport`.
The driver rejects these combinations at startup.

## Enable vGPU support

For a direct Helm installation, update the existing driver release with the values below.
The update restarts the GPU kubelet plugin pods.
For a GPU Operator-managed deployment, apply equivalent settings through the Operator configuration rather than managing the release separately.
The commands use `dra-driver-nvidia-gpu` for both the release and namespace names; replace these names if your installation uses different ones.

For deployments that configure the GPU kubelet plugin directly, use `--vgpu-profiles` or the `VGPU_PROFILES` environment variable with the same comma-separated string as `vgpu.profiles`.
Enable `VGPUSupport` in the driver's feature gates when specifying profiles.

1. List the vGPU profiles available to create on each GPU host:

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
     DeviceMetadata: true
     DynamicMIG: false
     PassthroughSupport: false
     DRADeviceCompatibilityGroups: false
   vgpu:
     profiles: "NVIDIA L40S-12Q"
   ```

   Replace the example profile names with the exact names reported by your hosts.
   The value is one comma-separated string, not a YAML list.
   Names are case-sensitive; the driver trims surrounding whitespace from each entry.
   `DeviceMetadata` supplies device metadata that KubeVirt needs to attach the allocation to the VM.
   This example advertises one profile to avoid mixing profiles on a physical GPU without compatibility groups.
   Set `DRADeviceCompatibilityGroups: true` only if you enabled the matching Kubernetes gate on both control-plane components.

3. Apply the overrides to the driver release:

   ```bash
   helm upgrade dra-driver-nvidia-gpu oci://registry.k8s.io/dra-driver-nvidia/charts/dra-driver-nvidia-gpu \
     --version {{< param "driver_version" >}} \
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

   The DeviceClass selector must match driver `gpu.nvidia.com` and device attribute `type == 'vgpu'`.

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

## Request a vGPU and start a VM

This example creates one VM and requests one vGPU with the `NVIDIA L40S-12Q` profile.
Use a profile advertised by your cluster, and use a guest image prepared as described in [Prerequisites](#prerequisites).
Run the Kubernetes and `virtctl` commands on your administration workstation.

1. Create a namespace for the workload:

   ```bash
   kubectl create namespace vgpu-demo
   ```

2. Save the following claim template as `vgpu-claim-template.yaml`:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: ResourceClaimTemplate
   metadata:
     name: vgpu-profile
     namespace: vgpu-demo
   spec:
     spec:
       devices:
         requests:
         - name: vgpu
           exactly:
             deviceClassName: vgpu.gpu.nvidia.com
             allocationMode: ExactCount
             count: 1
             selectors:
             - cel:
                 expression: device.attributes['gpu.nvidia.com'].profile == 'NVIDIA L40S-12Q'
   ```

   Replace `NVIDIA L40S-12Q` with the full name of your chosen profile.
   The DeviceClass selects vGPU slots; the expression narrows the request to one profile.
   Kubernetes chooses the physical GPU and slot, so the claim does not need a slot number or a parent GPU UUID.

3. Apply the claim template:

   ```bash
   kubectl apply -f vgpu-claim-template.yaml
   ```

   The template defines the request; the template alone does not allocate a vGPU.

4. Save the following VM instance manifest as `vgpu-vm.yaml`:

   ```yaml
   apiVersion: kubevirt.io/v1
   kind: VirtualMachineInstance
   metadata:
     name: vgpu-vm
     namespace: vgpu-demo
   spec:
     resourceClaims:
     - name: gpu
       resourceClaimTemplateName: vgpu-profile
     domain:
       cpu:
         cores: 2
       resources:
         requests:
           memory: 4Gi
       devices:
         disks:
         - name: guest-disk
           disk:
             bus: virtio
         interfaces:
         - name: default
           masquerade: {}
         gpus:
         - name: vgpu0
           claimName: gpu
           requestName: vgpu
     networks:
     - name: default
       pod: {}
     volumes:
     - name: guest-disk
       containerDisk:
         image: YOUR_REGISTRY/vgpu-guest:YOUR_TAG
   ```

   Replace `YOUR_REGISTRY/vgpu-guest:YOUR_TAG` with a container disk image containing your prepared Linux guest and vGPU driver.
   Use the login credentials configured in that image when connecting to the guest.
   The `resourceClaimTemplateName` references the claim template, `claimName` references the VM's claim entry, and `requestName` references the template's `vgpu` request.
   Refer to [KubeVirt DRA GPU assignment](https://kubevirt.io/user-guide/compute/dra_gpu/) for those field relationships.

5. Start the VM instance:

   ```bash
   kubectl apply -f vgpu-vm.yaml
   ```

   KubeVirt creates a claim from the template.
   Kubernetes allocates a matching slot, the driver prepares the vGPU instance, and KubeVirt attaches the device to the guest.

6. Wait for the VM instance to become ready:

   ```bash
   kubectl wait --for=condition=Ready vmi/vgpu-vm --namespace vgpu-demo --timeout=5m
   ```

   A successful wait confirms VM readiness; verify guest GPU access in the next procedure.

## Verify allocation and GPU access

Verify both the Kubernetes allocation and the device available inside the guest.
A scheduled VM alone does not prove that the guest driver or application can use the vGPU.

1. Inspect the claim generated for the VM:

   ```bash
   kubectl get resourceclaims --namespace vgpu-demo -o yaml
   ```

   Under `status.allocation.devices.results`, verify the driver is `gpu.nvidia.com` and the allocated device follows the `vgpu-gpu-<minor>-<shortProfileName>-<slot>` naming pattern.
   Match the allocated pool and device name to a published ResourceSlice and verify that the device's `profile` attribute matches the requested profile.
   Record the generated claim name for troubleshooting.

2. Connect to the guest console:

   ```bash
   virtctl console vgpu-vm --namespace vgpu-demo
   ```

   Log in with the credentials configured in the guest image.

3. Inspect the GPU from inside the guest:

   ```bash
   nvidia-smi --query-gpu=name,memory.total,uuid --format=csv
   ```

   Verify that `nvidia-smi` lists a vGPU and that the GPU memory is consistent with the selected profile, accounting for reserved memory.
   The ResourceSlice `uuid` identifies the parent physical GPU; do not use that value as the expected guest vGPU UUID.
   Confirm the guest's license status using the instructions for your NVIDIA vGPU software release.

4. Run the application inside the guest and verify its GPU operation.

   Use the application's device-selection and diagnostic tools to confirm GPU use, then check the workload's throughput and latency with the expected number of active VMs.
   Guest device visibility verifies attachment; successful application execution verifies the workload experience.

## Troubleshoot vGPU workloads

Use the workload state to identify the failing stage:

| Symptom | Check |
| --- | --- |
| The DeviceClass exists but no vGPU slots appear. | Check `VGPUSupport`, the `vgpu.profiles` allowlist, the profiles available on the GPU host, and the GPU's MIG mode. The DeviceClass alone does not establish device availability. |
| A profile is missing from ResourceSlices. | The allowlist does not make a profile available on the host. Check the full profile name, the host's vGPU configuration, and the GPU kubelet plugin's NVML discovery logs. |
| A VM remains pending and its claim is unallocated. | Inspect VM and launcher pod events. Check the requested profile, available framebuffer capacity, profile compatibility groups, node constraints, and taints. |
| A VM remains pending after enabling compatibility groups in the driver. | Verify `DRADeviceCompatibilityGroups=true` on both the kube-apiserver and kube-scheduler. A scheduler with this gate disabled excludes devices that declare compatibility groups from allocation. |
| A claim is allocated but the VM does not start. | Inspect the GPU kubelet plugin logs and launcher pod events for preparation or device attachment errors. Check host vGPU setup, `DeviceMetadata`, and KubeVirt's DRA configuration. |
| The guest boots but `nvidia-smi` cannot access a GPU. | Check vGPU attachment, the installed guest driver, and compatibility between the guest driver and vGPU Manager. |
| `nvidia-smi` lists a GPU but application capabilities or performance differ from expectations. | Check license status, profile-specific feature support, memory requirements, and contention from other active VMs. |
| Host profiles changed but ResourceSlices did not. | Restart the affected GPU kubelet plugin pods to refresh discovery after the host configuration change. |

To inspect the GPU kubelet plugin logs, run:

```bash
kubectl logs --namespace dra-driver-nvidia-gpu -l dra-driver-nvidia-gpu-component=kubelet-plugin -c gpus
```

Refer to [Troubleshooting](../troubleshooting.md) for collecting cluster events and additional diagnostic information.

## Release the vGPU

Deleting the VM instance stops the workload.
The claim generated from the template is released with the consuming VM, and the driver removes the vGPU instance so the GPU capacity can be allocated again.

1. Delete the VM instance and wait for deletion to complete:

   ```bash
   kubectl delete -f vgpu-vm.yaml --wait=true
   ```

2. Check that the generated claim is gone:

   ```bash
   kubectl get resourceclaims --namespace vgpu-demo
   ```

   Expect no claims in this example's namespace.
   If a claim remains, check its owner and consumers and inspect the driver logs before changing the claim.
   Advertised slots can remain in ResourceSlices after cleanup because slots describe capacity rather than existing vGPU instances.

3. Delete the namespace created for the example:

   ```bash
   kubectl delete namespace vgpu-demo
   ```

   Deleting the namespace also removes the claim template.

## Disable vGPU support

Stop the vGPU workloads and release their claims before changing the driver's vGPU configuration.
Set `vgpu.profiles: ""`, `featureGates.VGPUSupport: false`, and `featureGates.DeviceMetadata: false` in `vgpu-values.yaml`, then repeat the Helm upgrade command above.
Keep `DeviceMetadata` enabled if another configured feature requires device metadata.
The restarted plugin stops advertising vGPU slots.
Repeat the slot-listing query and verify that the output contains no rows.
The `vgpu.gpu.nvidia.com` DeviceClass remains while the GPU resource plugin is enabled.
