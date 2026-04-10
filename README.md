# RKE2 etcd Backup with Kasten K10 (Kanister Blueprint)

## 📌 Overview

This project implements a **custom backup strategy for RKE2 etcd** using:

* Kasten K10
* Kanister Blueprint
* Kopia (via Kasten Location Profile)

The goal is to provide a **reliable, restorable, and automated backup mechanism** for the Kubernetes control plane datastore (**etcd**), which is **not natively handled as an application workload**.

---

## 🎯 Objectives

* Perform **etcd snapshots directly from RKE2**
* Package snapshots into a **portable archive**
* Store backups securely using **Kasten + Kopia (S3/MinIO)**
* Allow **restore staging from Kasten UI**
* Keep the process **simple, auditable, and reproducible**

---

## 🏗️ Architecture

```text
Kasten Policy
    ↓
Kanister Blueprint
    ↓
RKE2 etcd snapshot
    ↓
Archive (.tar.gz)
    ↓
Kopia repository (via Location Profile)
    ↓
Restore via Kasten UI
```

---

## 📁 Repository Structure

```text
.
├── blueprint/
│   └── rke2-etcd-blueprint.yaml
├── policy/
│   └── rke2-etcd-policy.yaml
├── k8s/
│   ├── namespace.yaml
│   └── secret.yaml
└── README.md
```

---

## ⚙️ Prerequisites

* RKE2 cluster (multi-node or single control-plane)
* Kasten K10 installed
* A configured **Location Profile** (S3-MinIO for exemple)
* Access to control-plane nodes (privileged operations required)

---

## 🚀 Deployment Steps

### 1. Create namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

---

### 2. Create trigger secret

This secret is used by Kasten to trigger the Blueprint.

```bash
kubectl apply -f k8s/secret.yaml
```

Example content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rke2-etcd-details
  namespace: etcd-backup-rke2
type: Opaque
stringData:
  snapshotDir: "/var/lib/rancher/rke2/server/db/snapshots"
```

---

### 3. Apply Blueprint

```bash
kubectl apply -f blueprint/rke2-etcd-blueprint.yaml
```

---

### 4. Apply Policy

```bash
kubectl apply -f policy/rke2-etcd-policy.yaml
```

---

## 🔁 Backup Workflow

Each execution performs:

1. etcd snapshot via `rke2 etcd-snapshot save`
2. Packaging:

   * `snapshot.db`
   * `snapshot.sha256`
   * `token`
   * metadata
3. Archive creation (`.tar.gz`)
4. Upload to **Kopia repository (Location Profile)**
5. Cleanup of local files

---

## 📦 Backup Storage

Backups are stored in the configured Location Profile:

* Backend: S3 / MinIO
* Format: **Kopia repository (deduplicated & compressed)**
* Not directly visible as plain `.tar.gz` objects

---

## 🔄 Restore Process

Restore is performed via **Kasten UI**:

1. Select a Restore Point
2. Trigger Restore Action
3. Files are staged on a control-plane node:

```text
/var/lib/rancher/rke2/server/db/restore/
```

Contents:

* snapshot.db
* token
* metadata
* restore instructions

---

## 🛠️ Manual Restore (RKE2)

On a control-plane node:

```bash
systemctl stop rke2-server

sha256sum -c snapshot.sha256

rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path /var/lib/rancher/rke2/server/db/restore/snapshot.db

systemctl start rke2-server
```

For HA clusters:

* Stop other control-plane nodes
* Remove `/var/lib/rancher/rke2/server/db`
* Restart nodes

---

## 🔐 Security Considerations

* Backups include the **RKE2 token**

* Access to:

  * Kasten UI
  * Location Profile
  * Object storage
    must be strictly controlled

* Kopia provides:

  * encryption
  * deduplication

---

## ⚠️ Limitations

* Restore node is not displayed in Kasten UI
* Restore is **staged only**, not automatically applied
* Full cluster recovery remains a **manual operation**

---

## 🧪 Validation Status

* Backup execution: ✅
* Kopia storage: ✅
* Restore staging: ✅
* Manual restore procedure: ⚠️ (not executed in production)

---

## 📬 Review Request

This solution is provided for validation regarding:

* best practices for etcd backup with Kasten
* security considerations
* restore reliability

---

**Author:** Clément Legrand
**Context:** RKE2 + Kasten + MinIO lab environment
