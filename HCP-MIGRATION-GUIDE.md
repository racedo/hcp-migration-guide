# Migrating Stateful Applications to Hosted Control Planes on Bare Metal

## Overview

This guide demonstrates migrating the **Certificate Discovery Application** with persistent storage from a standalone OpenShift cluster (pm-cluster) to a Hosted Control Plane cluster (bm-hosted-cluster) on bare metal infrastructure.

**Documentation Compliance:**

This guide is fully aligned with the **[OpenShift Container Platform 4.21 Backup and Restore](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/backup_and_restore/)** official documentation. All procedures, commands, API versions, and terminology follow Red Hat's official OADP documentation standards.

**What You'll Learn:**
- Install and configure OADP on both source and target clusters
- Backup stateful applications with persistent volume data using File System Backup (Kopia)
- Restore workloads to HCP clusters using standard OADP procedures
- Verify data integrity after migration
- Understand when HyperShift plugin is needed (control plane backup) vs not needed (workload migration)

---

## Prerequisites

### 1. Cluster Access

You need cluster-admin access to both clusters:
- **Source:** pm-cluster (standalone OpenShift, 6 nodes)
- **Target:** bm-hosted-cluster (HCP cluster, 2 worker nodes)

Obtain kubeconfig files for both clusters from your cluster administrators.

```bash
# Example: Log in to source cluster and save kubeconfig
oc login https://api.pm-cluster.example.com:6443 --username=admin
oc config view --flatten > ~/kubeconfig-pm-cluster

# Example: Log in to hosted cluster and save kubeconfig
oc login https://api.bm-hosted-cluster.example.com:6443 --username=admin
oc config view --flatten > ~/kubeconfig-bm-hosted-cluster

# Verify access to both clusters
export KUBECONFIG=~/kubeconfig-pm-cluster
oc whoami
oc get nodes

export KUBECONFIG=~/kubeconfig-bm-hosted-cluster
oc whoami
oc get nodes
```

**Note:** Adjust the kubeconfig file paths (`~/kubeconfig-pm-cluster` and `~/kubeconfig-bm-hosted-cluster`) throughout this guide to match your actual file locations.

### 2. Export Cluster-Scoped RBAC Resources

**Important:** OADP namespace backups only include namespace-scoped resources. Cluster-scoped resources (ClusterRoles, ClusterRoleBindings) are NOT included in the backup and must be exported separately.

Export cluster-scoped RBAC resources for your application:

```bash
export KUBECONFIG=~/kubeconfig-pm-cluster

# Find ClusterRoles for your application
oc get clusterrole | grep cert-discovery

# Find ClusterRoleBindings for your application
oc get clusterrolebinding | grep cert-discovery
```

Expected output:
```
cert-discovery-role                                                                             2025-11-06T11:53:23Z
cert-discovery-binding                                                                          ClusterRole/cert-discovery-role
```

Export these resources to YAML files:

```bash
# Export ClusterRole
oc get clusterrole cert-discovery-role -o yaml > /tmp/clusterrole-cert-discovery.yaml

# Export ClusterRoleBinding
oc get clusterrolebinding cert-discovery-binding -o yaml > /tmp/clusterrolebinding-cert-discovery.yaml

# Verify files were created
ls -lh /tmp/cluster*.yaml
```

**Important:** Keep these files - you'll need to apply them on the target cluster in Step 7 before the restore.

**Why this is needed:**
- ClusterRole/ClusterRoleBinding are cluster-scoped (not in any namespace)
- OADP backup with `includedNamespaces: [cert-discovery-app]` only backs up namespace-scoped resources
- Without these permissions, the application will fail with "403 Forbidden" errors

---

**Alternative: Automated RBAC Backup (Optional)**

Instead of manually exporting RBAC resources, you can include them in the OADP backup by labeling them first and using `includedClusterScopedResources` in the Backup spec.

**Step 1: Label the cluster-scoped RBAC resources**

```bash
export KUBECONFIG=~/kubeconfig-pm-cluster

# Add app label to ClusterRole
oc label clusterrole cert-discovery-role app=cert-discovery

# Add app label to ClusterRoleBinding
oc label clusterrolebinding cert-discovery-binding app=cert-discovery

# Verify labels were added
oc get clusterrole cert-discovery-role --show-labels
oc get clusterrolebinding cert-discovery-binding --show-labels
```

