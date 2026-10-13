# WEKA LWH Exporter v2 — Grafana Dashboard

Grafana dashboard for the **WEKA Local WEKA Home (LWH) Prometheus Exporter v2**, scraped from:

```
/api/v3/clusters/<cluster_id>/stats/prometheus/v2            # cluster
/api/v3/clusters/<cluster_id>/stats/prometheus/v2/clients
/api/v3/clusters/<cluster_id>/stats/prometheus/v2/drives
/api/v3/clusters/<cluster_id>/stats/prometheus/v2/quota_domains
```

See [Prometheus scrape example](#prometheus-scrape-example) for setup.

| | |
|---|---|
| Dashboard title | `WEKA LWH Exporter v2` |
| UID | `weka-exporter-v2-2` |
| Grafana | 11.6.x (schemaVersion 41) |
| Datasource | Prometheus |
| Tags | `wekahome`, `weka`, `exporter-v2` |
| Default range / refresh | last 6h / 1m |

## Repository layout

```
.
├── README.md
└── dashboards/
    └── weka-lwh-exporter-v2.json   # Grafana dashboard JSON model
```

## Template variables

| Variable | Source | Used by |
|---|---|---|
| `$cluster` | `label_values(weka_cluster_status, cluster_name)` | All panels |
| `$fs` | `label_values(weka_filesystem_capacity_bytes, fs_name)` | Filesystems |
| `$server` | `label_values(weka_cpu_utilization_percent, server_name)` | Compute & Network, Drives |
| `$client` | `label_values(weka_posix_client_metadata_ops_per_second, server_name)` | WEKA Clients |
| `$tenant` | `label_values(weka_tenant_read_ops_per_second, tenant_name)` | Tenants |
| `$qd` | `label_values(weka_qd_write_ops_per_second, qd_name)` | Quota Domains |

All variables are multi-select with an `All` (`.*`) option.

## Dashboard sections

| Row | Default | Highlights |
|---|---|---|
| **Cluster Info** | expanded | Cluster / IO / license status, link layer, alerts, capacity, inactive drives, role & server counts, throughput, IOPS, protection level, read/write latency percentiles, metadata ops & latency, QoS delay, process health |
| **WEKA Clients** | expanded | Per-client POSIX throughput, IOPS, latency (incl. QoS-excluded), metadata ops, latency percentiles, IO-size heatmaps, client CPU |
| **Cluster Compute & Network** | collapsed | CPU by role/server, busiest processes, pending IOs, NIC RX/TX bytes and errors |
| **Filesystems** | collapsed | FS / snapshot / OBS bucket counts, SSD vs. OBS capacity breakdown, per-FS throughput, IOPS, latency, data reduction ratio |
| **Drives** | collapsed | Inactive drives table, temperature, SMART media errors & critical warnings, per-drive throughput/IOPS/latency |
| **Latency Distribution & Congestion** | collapsed | Read/write duration heatmaps, choking level, TX queue full (cluster + top servers) |
| **Quota Domains** | collapsed | Per-QD throughput, IOPS, latency |
| **Tenants** | collapsed | Per-tenant throughput, IOPS, latency (with/without QoS), QoS delay |
| **S3 Protocol** | collapsed | Buckets, SLB connections/requests/resets, API ops & failures, TTFB/TTLB, per-FS S3 traffic, lifecycle ops |

## Installation

### Grafana UI

1. **Dashboards → New → Import**.
2. Upload `dashboards/weka-lwh-exporter-v2.json`.
3. Pick your Prometheus datasource and import.

> The JSON references the datasource UID `ffyvyko3uo6bkd`. If your Prometheus datasource has a different UID, either select it at import time or replace the UID in the file:
>
> ```bash
> sed -i 's/ffyvyko3uo6bkd/<your-datasource-uid>/g' dashboards/weka-lwh-exporter-v2.json
> ```

## Prometheus scrape example

> Reference: [Export cluster metrics from LWH to Prometheus](https://docs.weka.io/monitor-the-weka-cluster/the-wekaio-support-cloud/export-cluster-metrics-to-prometheus) (WEKA docs)

Exporter V2 is the supported version starting with LWH 5.0 (V1 is deprecated). To enable it, open the cluster in LWH, go to the **Prometheus Exporter** tab, select **Generate Token**, and copy the endpoint URLs under **Prometheus Exporter URL v2**. The token is shown only once. Each cluster has one active token, and generating a new one invalidates the old one.

V2 exposes four endpoints. This dashboard uses metrics from all of them, so add one scrape job per endpoint:

| Endpoint | Content |
|---|---|
| `<base-url>` | Cluster health, capacity, S3 |
| `<base-url>/clients` | Per-client and per-filesystem I/O |
| `<base-url>/drives` | Per-drive health, I/O, network |
| `<base-url>/quota_domains` | Per-quota-domain I/O |

The full metric list is available in LWH via **Download YAML** under **Metric Reference**.

```yaml
# prometheus.yaml
scrape_configs:

  - job_name: weka_cluster
    metrics_path: <base-url>
    scheme: https
    scrape_interval: 1m
    authorization:
      credentials: <your-token>
    static_configs:
      - targets: ['<lwh-host>']

  - job_name: weka_clients
    metrics_path: <base-url>/clients
    scheme: https
    scrape_interval: 1m
    authorization:
      credentials: <your-token>
    static_configs:
      - targets: ['<lwh-host>']

  - job_name: weka_drives
    metrics_path: <base-url>/drives
    scheme: https
    scrape_interval: 1m
    authorization:
      credentials: <your-token>
    static_configs:
      - targets: ['<lwh-host>']

  - job_name: weka_quota_domains
    metrics_path: <base-url>/quota_domains
    scheme: https
    scrape_interval: 1m
    authorization:
      credentials: <your-token>
    static_configs:
      - targets: ['<lwh-host>']
```

| Placeholder | Value |
|---|---|
| `<lwh-host>` | LWH hostname from the cluster settings URL |
| `<base-url>` | Path portion of the endpoint URL from the token dialog (e.g. `/api/v3/clusters/<cluster_id>/stats/prometheus/v2`) |
| `<your-token>` | Authentication token from the token dialog |

Panels use a `1m` minimum interval, so a 1-minute scrape interval is recommended. To limit which metrics are exported, see [Configure selective Prometheus metric collection](https://docs.weka.io/monitor-the-weka-cluster/the-wekaio-support-cloud/configure-selective-prometheus-metric-collection).
