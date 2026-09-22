# Kubernetes Pod & Host Node Resource (USE Dahboard)

Visualize utilization, saturation, and errors for Kubernetes pod workloads and their underlying nodes.

A Grafana dashboard that correlates pod resource usage with host-node pressure and OpenEBS LocalPV LVM storage activity. It includes conditional warning overlays and automatic PVC-to-volume-group resolution.

## Required Exporters

Grafana requires a Prometheus datasource scraping:

| Source | Required Metrics |
| --- | --- |
| **kube-state-metrics** | Pod placement, container resource limits, PVC/PV metadata, and CSI volume handles. |
| **node_exporter** | CPU, memory, network drops/errors, and diskstats counters, including LVM `dm-*` devices. |
| **Kubelet / cAdvisor** | Container CPU, memory, network, and filesystem I/O counters; kubelet volume usage and capacity metrics. |
| **OpenEBS LocalPV LVM exporter** | `lvm_lv_total_size_bytes` with LV name, VG, and device labels for the LVM section. |

Container filesystem I/O and volume-capacity metrics depend on runtime and CSI support. Missing metrics appear as gaps, not zero activity. Automatic LVM discovery requires exporter node labels, exporter pod placement metadata, or a recognizable node-name/host-IP endpoint.

## The USE Method

The [USE method](https://www.brendangregg.com/usemethod.html) checks each resource for three kinds of performance signals:

- **Utilization:** How busy the resource is, or how much of its capacity is used.
- **Saturation:** Work queued or waiting because the resource cannot serve it immediately.
- **Errors:** Failed operations or resource-level errors, with packet drops as an additional network warning signal.

This dashboard applies that approach by comparing pod activity with host CPU, memory, storage, and network conditions. A pod can remain below its own limits while competing for an overloaded host resource.

CPU busy, device busy, IOPS, and capacity used are not interchangeable measures. The dashboard provides utilization, iowait/steal, memory-pressure warnings, and network error/drop signals; it does not directly measure every resource's queueing or saturation. Use these signals to guide investigation, not as proof of a bottleneck.

## Variables

| Variable | Purpose |
| --- | --- |
| `datasource` | Prometheus datasource. |
| `namespace`, `pod` | Select the workload. |
| `node` | Resolves the pod's current Kubernetes node. |
| `node_instance`, `node_job` | Select the resolved node_exporter target and scrape job. |
| `pvc` | Select a mounted PVC for capacity reporting and automatic VG resolution. |
| `pod_device` | Manually select a cAdvisor device for the pod IOPS comparison. |
| `node_disk` | Select node block devices; defaults to All. |
| `vg` | Automatically resolves the selected PVC's OpenEBS volume group. |

Hidden variables support host-IP discovery and the mapping:

`PVC -> PV -> CSI volume handle -> Logical Volume -> Volume Group`

The manual pod-device selector remains independent of PVC selection.

## Pod Section

- **CPU:** Pod CPU usage and configured container limits, with host CPU busy percentage.
- **Memory:** Pod working set and configured limits, with a red highlight when host memory unavailable exceeds 95%.
- **Network:** Pod RX/TX bandwidth, with a red highlight when host interfaces report drops or errors.
- **Storage:** Total pod read/write IOPS, selected-device IOPS, and selected-PVC capacity used percentage.

Configured limit sums are not complete pod ceilings if some containers are unlimited. Host memory unavailable uses `1 - MemAvailable/MemTotal`, not pod or node working set.

## Node Section

- **Compute and Memory:** Host CPU busy, RAM unavailable, CPU iowait, and steal percentages.
- **Disk I/O:** Read/write IOPS and device busy percentage for each selected device.
- **Network Errors:** Per-interface receive/transmit drops and errors.

These panels show host conditions shared by the workload and other pods. Host warnings do not prove that the selected pod was affected.

## OpenEBS Section

Selecting a PVC automatically resolves its VG and displays:

- **VG IOPS:** Aggregate read/write IOPS across reported logical volumes, compared with the selected PVC's logical-volume IOPS.
- **VG Throughput:** Aggregate read/write bytes per second.

OpenEBS metrics identify VG membership; node_exporter provides measured device I/O counters. Thin-pool devices are excluded to reduce double counting. These panels show **logical-volume-layer I/O**, not physical-drive totals. Missing device counters can make VG totals incomplete.

## Usage

1. Import `pod-node-saturation-dashboard.json` into Grafana.
2. Select the datasource, namespace, and pod.
3. Confirm the node_exporter target and job are populated.
4. Select a PVC for the OpenEBS section; its volume group resolves automatically.

Use a single-cluster datasource and configure its scrape interval correctly for `$__rate_interval`. Historical node correlation follows the selected pod's current placement.
