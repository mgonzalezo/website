---
title: PVC
description: Deploy models to KServe InferenceService from Persistent Volume Claims (PVC), including setup and configuration.
---

# Deploy InferenceService with a saved model on PVC

In this guide, you'll learn how to store machine learning models on Persistent Volume Claims (PVC) and deploy them as KServe InferenceServices. By following these steps, you'll be able to leverage Kubernetes storage for your model serving needs, which is especially useful when working with models that need to be persisted across pod restarts or shared between different components.

## Create PV and PVC

Refer to the [document](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) to create Persistent Volume (PV) and Persistent Volume Claim (PVC), the PVC will be used to store model. This document uses local PV.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/ubuntu/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pv-and-pvc.yaml
```

### Copy model to PV

Run pod `model-store-pod` and login into container `model-store`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: model-store-pod
spec:
  volumes:
    - name: model-store
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: model-store
      image: ubuntu
      command: [ "sleep" ]
      args: [ "infinity" ]
      volumeMounts:
        - mountPath: "/pv"
          name: model-store
      resources:
        limits:
          memory: "1Gi"
          cpu: "1"
```

```bash
kubectl apply -f pv-model-store.yaml

kubectl exec -it model-store-pod -- bash
```

In different terminal, copy the model from local into PV, then delete `model-store-pod`.

```bash
kubectl cp model.joblib model-store-pod:/pv/model.joblib -c model-store

kubectl delete pod model-store-pod
```

## Deploy `InferenceService` with models on PVC

Update the `${PVC_NAME}` to the created PVC name and create the InferenceService with the PVC `storageUri`.

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-pvc"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "pvc://${PVC_NAME}/${MODEL_NAME}/"
```

Apply the `sklearn-pvc.yaml`.

```bash
kubectl apply -f sklearn-pvc.yaml
```

Note that inside the folder `${PVC_NAME}/${MODEL_NAME}/` you should have your
model `model.joblib`.

Note also that `${MODEL_NAME}` is just a folder, but a good convention to keep
the same name.

### PVC Read-Write Configuration

By default, PVC volumes are mounted in read-only mode. If you need read-write access, you can add an annotation to the InferenceService:

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-pvc"
  annotations:
    # highlight-next-line
    storage.kserve.io/readonly: "false"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "pvc://${PVC_NAME}/${MODEL_NAME}/"
```

### PVC Volume Mount Configuration

KServe uses direct PVC volume mounting, where the PVC volume is directly mounted to `/mnt/models` in the user container rather than creating a symlink from `/mnt/models` to a shared volume. This approach improves performance and simplifies the storage architecture.

## Loading model weights using a Kubernetes Job

A common pattern for large models is to populate a PVC with a Kubernetes `Job` before or after deploying the `InferenceService`. This section covers two failure modes that are not obvious from the Kubernetes or KServe documentation.

### RWO PVC race condition

When you create an `InferenceService` that references an empty or partially-populated **ReadWriteOnce (RWO)** PVC, the predictor pod starts immediately and claims **exclusive** access to that PVC. Any model-loader `Job` or init container scheduled concurrently will fail to attach the volume:

```
Multi-Attach error for volume "pvc-<uuid>": Volume is already used by pod(s) sklearn-pvc-predictor-<hash>
```

This failure is silent from the `InferenceService` perspective — the predictor pod starts, but the loader never writes the weights, so the model server fails with a missing-model error rather than a volume error.

**Workaround: scale the predictor to zero before loading**

Scale the predictor down before running the loader Job, then scale it back up once the weights are in place:

```bash
# 1. Create the InferenceService with zero replicas so the predictor does not claim the PVC
kubectl patch inferenceservice sklearn-pvc \
  --type='merge' \
  -p '{"spec":{"predictor":{"minReplicas":0,"maxReplicas":0}}}'

# 2. Run your loader Job to populate the PVC
kubectl apply -f model-loader-job.yaml
kubectl wait --for=condition=complete job/model-loader --timeout=600s

# 3. Restore replicas so the predictor can start and attach the now-populated PVC
kubectl patch inferenceservice sklearn-pvc \
  --type='merge' \
  -p '{"spec":{"predictor":{"minReplicas":1,"maxReplicas":1}}}'
```

Alternatively, create the `InferenceService` **after** the loader Job has completed and the loader pod has been deleted, so the RWO PVC is free when the predictor starts.

:::tip
If you are using a **ReadWriteMany (RWX)** PVC (e.g., NFS, CephFS, or a cloud-provider ReadWriteMany storage class), multiple pods can attach simultaneously and the race condition does not apply.
:::

### PVC permissions for loader pods (fsGroup)

When KServe creates a predictor pod it sets a **namespace-derived `fsGroup`** on the volume mount (for example `1000840000` on OpenShift). Files written to the PVC by the predictor are owned by that group.

If a loader `Job` mounts the same PVC **without the matching `fsGroup`**, it receives a `Permission denied` error when writing model weights — even with the correct `serviceAccount` and RBAC.

Set `securityContext.fsGroup` on your loader `Job` pod spec to match the namespace's UID/GID range:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-loader
spec:
  template:
    spec:
      # highlight-start
      securityContext:
        fsGroup: 1000840000   # match the namespace-derived fsGroup used by KServe
      # highlight-end
      volumes:
        - name: model-store
          persistentVolumeClaim:
            claimName: task-pv-claim
      containers:
        - name: loader
          image: python:3.11-slim
          command: ["python", "download_model.py"]
          volumeMounts:
            - mountPath: "/mnt/models"
              name: model-store
      restartPolicy: Never
```

To find the correct `fsGroup` for your namespace:

```bash
# OpenShift / OCP — read the namespace supplemental-groups annotation
kubectl get namespace <your-namespace> \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}'
# Example output: 1000840000/10000  →  use 1000840000

# Vanilla Kubernetes with PodSecurity admission
kubectl get namespace <your-namespace> \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/uid-range}'
```

Use the **first value** of the range as the `fsGroup`.

:::note
This requirement applies to any external pod — loader Jobs, init containers, or debugging pods — that mounts a PVC previously attached to a KServe `InferenceService` predictor. The KServe predictor itself sets the correct `fsGroup` automatically.
:::

## Run a prediction

Now, the ingress can be accessed at `${INGRESS_HOST}:${INGRESS_PORT}` or follow [this instruction](../../../getting-started/predictive-first-isvc.md#4-determine-the-ingress-ip-and-ports)
to find out the ingress IP and port.

### Sample Input

Here's the sample input to test the model:

```json title="input.json"
{
    "instances": [
      [6.8,  2.8,  4.8,  1.4],
      [6.0,  3.4,  4.5,  1.6]
    ]
}
```

### Making the request

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-pvc -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=sklearn-pvc
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" -H "Content-Type: application/json" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

:::tip Expected Output

```bash
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> POST /v1/models/sklearn-pvc:predict HTTP/1.1
> Host: sklearn-pvc.default.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 84
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 84 out of 84 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 23
< content-type: application/json; charset=UTF-8
< date: Mon, 20 Sep 2021 04:55:50 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 6
<
* Connection #0 to host localhost left intact
{"predictions": [1, 1]}
```

:::
