## Deploy Flink on Kubernetes (On-Prem Cluster)

This document explains how to deploy Apache Flink on an on-premises Kubernetes cluster using session mode with separate JobManager and TaskManager deployments.

---

## Step 1: Create Namespace

Create a dedicated namespace for Flink.

```bash
kubectl create namespace flinktest
```

Verify:

```bash
kubectl get namespaces
```

---

## Step 2: Create Service Account

Create a service account for Flink.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flinktest-account
  namespace: flinktest
EOF
```

Verify:

```bash
kubectl get serviceaccount -n flinktest
```

Expected output:

```bash
NAME                     SECRETS   AGE
flinktest-account    1         10s
```

---

## Step 3: Create Cluster Role Binding

Grant the service account cluster-admin permissions.

```bash
kubectl create clusterrolebinding flinktest-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=flinktest:flinktest-account
```

Verify:

```bash
kubectl get clusterrolebinding | grep flinktest
```

---

## Step 4: Create Flink ConfigMap

Create a file named `flink-flinktest-configmap.yaml`.

Important settings in this ConfigMap include:

* JobManager RPC address
* TaskManager slots
* Memory settings
* HA configuration
* Checkpoint directory
* Upload directory
* Log path
* Restart strategy
* Web UI settings

Apply the ConfigMap:

```bash
kubectl apply -f flink-flinktest-configmap.yaml -n flinktest
```

Verify:

```bash
kubectl get configmap -n flinktest
```

Expected output:

```bash
NAME                     DATA   AGE
flink-flinktest-config    2      10s
```

---

## Step 5: Create JobManager Service

Create a file named `jobmanager-flinktest-service.yaml`.

This service exposes:

* RPC Port: 6123
* Blob Server Port: 6124
* Web UI Port: 8081

Apply the service:

```bash
kubectl apply -f jobmanager-flinktest-service.yaml -n flinktest
```

Verify:

```bash
kubectl get svc -n flinktest
```

Expected output:

```bash
NAME                TYPE        CLUSTER-IP      PORT(S)
flink-jobmanager    ClusterIP   10.97.84.146   6123/TCP,6124/TCP,8081/TCP
```

---

## Step 6: Deploy JobManager

Create a file named `jobmanager-deployment-flinktest.yaml`.

Important details in your JobManager deployment:

* Image:

```bash
flinkkrb5kdc_1.18v2:latest
```

* JobManager memory:

```bash
jobmanager.memory.process.size: 8192m
```

* Mounted directories:

```bash
/NAS/rradlk_flink_storage/flinktest
/NAS/flink_depjars_1.18.1
/NAS/kafka_acl/pvtlinktruststore.jks
/NAS/fusion3_krb/krb5.conf
/NAS/newjks
```

Apply the JobManager deployment:

```bash
kubectl apply -f jobmanager-deployment-flinktest.yaml -n flinktest
```

Verify:

```bash
kubectl get deployments -n flinktest
kubectl get pods -n flinktest
```

---

## Step 7: Deploy TaskManager

Create a file named `taskmanager-deployment-flinktest.yaml`.

Important details in your TaskManager deployment:

* Image:

```bash
flinkkrb5kdc_1.18v2:latest
```

* TaskManager memory:

```bash
taskmanager.memory.process.size: 20480m
```

* Number of task slots:

```bash
taskmanager.numberOfTaskSlots: 14
```

Apply the TaskManager deployment:

```bash
kubectl apply -f taskmanager-deployment-flinktest.yaml -n flinktest
```

Verify:

```bash
kubectl get deployments -n flinktest
kubectl get pods -n flinktest
```

Expected output:

```bash
NAME                                READY   STATUS    RESTARTS   AGE
flink-jobmanager-xxxxxxxxxx         1/1     Running   0          1m
flink-taskmanager-xxxxxxxxxx        1/1     Running   0          1m
```

---

## Step 8: Verify Role Bindings

Check role bindings:

```bash
kubectl get rolebinding -n flinktest
```

Check cluster role bindings:

```bash
kubectl get clusterrolebinding
```

---

## Step 9: Access Kubernetes Dashboard

Generate dashboard token:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Port forward dashboard:

```bash
nohup kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard --address IP.IP68.209 8443:443 > /dev/null 2>&1 &
```

Access dashboard:

```bash
https://IP.IP68.209:8443
```

---

## Step 10: Access Flink UI

Port forward Flink JobManager UI:

```bash
kubectl port-forward svc/flink-jobmanager 8081:8081 -n flinktest
```

Or run in background:

```bash
nohup kubectl -n flinktest port-forward svc/flink-jobmanager --address IP.IP68.209 8081:8081 > /dev/null 2>&1 &
```

Access Flink UI:

```bash
http://IP.IP68.209:8081
```

---

## Step 11: Verify Flink Cluster

Check pods:

```bash
kubectl get pods -n flinktest
```

Check services:

```bash
kubectl get svc -n flinktest
```

Check deployments:

```bash
kubectl get deployments -n flinktest
```

Check logs:

```bash
kubectl logs -f <jobmanager-pod-name> -n flinktest
kubectl logs -f <taskmanager-pod-name> -n flinktest
```

Describe pods:

```bash
kubectl describe pod <pod-name> -n flinktest
```

---

## Step 12: Common Issues

>--- Image Pull Failure

```bash
ImagePullBackOff
```

Possible reasons:

* Image registry is not reachable.
* Missing image pull secret.
* Wrong image name.

---

>--- HostPath Mount Failure

```bash
MountVolume.SetUp failed
```

Possible reasons:

* Host directory does not exist on worker node.
* Wrong hostPath configured.

Required host paths in your setup:

```bash
/NAS/rradlk_flink_storage/flinktest
/NAS/flink_depjars_1.18.1
/NAS/kafka_acl/pvtlinktruststore.jks
/NAS/fusion3_krb/krb5.conf
/NAS/newjks
```

---

>--- Service Account Permission Failure

```bash
Forbidden: User cannot create configmaps
```

Possible reason:

* Cluster role binding not created correctly.

---

## Step 13: Cleanup

Delete deployments:

```bash
kubectl delete deployment flink-jobmanager -n flinktest
kubectl delete deployment flink-taskmanager -n flinktest
```

Delete service:

```bash
kubectl delete svc flink-jobmanager -n flinktest
```

Delete ConfigMap:

```bash
kubectl delete configmap flink-flinktest-config -n flinktest
```

Delete namespace:

```bash
kubectl delete namespace flinktest
```