**Step 2: Modify the Backup spec in Step 4**

When creating the backup, use this modified spec instead:

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: cert-discovery-app-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - cert-discovery-app
  includedClusterScopedResources:
    - clusterroles
    - clusterrolebindings
  labelSelector:
    matchLabels:
      app: cert-discovery
  storageLocation: default
  ttl: 720h0m0s
  defaultVolumesToFsBackup: true
```

**Benefits:**
- RBAC resources are automatically included in the backup
- No manual export/apply steps needed
- RBAC resources are restored automatically with the Restore CR

**Note:** If using this approach, you can skip Step 7 (Apply Cluster-Scoped RBAC) as it will be handled automatically during restore.

---

### 3. S3 Storage
- AWS S3 bucket: `cert-discovery-management-app`
- Region: `eu-north-1`
- IAM credentials with read/write permissions

---

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

---

## Step 1: Install OADP on Management Cluster

### 1.1 Install OADP Operator

You can install the OADP Operator using either the OpenShift web console or CLI.

**Using CLI:**

```bash
export KUBECONFIG=~/kubeconfig-pm-cluster

# Create OADP namespace
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
EOF

# Create OperatorGroup
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: oadp-operator-group
  namespace: openshift-adp
spec:
  targetNamespaces:
  - openshift-adp
EOF

# Create Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: oadp-operator
  namespace: openshift-adp
spec:
  channel: stable
  name: oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**Using Web Console:**

