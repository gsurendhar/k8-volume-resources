# Kubernetes Volume Resources: EBS & EFS Provisioning
---

* This repository contains basic to intermediate-level notes on how Kubernetes handles storage using **Persistent Volumes**, **Claims**, and **StorageClasses**, with examples focused on  provisioning **Amazon Elastic Block Store (EBS)** and **Elastic File System (EFS)** volumes in a Kubernetes cluster, both statically and dynamically. These methods allow your pods to have persistent storage in the cloud using Amazon Web Services (AWS).
---
## Table of Contents

* [Types of Storage](#types-of-storage)
* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Static EBS Provisioning](#static-ebs-provisioning)
* [Dynamic EBS Provisioning](#dynamic-ebs-provisioning)
* [Static EFS Provisioning](#static-efs-provisioning)
* [Dynamic EFS Provisioning](#dynamic-efs-provisioning)
* [Troubleshooting](#troubleshooting)
* [References](#references)

## 📦 Types of Storage

### 1. **Hard Disk (EBS - Elastic Block Store)**

- Comparable to an external hard drive.
- Must be in the **same Availability Zone (AZ)** as the compute instance (e.g., EC2).
- Offers **high-speed** data transfer.
- Can store **Operating Systems** and **Databases**.
- Can be mounted to **only one instance** at a time (`ReadWriteOnce`).

### 2. **Drive (EFS - Elastic File System / NFS)**

- Accessible **over the internet or VPC**.
- Can be mounted to **multiple instances** simultaneously (`ReadWriteMany`).
- Suitable for **file storage**, **shared config**, or **web assets**.
- **Cannot** store Operating Systems or Databases.
- Slower than EBS in most scenarios.
---
## 🔐 Storage Administration Concepts

- **Backup**: Regular data snapshots or archiving.
- **Security**: Encryption, IAM policies, and access control.
- **Restore**: Recover from backup in case of failures.
- **Replication**: Copying data across regions or AZs for high availability.
---
## Overview

This guide walks through two types of volume provisioning strategies:

1. **Static Provisioning:** The volume is manually created beforehand and is then referenced by the Kubernetes Persistent Volume (PV) and Persistent Volume Claim (PVC).
2. **Dynamic Provisioning:** The volume is automatically created by a StorageClass and associated with the PVC at the time of pod deployment.

## Prerequisites

Before you begin, ensure you have:

* A working **Kubernetes Cluster** (v1.16+).
* AWS IAM permissions for **EBS** and **EFS** management.
* The **EBS CSI Driver** and **EFS CSI Driver** installed.
* A **Kubernetes EC2 instance** in the same AWS region as your volumes.
* **kubectl** CLI configured to access your Kubernetes cluster.

Refer to the official documentation for [EBS CSI Driver](https://kubernetes.io/docs/concepts/storage/volumes/#ebs) and [EFS CSI Driver](https://kubernetes.io/docs/concepts/storage/volumes/#efs) to help with setup and configuration.

---

## ⚙️ Kubernetes Storage Resources

| Component | Description |
|----------|-------------|
| `PV`     | Persistent Volume — represents the **actual storage** resource (EBS, NFS, etc.) |
| `PVC`    | Persistent Volume Claim — a **request** for storage by a user or Pod |
| `SC`     | StorageClass — defines **how** storage is provisioned (static/dynamic) |

---

## 📂 Storage Provisioning Types

# Static EBS Provisioning

* 1. Install EBSCSIDriver
* 2. Provide access to EC2 instance through role EBSCSIDriverPolicy
* 3. Create Volume in same AZ as in EC2 Instance 
* 4. Create PV, PVC and create pod with nodeSelector option and volumeMount

# Dynamic EBS Provisioning

* 1. Install EBSCSIDriver
* 2. Provide access to EC2 instance through role EBSCSIDriverPolicy
* 3. Create StorageClass (Manual creation of volume is not required and StorageClass is the resource that creates disk automatically)
* 4. Create PVC and create pod with nodeSelector option and volumeMount

# Static EFS Provisioning

* 1. Install EFSCSIDriver
* 2. Provide access to EC2 instance through role EFSCSIDriverPolicy
* 3. Create EFS Volume
* 4. In EFS Security Group Add 2049 from Instance so add source as Instance_id
* 5. Create PV, PVC and mount to pod

# Dynamic EFS Provisioning

* 1. Install EFSCSIDriver
* 2. Provide access to EC2 instance through role EFSCSIDriverPolicy
* 3. Create StorageClass (Manual creation of volume is not required and StorageClass is the resource that creates disk automatically)
* 4. In EFS Security Group Add 2049 from Instance so add source as Instance_id
* 5. Create PV, PVC and mount to pod

---

## Static EBS Provisioning

### Steps

1. **Install the EBS CSI Driver**

   Follow the official AWS documentation to install the [EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

 ```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.43"
```

2. **Provide Access to EC2 Instance**

   Ensure the EC2 instance has the necessary IAM role to access the EBS resources:

   * Attach `EBSCSIDriverPolicy` to the EC2 instance or use an existing policy with permissions to manage EBS.

3. **Create an EBS Volume**

   Create the EBS volume in the same Availability Zone (AZ) as your EC2 instance using the AWS Console or CLI:

   ```bash
   aws ec2 create-volume --size 10 --availability-zone <AZ> --volume-type gp2
   ```

4. **Create Persistent Volume (PV)**

   Define the Persistent Volume (PV) manifest with details about the volume:

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: ebs-pv
   spec:
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: ebs-sc
     csi:
       driver: ebs.csi.aws.com
       fsType: ext4
       volumeHandle: <volume-id>
   ```

5. **Create Persistent Volume Claim (PVC)**

   Define a PVC manifest to request the volume:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: ebs-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: ebs-sc
     resources:
       requests:
         storage: 10Gi
   ```

6. **Create Pod with nodeSelector and Volume Mount**

   Create a pod that uses the PVC:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: ebs-pod
   spec:
     nodeSelector:
       kubernetes.io/hostname: <node-name>
     volumes:
       - name: ebs-volume
         persistentVolumeClaim:
           claimName: ebs-pvc
     containers:
       - name: nginx
         image: nginx
         volumeMounts:
           - mountPath: /data
             name: ebs-volume
   ```

---

## Dynamic EBS Provisioning

### Steps

1. **Install the EBS CSI Driver**

   Follow the official AWS documentation to install the [EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

2. **Provide Access to EC2 Instance**

   Attach the `EBSCSIDriverPolicy` IAM role to your EC2 instance.

3. **Create a StorageClass**

   Define a StorageClass that will automatically provision EBS volumes when requested by a PVC:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ebs-sc
   provisioner: ebs.csi.aws.com
   ```

4. **Create PVC and Pod**

   Create a PVC to request storage and a pod to use it:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: ebs-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: ebs-sc
     resources:
       requests:
         storage: 10Gi
   ```

   Then, define a pod that uses the PVC:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: ebs-pod
   spec:
     nodeSelector:
       kubernetes.io/hostname: <node-name>
     volumes:
       - name: ebs-volume
         persistentVolumeClaim:
           claimName: ebs-pvc
     containers:
       - name: nginx
         image: nginx
         volumeMounts:
           - mountPath: /data
             name: ebs-volume
   ```

---

## Static EFS Provisioning

### Steps

1. **Install the EFS CSI Driver**

   Follow the official AWS documentation to install the [EFS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html).

  ```
   kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-2.1" > private-ecr-driver.yaml
   ```
   ```
   kubectl apply -f private-ecr-driver.yaml
   ```

2. **Provide Access to EC2 Instance**

   Attach the `EFSCSIDriverPolicy` IAM role to your EC2 instance.

3. **Create EFS Volume**

   Create the EFS file system in the AWS Console or CLI:

   ```bash
   aws efs create-file-system --creation-token <your-token>
   ```

4. **Configure Security Groups**

   Add an inbound rule to the EFS security group to allow traffic from your EC2 instance on port 2049:

   ```bash
   aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 2049 --source-group <ec2-sg-id>
   ```

5. **Create Persistent Volume (PV) for EFS**

   Define the PV manifest:

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: efs-pv
   spec:
     capacity:
       storage: 10Gi
     volumeMode: Filesystem
     accessModes:
       - ReadWriteMany
     persistentVolumeReclaimPolicy: Retain
     storageClassName: efs-sc
     csi:
       driver: efs.csi.aws.com
       volumeHandle: <efs-id>
   ```

6. **Create PVC and Mount to Pod**

   Define a PVC manifest and pod that mounts the EFS volume:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: efs-pvc
   spec:
     accessModes:
       - ReadWriteMany
     storageClassName: efs-sc
     resources:
       requests:
         storage: 10Gi
   ```

   Then, define a pod that mounts the EFS volume:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: efs-pod
   spec:
     volumes:
       - name: efs-volume
         persistentVolumeClaim:
           claimName: efs-pvc
     containers:
       - name: nginx
         image: nginx
         volumeMounts:
           - mountPath: /data
             name: efs-volume
   ```

---

## Dynamic EFS Provisioning

### Steps

1. **Install the EFS CSI Driver**

   Follow the official AWS documentation to install the [EFS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html).

2. **Provide Access to EC2 Instance**

   Attach the `EFSCSIDriverPolicy` IAM role to your EC2 instance.

3. **Create StorageClass**

   Define a StorageClass for dynamic provisioning of EFS volumes:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: efs-sc
   provisioner: efs.csi.aws.com
   ```

4. **Configure Security Groups**

   Add inbound rule to EFS security group for EC2 access on port 2049.

5. **Create PVC and Pod**

   Create the Persistent Volume Claim (PVC) for dynamic provisioning of the EFS volume, and define a pod that uses the PVC.

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: efs-pvc
   spec:
     accessModes:
       - ReadWriteMany
     storageClassName: efs-sc
     resources:
       requests:
         storage: 10Gi
   ```

   Then, define a pod that mounts the EFS volume:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: efs-pod
   spec:
     volumes:
       - name: efs-volume
         persistentVolumeClaim:
           claimName: efs-pvc
     containers:
       - name: nginx
         image: nginx
         volumeMounts:
           - mountPath: /data
             name: efs-volume
   ```

The above steps ensure that the EFS volume is dynamically created and mounted to the pod via the PVC.

---

## Troubleshooting

If you encounter issues, here are a few common areas to check:

1. **IAM Permissions**

   * Ensure the EC2 instance has the appropriate IAM policies attached (`EBSCSIDriverPolicy` for EBS, `EFSCSIDriverPolicy` for EFS).
   * Check if the IAM role has the `eks:DescribeCluster` permission if using EFS.

2. **Security Group Configuration**

   * Make sure the security group for your EFS or EC2 instance allows traffic on port `2049`.
   * Ensure that the source security group or EC2 instance ID is correctly specified for ingress rules.

3. **Storage Class Issues**

   * Verify that the `StorageClass` name in your PVC matches the `StorageClass` defined in your cluster.

4. **Volume Size**

   * If you're using dynamic provisioning, ensure your PVC's requested storage size matches the limits defined in your StorageClass or is within allowable limits for the cloud provider.

5. **EBS/ EFS Availability Zone Matching**

   * For EBS, ensure that the volume is created in the same Availability Zone as the EC2 instance or node.

---

## References

* [EBS CSI Driver](https://kubernetes.io/docs/concepts/storage/volumes/#ebs)
* [EFS CSI Driver](https://kubernetes.io/docs/concepts/storage/volumes/#efs)
* [AWS EBS CSI Documentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
* [AWS EFS CSI Documentation](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
* [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

---

## GitHub Repository

For more details or to contribute, visit the GitHub repository:

[https://github.com/gsurendhar/k8-volume-resources](https://github.com/gsurendhar/k8-volume-resources)

---

This README should now provide a complete guide for both static and dynamic provisioning for EBS and EFS in Kubernetes, along with all the necessary configuration files and steps. Let me know if you need any further adjustments!
