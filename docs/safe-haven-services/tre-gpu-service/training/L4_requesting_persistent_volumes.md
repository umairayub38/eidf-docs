# Requesting persistent volumes With Kubernetes

## Requirements

It is recommended that users complete [Getting started with Kubernetes](../L1_getting_started/#requirements) before proceeding with this tutorial.

## Overview

!!! important "BeeGFS - ReadWriteMany Volume Recommendation*"

    As of May 2025, the GPU cluster has migrated to **BeeGFS** The new default, **BeeGFS**, supports `ReadWriteMany`, enabling multiple pods to mount and use the same volume concurrently.

Pods in the K8s TRE GPU Cluster are intentionally ephemeral.

They only last as long as required to complete the task that they were created for.

Keeping pods ephemeral ensures the cluster resources are released for other users to request.

However, this means the default storage volumes within a pod are temporary.

If multiple pods require access to the same large data set or they output large files, then computationally costly file transfers need to be included in every pod instance.

K8s allows you to request persistent volumes that can be mounted to multiple pods to share files or collate outputs.

These persistent volumes will remain even if the pods they are mounted to are deleted, are updated or crash.

## Predefined BeeGFS Persistent Volume Claims (PVCs)

The following **predefined PVCs** are available in every project namespace:

| BeeGFS Path                                                  | PVC Name                              | Mount in Container | Use Case                                       |
|--------------------------------------------------------------|----------------------------------------|--------------------|------------------------------------------------|
| `/mnt/beegfs/<project_id>/shared`                            | `pvc-<project_id>-shared`              | `/safe_data`       | Shared project data (read-only or read-write)  |
| `/mnt/beegfs/<project_id>/users/<username>/outputs_<job_id>` | `pvc-<project_id>-users-<username>`    | `/safe_outputs`    | User output files (read-write)                |
| `~/scratch_<job_id>`                                         | *(not a PVC; uses emptyDir)*           | `/scratch`         | Temporary scratch space (deleted after job)   |

These PVCs are automatically provisioned and do not require user creation.

## Mounting BeeGFS Volumes to a Pod

To use BeeGFS-based shared and output directories in a container, add `volumeMounts` and `volumes` entries to your pod/job YAML.

### Example pod specification yaml with mounted persistent volume

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: beegfs-shared-output-job-
  labels:
    kueue.x-k8s.io/queue-name: <project namespace>-user-queue
spec:
  completions: 1
  backoffLimit: 1
  ttlSecondsAfterFinished: 1800
  template:
    metadata:
      name: beegfs-pod
    spec:
      containers:
        - name: job-container
          image: busybox
          args: ["sleep", "infinity"]
          resources:
            requests:
              cpu: 2
              memory: "1Gi"
            limits:
              cpu: 2
              memory: "4Gi"
          volumeMounts:
            - mountPath: /safe_data
              name: shared-data
              readOnly: true
            - mountPath: /safe_outputs
              name: user-output
            - mountPath: /scratch
              name: scratch
      restartPolicy: Never
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: pvc-<project_id>-shared
        - name: user-output
          persistentVolumeClaim:
            claimName: pvc-<project_id>-users-<username>
        - name: scratch
          emptyDir: {}
```
✅ **You do not need to create** the PVCs `pvc-<project_id>-shared` and `pvc-<project_id>-users-<username>`. These are already available for you in your namespace.

