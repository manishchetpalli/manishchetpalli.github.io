##  Deploy Spark on Kubernetes (On-Prem Cluster)

This document explains how to deploy and run Apache Spark on an on-premises Kubernetes cluster using the details from your environment.

Reference:

* Spark Kubernetes Documentation: [https://spark.apache.org/docs/3.5.4/running-on-kubernetes.html](https://spark.apache.org/docs/3.5.4/running-on-kubernetes.html)

---

##  Step 1: Verify Kubernetes Cluster Connectivity

Run the following command to confirm that the Kubernetes cluster is reachable:

```bash
kubectl cluster-info
```

Expected output:

```bash
Kubernetes control plane is running at https://kubelb.prod.local:16443

CoreDNS is running at https://kubelb.prod.local:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

If needed, collect more debugging details:

```bash
kubectl cluster-info dump
```

---

##  Step 2: Create Namespace for Spark

Create a dedicated namespace where all Spark resources will run.

```bash
kubectl create namespace k8-spark
```

Verify:

```bash
kubectl get namespaces
```

---

##  Step 3: Create Service Account

Spark driver pods need a Kubernetes service account to create executor pods.

```bash
kubectl create serviceaccount k8-spark-serviceaccount -n k8-spark
```

Verify:

```bash
kubectl get serviceaccount -n k8-spark
```

Expected output:

```bash
NAME                        SECRETS   AGE
k8-spark-serviceaccount     1         10s
```

---

##  Step 4: Create Persistent Volume Claims

You are using two PVCs:

1. `k8-spark-code-jar`

   * Stores Spark application JARs.

2. `k8-spark-dependency-jar`

   * Stores dependency JARs.

Verify existing PVCs:

```bash
kubectl get pvc -n k8-spark
```

Expected output:

```bash
NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES
k8-spark-code-jar          Bound    pvc-xxxx                                   10Gi       RWX
k8-spark-dependency-jar    Bound    pvc-yyyy                                   20Gi       RWX
```

---

##  Step 5: Copy Required JARs to PVC Mount Path

Ensure the following files are present inside the mounted PVC locations:

>--- Application JAR

```bash
/opt/spark/work-dir/spark-examples_2.12-3.2.2.jar
```

>--- Dependency JARs

```bash
/home/kafka-clients-2.0.0.jar
/home/kafka_2.11-2.0.0.jar
/home/library_rabf-1.1.0.jar
/home/mysql-connector-java-8.0.18.jar
/home/ngdbc-2.7.14.jar
/home/scalaj-http.jar
/home/slf4j-api-1.7.25.jar
/home/spark-sql-kafka-0-10_2.11-2.2.1.jar
/home/spark-streaming-kafka-0-10_2.11-2.4.3.jar
```

---

##  Step 6: Ensure Spark Docker Image is Available

Your Spark jobs use the following image:

```bash
sjdcretdevops01.ril.com:8899/spark:v3.3.2
```

Verify that Kubernetes worker nodes can pull this image.

You can test it using:

```bash
kubectl run image-test \
  --image=sjdcretdevops01.ril.com:8899/spark:v3.3.2 \
  -n k8-spark
```

Check pod status:

```bash
kubectl get pods -n k8-spark
```

---

##  Step 7: Submit Spark Job

Run the following command from the node where Spark is installed:

```bash
spark-submit \
  --master k8s://https://kubelb.prod.local:16443 \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.executor.instances=2 \
  --conf spark.driver.memory=1g \
  --conf spark.executor.memory=1g \
  --conf spark.kubernetes.container.image=sjdcretdevops01.ril.com:8899/spark:v3.3.2 \
  --conf spark.kubernetes.namespace=k8-spark \
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.code-pvc-jar.mount.path=/opt/spark/work-dir \
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.code-pvc-jar.options.claimName=k8-spark-code-jar \
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.code-pvc-jar.mount.path=/opt/spark/work-dir \
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.code-pvc-jar.options.claimName=k8-spark-code-jar \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=k8-spark-serviceaccount \
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.dependency-pvc-jar.options.claimName=k8-spark-dependency-jar \
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.dependency-pvc-jar.mount.path=/home \
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.dependency-pvc-jar.mount.path=/home \
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.dependency-pvc-jar.options.claimName=k8-spark-dependency-jar \
  --jars local:///home/kafka-clients-2.0.0.jar,local:///home/kafka_2.11-2.0.0.jar,local:///home/library_rabf-1.1.0.jar,local:///home/mysql-connector-java-8.0.18.jar,local:///home/ngdbc-2.7.14.jar,local:///home/scalaj-http.jar,local:///home/slf4j-api-1.7.25.jar,local:///home/spark-sql-kafka-0-10_2.11-2.2.1.jar,local:///home/spark-streaming-kafka-0-10_2.11-2.4.3.jar \
  local:///opt/spark/work-dir/spark-examples_2.12-3.2.2.jar
```

---

##  Step 8: Verify Spark Pods

Check whether the Spark driver and executor pods are created:

```bash
kubectl get pods -n k8-spark
```

Example output:

```bash
NAME                                  READY   STATUS    RESTARTS   AGE
spark-pi-driver                       1/1     Running   0          30s
spark-pi-xxxxxxxxxx-exec-1            1/1     Running   0          20s
spark-pi-xxxxxxxxxx-exec-2            1/1     Running   0          20s
```

---

##  Step 9: Check Driver Logs

View the Spark driver logs:

```bash
kubectl logs -f spark-pi-driver -n k8-spark
```

If the job completes successfully, you should see output from the SparkPi application.

---

##  Step 10: Troubleshooting

>--- Check Pod Details

```bash
kubectl describe pod spark-pi-driver -n k8-spark
```

>--- Check Events

```bash
kubectl get events -n k8-spark
```

>--- Common Errors

>---##  Image Pull Failure

```bash
ImagePullBackOff
```

Reason:

* Docker image is not reachable.
* Registry credentials are missing.

>---##  PVC Mount Failure

```bash
Unable to attach or mount volumes
```

Reason:

* PVC does not exist.
* Wrong claim name used.

>---##  Permission Error

```bash
Forbidden: User cannot create pods
```

Reason:

* Service account does not have required permissions.

---

##  Step 11: Cleanup

Delete Spark pods after testing:

```bash
kubectl delete pods --all -n k8-spark
```

Delete namespace if no longer required:

```bash
kubectl delete namespace k8-spark
```