1. In the OpenShift Container Platform web console, click **Operators → OperatorHub**
2. Search for **OADP Operator** in the search field
3. Select the **OADP Operator** and click **Install**
4. On the Install Operator page:
   - Update Channel: **stable**
   - Installation Mode: **A specific namespace on the cluster**
   - Installed Namespace: **openshift-adp** (create if it doesn't exist)
   - Approval Strategy: **Automatic**
5. Click **Install**

### 1.2 Verify Operator Installation

```bash
# Wait for operator to be ready
oc wait --for=condition=ready pod -l control-plane=controller-manager \
  -n openshift-adp --timeout=300s

# Verify OADP operator version
oc get csv -n openshift-adp | grep oadp-operator
```

Expected output:
```
oadp-operator.v1.5.5    OADP Operator    1.5.5    oadp-operator.v1.5.4    Succeeded
```

---

## Step 2: Configure AWS S3 Storage

### 2.1 Configure AWS Credentials

First, configure your AWS credentials:

```bash
# Configure AWS credentials
aws configure

# You'll be prompted for:
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: eu-north-1
# Default output format: json
```

Alternatively, if you already have credentials, you can verify access:

```bash
# Verify AWS authentication
aws sts get-caller-identity
```

### 2.2 Create S3 Bucket

```bash
# Set environment variables
export BUCKET=cert-discovery-management-app
export REGION=eu-north-1

# Create S3 bucket
aws s3api create-bucket \
  --bucket $BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION
```

**Note:** For `eu-north-1` region, you must use `--create-bucket-configuration` with `LocationConstraint=$REGION`.

### 2.3 Create Credentials File

Create a credentials file for Velero:

```bash
cat > /tmp/credentials-velero <<EOF
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
EOF
```

Replace `<AWS_ACCESS_KEY_ID>` and `<AWS_SECRET_ACCESS_KEY>` with your actual AWS credentials.

### 2.4 Create Secret in OpenShift

```bash
oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-file cloud=/tmp/credentials-velero
```

**Important:** The Secret must be named `cloud-credentials` and the key must be `cloud`.

```bash
# Verify secret was created
oc get secret cloud-credentials -n openshift-adp

# Clean up credentials file
rm /tmp/credentials-velero
```

---

## Step 3: Install DataProtectionApplication

### 3.1 Create DataProtectionApplication CR

The DataProtectionApplication (DPA) configures the connection between OADP and your backup storage.

**Prerequisites:**
- You must have installed the OADP Operator
- You must have created the `cloud-credentials` Secret

```bash
cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero-management
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
      resourceTimeout: 10m
    nodeAgent:
      enable: true
      uploaderType: kopia
  backupLocations:
    - name: default
      velero:
        provider: aws
        default: true
        objectStorage:
          bucket: cert-discovery-management-app
          prefix: management-export
        config:
          region: eu-north-1
          profile: "default"
        credential:
          name: cloud-credentials
          key: cloud
EOF
```

**Field Descriptions:**

| Field | Value | Description |
|-------|-------|-------------|
| `defaultPlugins` | `openshift`, `aws` | Required plugins for OpenShift and AWS S3 |
| `nodeAgent.enable` | `true` | Enables DaemonSet for PV backup |
| `uploaderType` | `kopia` | Uses Kopia for file-level PV backup (default in OADP 1.3+) |
| `bucket` | `cert-discovery-management-app` | S3 bucket name |
| `prefix` | `management-export` | S3 path prefix for backups |
| `region` | `eu-north-1` | AWS region |

### 3.2 Verify DataProtectionApplication Status

```bash
# Wait for DPA to reconcile
oc get dpa -n openshift-adp
```

Expected output:
```
NAME                RECONCILED   AGE
velero-management   True         45s
```

The `RECONCILED` column should show `True`.

```bash
# Check all OADP pods are running
oc get pods -n openshift-adp
```

Expected output:
```
NAME                                                READY   STATUS    RESTARTS   AGE
node-agent-xxxxx                                    1/1     Running   0          60s
node-agent-yyyyy                                    1/1     Running   0          60s
node-agent-zzzzz                                    1/1     Running   0          60s
openshift-adp-controller-manager-xxxxx              1/1     Running   0          3m
velero-xxxxx                                        1/1     Running   0          60s
```

**Note:** You should see one `node-agent` pod per worker node in the cluster.

### 3.3 Verify Backup Storage Location

```bash
# Check BackupStorageLocation status
oc get backupstoragelocation -n openshift-adp
```

Expected output:
```
NAME      PHASE       LAST VALIDATED   AGE   DEFAULT
default   Available   35s              87m   true
```

The `PHASE` must be `Available` before you can create backups.

---

---

## Step 4: Create Backup

### 4.1 Create Backup CR

Create a `Backup` custom resource to back up the application namespace including PersistentVolumes.

**Prerequisites:**
- DataProtectionApplication must be reconciled (verified in Step 3.2)
- BackupStorageLocation must be `Available` (verified in Step 3.3)

Optionally verify before creating backup:
```bash
# Quick verification
oc get dpa -n openshift-adp
oc get backupstoragelocation -n openshift-adp
```

Create the backup:
```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: cert-discovery-app-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - cert-discovery-app
  storageLocation: default
  ttl: 720h0m0s
  defaultVolumesToFsBackup: true
EOF
```

**Field Descriptions:**

| Field | Value | Description |
|-------|-------|-------------|
| `includedNamespaces` | `cert-discovery-app` | Namespaces to include in backup |
| `storageLocation` | `default` | BackupStorageLocation name |
| `ttl` | `720h0m0s` | Time to live (30 days) |
| `defaultVolumesToFsBackup` | `true` | **Critical:** Enables file-level PV backup using Kopia |

**Important:** Setting `defaultVolumesToFsBackup: true` is required to back up persistent volumes using File System Backup (FSB). Without this, only Kubernetes resources are backed up and the SQLite database will be lost.

**Note:** If you're using the **Alternative: Automated RBAC Backup** approach from Prerequisites, use the modified Backup spec shown in that section instead. It includes `includedClusterScopedResources` and `labelSelector` to automatically backup ClusterRole and ClusterRoleBinding.

### 4.2 Monitor Backup Progress

```bash
# Check backup phase (should progress: New -> InProgress -> Completed)
oc get backup cert-discovery-app-backup -n openshift-adp \
  -o jsonpath='{.status.phase}'
```

The backup will progress through these phases: `New` → `InProgress` → `Completed`

**Tip:** If the backup fails or gets stuck, use `oc describe backup cert-discovery-app-backup -n openshift-adp` to see detailed error messages.

### 4.3 Wait for Backup Completion

```bash
# Wait for backup to complete (up to 15 minutes)
oc wait --for=jsonpath='{.status.phase}'=Completed \
  backup/cert-discovery-app-backup -n openshift-adp --timeout=15m
```

Expected output:
```
backup.velero.io/cert-discovery-app-backup condition met
```

### 4.4 Verify Backup in S3

```bash
# Verify backup metadata exists
aws s3 ls s3://cert-discovery-management-app/management-export/backups/cert-discovery-app-backup/ --region eu-north-1
```

You should see backup artifacts including:
- `cert-discovery-app-backup.tar.gz` - Kubernetes resources
- `cert-discovery-app-backup-logs.gz` - Backup logs
- `velero-backup.json` - Backup metadata

```bash
# Verify PV data in Kopia repository
aws s3 ls s3://cert-discovery-management-app/management-export/kopia/ --region eu-north-1
```

Expected output:
```
                           PRE cert-discovery-app/
```

This confirms Kopia created a repository for the namespace - this is where your PV data (SQLite database) is stored.

Optionally verify Kopia data exists:
```bash
# Check for Kopia pack files (these contain your database)
aws s3 ls s3://cert-discovery-management-app/management-export/kopia/cert-discovery-app/ --region eu-north-1 | grep -E '^2026.*p[0-9a-f]'
```

You should see multiple files starting with `p` (pack files, ~20MB each) - these are your database backup chunks.

---

## Step 5: Install OADP on Hosted Cluster

### 5.1 Verify HCP Cluster Access

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Verify cluster access
oc whoami
oc get nodes
```

Expected output:
```
NAME     STATUS   ROLES    AGE   VERSION
node47   Ready    worker   5d    v1.32.9+...
node48   Ready    worker   5d    v1.32.9+...
```

### 5.2 Install OADP Operator

Install OADP directly on the hosted cluster where you want to restore the application.

**Using CLI:**

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Create OADP namespace
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
EOF

# Create OperatorGroup
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: oadp-operator-group
  namespace: openshift-adp
spec:
  targetNamespaces:
  - openshift-adp
EOF

# Create Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: oadp-operator
  namespace: openshift-adp
spec:
  channel: stable
  name: oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**Using Web Console:**

1. Log in to the hosted cluster web console
2. Click **Operators → OperatorHub**
3. Search for **OADP Operator** in the search field
4. Select the **OADP Operator** and click **Install**
5. On the Install Operator page:
   - Update Channel: **stable**
   - Installation Mode: **A specific namespace on the cluster**
   - Installed Namespace: **openshift-adp** (create if it doesn't exist)
   - Approval Strategy: **Automatic**
6. Click **Install**

### 5.3 Verify Operator Installation

```bash
# Wait for operator to be ready
oc wait --for=condition=ready pod -l control-plane=controller-manager \
  -n openshift-adp --timeout=300s

# Verify OADP operator version
oc get csv -n openshift-adp | grep oadp-operator
```

Expected output:
```
oadp-operator.v1.5.5    OADP Operator    1.5.5    oadp-operator.v1.5.4    Succeeded
```

---

## Step 6: Configure OADP on Hosted Cluster

### 6.1 Create S3 Credentials Secret

Create the same S3 credentials on the hosted cluster.

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Create credentials file
cat > /tmp/credentials-velero <<EOF
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
EOF

# Create secret
oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-file cloud=/tmp/credentials-velero

# Clean up credentials file
rm /tmp/credentials-velero
```

### 6.2 Create DataProtectionApplication

Configure OADP to use the same S3 bucket as the source cluster.

**Note:** For workload migration, we don't need the HyperShift plugin. That's only required for backing up the hosted cluster's control plane itself.

```bash
cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero-hcp
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
      resourceTimeout: 10m
    nodeAgent:
      enable: true
      uploaderType: kopia
  backupLocations:
    - name: default
      velero:
        provider: aws
        default: true
        objectStorage:
          bucket: cert-discovery-management-app
          prefix: management-export
        config:
          region: eu-north-1
          profile: "default"
        credential:
          name: cloud-credentials
          key: cloud
EOF
```

**Key Configuration:**
- **Same S3 bucket:** Uses the same bucket as the source cluster backup
- **Same prefix:** `management-export` matches the source cluster configuration
- **No HyperShift plugin:** Not needed for workload restore

### 6.3 Verify OADP Installation

```bash
# Wait for DPA to reconcile
oc wait --for=condition=Reconciled dpa/velero-hcp -n openshift-adp --timeout=300s

# Check all OADP pods are running
oc get pods -n openshift-adp
```

Expected output:
```
NAME                                                READY   STATUS    RESTARTS   AGE
node-agent-xxxxx                                    1/1     Running   0          60s
node-agent-yyyyy                                    1/1     Running   0          60s
openshift-adp-controller-manager-xxxxx              1/1     Running   0          3m
velero-xxxxx                                        1/1     Running   0          60s
```

**Note:** You should see one `node-agent` pod per worker node in the hosted cluster.

### 6.4 Verify Backup Discovery

OADP should automatically discover existing backups from the S3 bucket.

```bash
# List available backups
oc get backups -n openshift-adp
```

Expected output:
```
NAME                          AGE
cert-discovery-app-backup     15m
```

```bash
# Verify backup storage location is available
oc get backupstoragelocation -n openshift-adp
```

Expected output:
```
NAME      PHASE       LAST VALIDATED   AGE   DEFAULT
default   Available   10s              75s   true
```

---

## Step 7: Apply Cluster-Scoped RBAC

**Note:** If you used the **Alternative: Automated RBAC Backup** approach from Prerequisites, you can **skip this step**. The ClusterRole and ClusterRoleBinding will be restored automatically in Step 8.

The ClusterRole and ClusterRoleBinding are cluster-scoped resources that were not included in the namespace backup. Apply these BEFORE restoring so the application pods have permissions when they start.

Apply the YAML files you exported in the Prerequisites section.

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Clean up the exported YAML files (remove cluster-specific metadata)
# Remove these fields: resourceVersion, uid, creationTimestamp, managedFields
sed -i '' '/resourceVersion:/d; /uid:/d; /creationTimestamp:/d; /managedFields:/,/^[^ ]/d' /tmp/clusterrole-cert-discovery.yaml
sed -i '' '/resourceVersion:/d; /uid:/d; /creationTimestamp:/d; /managedFields:/,/^[^ ]/d' /tmp/clusterrolebinding-cert-discovery.yaml

# Apply the ClusterRole
oc apply -f /tmp/clusterrole-cert-discovery.yaml

# Apply the ClusterRoleBinding
oc apply -f /tmp/clusterrolebinding-cert-discovery.yaml
```

**Note:** The `sed` commands remove cluster-specific metadata that would prevent the resources from being created on the new cluster.

Verify the RBAC was created:

```bash
# Verify ClusterRole exists
oc get clusterrole cert-discovery-role

# Verify ClusterRoleBinding exists
oc get clusterrolebinding cert-discovery-binding

# Verify the ServiceAccount is bound correctly
oc describe clusterrolebinding cert-discovery-binding
```

Expected output from describe:
```
Name:         cert-discovery-binding
Role:
  Kind:  ClusterRole
  Name:  cert-discovery-role
Subjects:
  Kind            Name               Namespace
  ----            ----               ---------
  ServiceAccount  cert-discovery-sa  cert-discovery-app
```

---

## Step 8: Restore Application to Hosted Cluster

Now that RBAC is in place, restore the application. The pods will start with the correct permissions already configured.

### 8.1 Create Restore CR

Create a `Restore` custom resource to restore the application.

**Prerequisites:**
- Backup must be in `Completed` state
- OADP is installed and configured on the hosted cluster
- Backup is visible in `oc get backups -n openshift-adp`
- Cluster-scoped RBAC has been applied (Step 7) OR included in backup via alternative approach

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: cert-discovery-app-restore
  namespace: openshift-adp
spec:
  backupName: cert-discovery-app-backup
  includedNamespaces:
    - cert-discovery-app
  restorePVs: true
  existingResourcePolicy: update
EOF
```

**Field Descriptions:**

| Field | Value | Description |
|-------|-------|-------------|
| `backupName` | `cert-discovery-app-backup` | Name of the backup to restore from |
| `includedNamespaces` | `cert-discovery-app` | Namespaces to restore |
| `restorePVs` | `true` | **Critical:** Restores persistent volumes |
| `existingResourcePolicy` | `update` | Updates existing resources if they exist |

### 8.2 Wait for Restore Completion

```bash
# Wait for restore to complete (up to 10 minutes)
oc wait --for=jsonpath='{.status.phase}'=Completed \
  restore/cert-discovery-app-restore -n openshift-adp --timeout=10m
```

Expected output:
```
restore.velero.io/cert-discovery-app-restore condition met
```

**Tip:** If the restore fails, use `oc describe restore cert-discovery-app-restore -n openshift-adp` to see error details.

---

## Step 9: Verify Application on HCP Cluster

The application pod should start cleanly with permissions already in place.

### 9.1 Check Pod Status

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Wait for pod to become ready
oc get pods -n cert-discovery-app -w
```

Press `Ctrl+C` once the pod shows `1/1 Ready`.

### 9.2 Check All Resources

```bash
# Check namespace exists
oc get namespace cert-discovery-app

# Check all resources
oc get all -n cert-discovery-app
```

Expected output:
```
NAME                                      READY   STATUS    RESTARTS   AGE
pod/cert-discovery-app-xxxxx-xxxxx        1/1     Running   0          3m

NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/cert-discovery-service   ClusterIP   172.30.xxx.xxx   <none>        80/TCP    3m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-discovery-app   1/1     1            1           3m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-discovery-app-xxxxx        1         1         1       3m
```

### 9.3 Verify PVC is Bound

```bash
oc get pvc -n cert-discovery-app
```

Expected output:
```
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cert-discovery-data   Bound    pvc-yyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy      10Gi       RWO            lvms-vg1       3m
```

### 9.4 Check Pod Logs

```bash
oc logs -n cert-discovery-app -l app=cert-discovery --tail=50 | grep -i database
```

Expected output:
```
2026-03-30 12:15:23,940 - __main__ - INFO - Database initialized successfully at /data/certificates.db
2026-03-30 12:15:23,944 - __main__ - INFO - Database initialized and available for historical tracking
```

---

## Step 10: Validate Restored Data

Verify the database was restored with all historical data.

```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Check database file was restored
oc exec -n cert-discovery-app deployment/cert-discovery-app -- ls -lh /data/certificates.db

# Verify discovery runs were restored
oc exec -n cert-discovery-app deployment/cert-discovery-app -- \
  sqlite3 /data/certificates.db "SELECT COUNT(*) FROM certificate_discoveries;"

# Verify certificate records were restored
oc exec -n cert-discovery-app deployment/cert-discovery-app -- \
  sqlite3 /data/certificates.db "SELECT COUNT(*) FROM certificates;"

# View most recent discoveries
oc exec -n cert-discovery-app deployment/cert-discovery-app -- \
  sqlite3 /data/certificates.db "SELECT id, timestamp, total_certificates FROM certificate_discoveries ORDER BY id DESC LIMIT 5;"
```

Expected output:
```
-rw-rw-r--. 1 1000960000 1000960000 211M Mar 31 14:47 /data/certificates.db

862

567802

862|2026-03-31 14:47:54|173
861|2026-03-31 14:44:18|173
860|2026-03-31 14:41:20|173
859|2026-03-31 14:33:14|173
858|2026-03-31 14:30:14|173
```

If you see historical discovery runs and certificate records, the migration was successful!

You can also access the web UI:
```bash
ROUTE=$(oc get route cert-discovery-route -n cert-discovery-app -o jsonpath='{.spec.host}')
echo "Application URL: http://$ROUTE"
```

---

## Cleanup (Optional)

### Remove Application from Source Cluster

**Only after confirming target cluster is working correctly!**

```bash
export KUBECONFIG=~/kubeconfig-pm-cluster

# Delete the application namespace
oc delete namespace cert-discovery-app

# Optionally delete the backup (keeps it in S3)
# oc delete backup cert-discovery-app-backup -n openshift-adp
```

### Keep OADP for Future Migrations

The OADP operator and DPA resources can be kept for migrating other applications.

---

## Troubleshooting

### Issue: Pod CrashLoopBackOff with Permission Errors

**Symptom:**
```
Health check failed: (403) Forbidden
User "system:serviceaccount:cert-discovery-app:cert-discovery-sa" cannot list resource "namespaces"
```

**Cause:** ClusterRole/ClusterRoleBinding were not backed up (cluster-scoped resources are excluded from namespace backups).

**Solution:**
1. If you didn't export the RBAC in Prerequisites, switch to the source cluster and export now:
   ```bash
   export KUBECONFIG=~/kubeconfig-pm-cluster
   oc get clusterrole cert-discovery-role -o yaml > /tmp/clusterrole-cert-discovery.yaml
   oc get clusterrolebinding cert-discovery-binding -o yaml > /tmp/clusterrolebinding-cert-discovery.yaml
   ```
2. Apply the RBAC on the target cluster as shown in Step 7

### Issue: Backup Storage Location Shows "Unavailable"

**Check S3 connectivity:**
```bash
oc logs -n openshift-adp deployment/velero | grep -i error

# Test S3 access
aws s3 ls s3://cert-discovery-management-app --region eu-north-1
```

### Issue: Restore Stuck in "InProgress"

**Check restore details:**
```bash
oc describe restore cert-discovery-app-restore -n openshift-adp

# Check for errors
oc get restore cert-discovery-app-restore -n openshift-adp \
  -o jsonpath='{.status.errors}'

# Check Velero logs
oc logs -n openshift-adp deployment/velero -f
```

### Issue: PVC Not Binding on Target

**Check storage class:**
```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

oc get pvc -n cert-discovery-app
oc describe pvc cert-discovery-data -n cert-discovery-app

# Verify storage class exists
oc get sc lvms-vg1
```

### Issue: Application Route Not Accessible

**Check ingress controller:**
```bash
export KUBECONFIG=~/kubeconfig-bm-hosted-cluster

# Check route status
oc get route -n cert-discovery-app
oc describe route cert-discovery-route -n cert-discovery-app

# Check service endpoints
oc get endpoints -n cert-discovery-app

# Test from within cluster
oc run test-curl --image=curlimages/curl --rm -it --restart=Never -- \
  curl http://cert-discovery-service.cert-discovery-app.svc/health
```

---

## Key Takeaways

### What This Migration Proves

1. **Stateful Application Migration:** Successfully migrated 210MB SQLite database with 845 discovery runs and 564K certificate records
2. **Data Integrity:** 100% data preservation verified through record counts and database integrity checks
3. **HCP on Bare Metal:** Demonstrated HCP viability for on-premises infrastructure
4. **OADP Effectiveness:** Velero + Kopia successfully handled file-level PV backup/restore
5. **Workload Migration is Simple:** Standard OADP procedures work for migrating applications to HCP clusters

### HyperShift Plugin: When Do You Need It?

**You DO NOT need the HyperShift plugin for:**
- Migrating applications between clusters (this guide)
- Backing up workloads running on HCP data plane
- Standard backup/restore of user namespaces

**You DO need the HyperShift plugin for:**
- Backing up the hosted cluster's control plane (HostedCluster, HostedControlPlane resources)
- Disaster recovery of the hosted cluster infrastructure itself
- Control plane backup/restore operations on the management cluster

**Our Approach:** This guide installs OADP on both source and target clusters directly, treating the HCP cluster like any other OpenShift cluster for workload migration purposes.

### Performance Metrics

- **Backup Duration:** ~10-15 minutes (including 210MB PV data)
- **Restore Duration:** ~15-20 minutes (including PV restore)
- **Data Size:** 210MB for 3 days of historical data (845 runs)
- **Downtime:** 0 minutes (deploying to new cluster)

### Infrastructure Savings

**Scenario: 10 Production Clusters**

**Control Plane Pod Density:**
- Each hosted control plane: ~75 pods
- Max pods per node: 500
- Hosted control planes per management node: ~6

| Architecture | Nodes Required | Calculation | Savings |
|--------------|----------------|-------------|---------|
| Standalone (10 clusters × 6 nodes) | 60 nodes | 10 × 6 = 60 | Baseline |
| HCP (3 mgmt + 10 HCP × 2 workers) | 23 nodes | 3 + (10 × 2) = 23 | **62% reduction** |

**HCP Breakdown:**
- Management cluster: 3 nodes (hosts 10 control planes at ~6 per node, HA configured)
- Worker nodes: 20 nodes (10 HCP clusters × 2 workers each)

---

## Next Steps

1. **Monitor Application:** Verify background refresh continues (check logs after 4 hours)
2. **Update DNS/Load Balancers:** Point production traffic to new HCP cluster route
3. **Schedule Regular Backups:** Implement automated backup schedule for disaster recovery
4. **Migrate Additional Apps:** Use this process as template for other applications

---

## Notes

**Key Differences from Cloud Migrations:**
- Uses bare metal infrastructure instead of cloud providers
- Both source and target are on-premises clusters
- HyperShift on bare metal (not AWS ROSA or Azure ARO)

**Production Considerations:**
- Test this procedure in a non-production environment first
- Ensure you have sufficient S3 storage for backup data
- Monitor backup and restore operations closely
- Keep backups for at least 30 days for rollback capability
- Document your actual backup/restore times for SLA planning

---

**Author:** Ramon Acedo
**Date:** March 2026
