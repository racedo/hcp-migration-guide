# Migrating Stateful Applications to Hosted Control Planes

A comprehensive guide for migrating stateful applications with persistent storage from standalone OpenShift clusters to Hosted Control Plane (HCP) clusters on bare metal using OADP (OpenShift API for Data Protection).

## Overview

This guide demonstrates a real-world migration of a stateful application (Certificate Discovery Application) with a 211MB SQLite database containing 567,802 certificate records from a standalone OpenShift cluster to an HCP cluster.

**Key Topics Covered:**
- OADP installation and configuration on both source and target clusters
- Backup with File System Backup (Kopia) for persistent volumes
- Handling cluster-scoped RBAC resources (ClusterRole/ClusterRoleBinding)
- Restoring to HCP clusters
- Data validation and verification

## Documentation Compliance

This guide is fully aligned with the **[OpenShift Container Platform 4.21 Backup and Restore](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/backup_and_restore/)** official documentation. All procedures, commands, API versions, and terminology follow Red Hat's official OADP documentation standards.

## Migration Scope

**Source Cluster:**
- Type: Standalone OpenShift cluster (pm-cluster)
- Nodes: 6 nodes
- Application: Certificate Discovery App with SQLite database
- Database Size: 211MB (567,802 certificate records)

**Target Cluster:**
- Type: Hosted Control Plane cluster (bm-hosted-cluster)
- Infrastructure: Bare metal
- Nodes: 2 worker nodes

## Architecture Overview

**Migration Flow:**
```
Source: pm-cluster (Standalone)              Target: bm-hosted-cluster (HCP)
├── OADP installed                           │
├── cert-discovery-app                       ├── OADP installed
│   ├── Deployment                           │
│   ├── Service                              │
│   ├── Route                                │
│   ├── PVC (10Gi, 210MB DB)                 │
│   └── ServiceAccount                       │
│                                            │
├── Cluster-scoped RBAC                      │
│   ├── ClusterRole ────────────────┐        │
│   └── ClusterRoleBinding ─────────┼─┐      │
│                                   │ │      │
├── Export RBAC (manual) ───────────┘ │      │
│                                     │      │
├── Create Backup ──────┐             │      │
│                       │             │      │
│                       ▼             │      │
│                  AWS S3 Bucket      │      │
│                  (eu-north-1)       │      │
│                       │             │      │
│                       │             └──────┼──▶ Apply RBAC first (manual)
│                       │                    │
│                       └────────────────────┼──▶ Restore (namespace-scoped)
│                                            │
└────────────────────────────────────────────┼── cert-discovery-app
                                             │   ├── Deployment
                                             │   ├── Service
                                             │   ├── Route
                                             │   ├── PVC (10Gi, DB restored)
                                             │   └── ServiceAccount
                                             │
                                             └── Cluster-scoped RBAC
                                                 ├── ClusterRole
                                                 └── ClusterRoleBinding
```

**Key Points:**
- **Workload migration**, not control plane backup - OADP installed on both clusters directly
- **HyperShift plugin not needed** for this use case
- **RBAC requires manual handling** - OADP namespace backups don't include cluster-scoped resources
- **Two-step process**: Apply cluster-scoped RBAC first → Then restore namespace-scoped resources

## What You'll Learn

1. **OADP Setup:** Install and configure OADP on both standalone and HCP clusters
2. **Persistent Volume Backup:** Use Kopia for file-level backup of SQLite database
3. **RBAC Handling:** Export and apply cluster-scoped resources not included in namespace backups
4. **HCP Considerations:** Understand when HyperShift plugin is needed vs. not needed
5. **Data Validation:** Verify complete restoration of application data

## Getting Started

Read the complete guide: **[HCP-MIGRATION-GUIDE.md](HCP-MIGRATION-GUIDE.md)**

## Use Case

This guide uses the **[Certificate Discovery Application](https://github.com/racedo/openshift-certificate-analyzer)** as a practical example. The application:
- Discovers certificates across OpenShift clusters
- Stores historical data in SQLite database on persistent volume
- Requires cluster-scoped RBAC permissions
- Runs background refresh every 4 hours
- Perfect example of a stateful application with database persistence

## Migration Results

**Successfully migrated:**
- 862 certificate discovery runs
- 567,802 certificate records
- 211MB SQLite database
- Full application functionality on HCP cluster

**Migration metrics:**
- Backup Duration: ~10-15 minutes
- Restore Duration: ~15-20 minutes
- Data Integrity: 100% preservation
- Downtime: 0 minutes (parallel deployment)

## Infrastructure Benefits

Migrating to HCP provides significant infrastructure savings:

**Control Plane Pod Density:**
- Each hosted control plane: ~75 pods
- Max pods per node: 500
- Hosted control planes per management node: ~6

| Architecture | Nodes Required | Calculation | Savings |
|--------------|----------------|-------------|---------|
| 10 Standalone clusters (6 nodes each) | 60 nodes | 10 × 6 = 60 | Baseline |
| HCP (3 mgmt + 10 HCP × 2 workers) | 23 nodes | 3 + (10 × 2) = 23 | **62% reduction** |

**HCP Breakdown:**
- Management cluster: 3 nodes (hosts 10 control planes at ~6 per node, HA configured)
- Worker nodes: 20 nodes (10 HCP clusters × 2 workers each)

## Prerequisites

- Cluster-admin access to both source and target clusters
- AWS S3 bucket for backup storage
- Basic understanding of OpenShift and OADP

## Key Takeaway

**Workload migration to HCP clusters uses standard OADP procedures.** The HyperShift plugin is only needed for control plane backup/restore, not for migrating applications.

## Additional Resources

- **[OpenShift 4.21 Backup and Restore](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/backup_and_restore/)** - Official OADP documentation
- **[OADP Operator](https://docs.openshift.com/container-platform/4.21/backup_and_restore/application_backup_and_restore/oadp-intro.html)** - Installing and configuring OADP
- **[Hosted Control Planes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/hosted_control_planes/index)** - Official HCP documentation
- **[HyperShift Documentation](https://hypershift-docs.netlify.app/)** - Upstream HyperShift project
- **[Velero Documentation](https://velero.io/docs/)** - Upstream Velero project
- **[Certificate Discovery App](https://github.com/racedo/openshift-certificate-analyzer)** - Example application repository

## License

Apache-2.0

## Contributing

This guide is based on real-world migration experience. If you have suggestions or improvements, please open an issue or submit a pull request.

---

**Author:** Red Hat Partner Engineering
**Date:** March 2026
**Environment:** pm-cluster → bm-hosted-cluster (Bare Metal HCP)
**Documentation Version:** Aligned with OpenShift 4.21
