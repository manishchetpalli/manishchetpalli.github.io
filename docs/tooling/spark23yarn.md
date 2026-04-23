## Running Custom Spark Versions on YARN in HDP and CDP

This guide explains how to run a custom Apache Spark version on YARN in both Hortonworks Data Platform (HDP) and Cloudera Data Platform (CDP) environments.

---

## Overview

In managed Hadoop distributions like HDP and CDP, the cluster already ships with a bundled Spark version.

If you want to:

* Use a newer Spark version
* Use an older Spark version
* Test a custom Spark build
* Run Spark independently from the cluster-provided Spark

You can configure your own Spark installation and submit jobs to YARN.

---

## Prerequisites

* Access to cluster edge node
* Java installed
* Hadoop client installed
* HDFS access
* Permission to submit YARN jobs
* Custom Spark binary downloaded

---

## 1. Download Spark

Download the required Spark prebuilt package from the official Apache Spark downloads page.

Choose a Spark build that matches your Hadoop version.

Example:

```bash
spark-2.3.1-bin-hadoop2.7.tgz
```

Extract it:

```bash
mkdir -p /home/centos/spark
cd /home/centos/spark

tar -xvzf spark-2.3.1-bin-hadoop2.7.tgz
```

Example extracted location:

```bash
/home/centos/spark/spark-2.3.1-bin-hadoop2.7/
```

---

## 2. Copy Required Jersey Bundle Jar

Some HDP/CDP environments require the Jersey bundle jar to avoid runtime classpath issues.

Download:

```text
jersey-bundle-1.19.1.jar
```

Copy it into the Spark jars directory:

```bash
cp jersey-bundle-1.19.1.jar /home/centos/spark/spark-2.3.1-bin-hadoop2.7/jars/
```

---

## 3. Create Spark JAR Archive for YARN

YARN distributes Spark libraries using an archive.

Create a zip file containing all Spark jars:

```bash
cd /home/centos/spark/spark-2.3.1-bin-hadoop2.7/jars
zip -r spark-jars.zip .
```

Upload the archive to HDFS:

```bash
hdfs dfs -mkdir -p /user/centos/data/spark/
hdfs dfs -put spark-jars.zip /user/centos/data/spark/
```

Example HDFS location:

```bash
hdfs:///user/centos/data/spark/spark-jars.zip
```

---

## 4. Identify Hadoop Distribution Version

>--- For HDP

Get the HDP version:

```bash
hdp-select status hadoop-client
```

Example output:

```bash
hadoop-client - 3.0.1.0-187
```

Version used in this guide:

```bash
3.0.1.0-187
```

>--- For CDP

In CDP, Hadoop libraries are usually installed under Cloudera parcel directories.

Example Hadoop client path:

```bash
/opt/cloudera/parcels/CDH/lib/hadoop
```

You can identify the parcel version using:

```bash
ls /opt/cloudera/parcels/
```

or:

```bash
hadoop version
```

---

## 5. Set Environment Variables

>--- HDP Environment Variables

```bash
export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-/usr/hdp/3.0.1.0-187/hadoop/conf}
export HADOOP_HOME=${HADOOP_HOME:-/usr/hdp/3.0.1.0-187/hadoop}
export SPARK_HOME=/home/centos/spark/spark-2.3.1-bin-hadoop2.7/
export LD_LIBRARY_PATH=/usr/hdp/3.0.1.0-187/hadoop/lib/native:/usr/hdp/3.0.1.0-187/hadoop/lib/native/Linux-amd64-64
export PATH=$SPARK_HOME/bin:$PATH
```

>--- CDP Environment Variables

```bash
export HADOOP_CONF_DIR=/etc/hadoop/conf
export HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export SPARK_HOME=/home/centos/spark/spark-2.3.1-bin-hadoop2.7/
export LD_LIBRARY_PATH=/opt/cloudera/parcels/CDH/lib/hadoop/lib/native
export PATH=$SPARK_HOME/bin:$PATH
```

---

## 6. Configure spark-defaults.conf

Edit:

```bash
$SPARK_HOME/conf/spark-defaults.conf
```

Example configuration:

```properties
spark.eventLog.enabled                         true
spark.eventLog.dir                             hdfs:///spark2-history/
spark.history.fs.logDirectory                  hdfs:///spark2-history/
spark.history.provider                         org.apache.spark.deploy.history.FsHistoryProvider
spark.yarn.historyServer.address               hostname:8088
spark.port.maxRetries                          24
spark.yarn.queue                               default
spark.master                                   yarn
spark.yarn.historyServer.allowTracking         true
spark.history.fs.cleaner.enabled               true
spark.history.fs.cleaner.interval              1d
spark.history.fs.cleaner.maxAge                3d
spark.eventLog.rolling.enabled                 true
spark.eventLog.rolling.maxFileSize             128m
spark.eventLog.compress                        true
spark.eventLog.compression.codec               snappy
spark.shuffle.push.server.mergedShuffleFileManagerImpl  org.apache.spark.network.shuffle.NoOpMergedShuffleFileManager
spark.shuffle.push.server.minChunkSizeInMergedShuffleFile 2m
spark.shuffle.push.server.mergedIndexCacheSize 100m
spark.yarn.archive                             hdfs:///user/centos/data/spark/spark-jars.zip
```

>--- Additional HDP Configuration

```properties
spark.driver.extraJavaOptions                  -Dhdp.version=2.6.3.0-235
spark.yarn.am.extraJavaOptions                 -Dhdp.version=2.6.3.0-235
spark.executor.extraJavaOptions                -Dhdp.version=2.6.3.0-235
```

>--- CDP Configuration

```properties
spark.yarn.archive                             hdfs:///user/centos/data/spark/spark-jars.zip
spark.eventLog.enabled                         true
spark.history.fs.logDirectory                  hdfs:///spark-history
```

---

## 7. Configure spark-env.sh

Edit:

```bash
$SPARK_HOME/conf/spark-env.sh
```

Example:

```bash
##!/usr/bin/env bash
export HADOOP_HOME=/usr/hdp/2.6.3.0-235/hadoop
export HADOOP_CONF_DIR=/opt/spark-3.2.2-bin-hadoop2.7/conf
export JAVA_HOME=/home/spark3/zulujdk11
##export HDP_VERSION="-Dhdp.version=2.6.3.0-235"
```

You can also enable the HDP version line if required by your environment:

```bash
export HDP_VERSION="-Dhdp.version=2.6.3.0-235"
```

---

## 8. Create java-opts File

Create the file:

```bash
$SPARK_HOME/conf/java-opts
```

>--- HDP Example

```text
-Dhdp.version=3.0.1.0-187
```

This helps Spark containers pick up the HDP version properly.

---

## 8. Verify Spark Installation

Check Spark version:

```bash
$SPARK_HOME/bin/spark-shell --version
```

Check Hadoop configuration:

```bash
echo $HADOOP_CONF_DIR
echo $HADOOP_HOME
```

---

## 9. Run Spark Shell in YARN Client Mode

```bash
spark-shell \
  --master yarn \
  --deploy-mode client \
  --conf spark.yarn.archive=hdfs:///user/centos/data/spark/spark-jars.zip
```

---

## 10. Run Spark Shell in YARN Cluster Mode

```bash
spark-shell \
  --master yarn \
  --deploy-mode cluster \
  --conf spark.yarn.archive=hdfs:///user/centos/data/spark/spark-jars.zip
```

---

## 11. Submit Spark Job

Example:

```bash
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.yarn.archive=hdfs:///user/centos/data/spark/spark-jars.zip \
  $SPARK_HOME/examples/jars/spark-examples_2.11-2.3.1.jar \
  10
```

---

## 12. Common Issues and Fixes

>--- ClassNotFoundException for Jersey Classes

Fix:

* Ensure `jersey-bundle-1.19.1.jar` is copied into Spark jars folder
* Recreate and re-upload `spark-jars.zip`

>--- Native Hadoop Library Warnings

Fix:

```bash
export LD_LIBRARY_PATH=<correct_hadoop_native_library_path>
```

>--- Missing hdp.version Error

Fix:

Ensure the following are set:

```properties
spark.driver.extraJavaOptions=-Dhdp.version=<hdp_version>
spark.yarn.am.extraJavaOptions=-Dhdp.version=<hdp_version>
spark.executor.extraJavaOptions=-Dhdp.version=<hdp_version>
```

>--- YARN Containers Failing Immediately

Fix:

* Verify `spark.yarn.archive` path exists in HDFS
* Verify all Spark jars are present in archive
* Check YARN application logs

```bash
yarn logs -applicationId <application_id>
```

---

## 13. Recommended Directory Structure

```text
/home/centos/spark/
├── spark-2.3.1-bin-hadoop2.7/
│   ├── bin/
│   ├── conf/
│   ├── jars/
│   └── examples/
└── downloads/
```

---

## 14. Quick Reference Commands

```bash
## Get HDP version
hdp-select status hadoop-client

## Upload archive to HDFS
hdfs dfs -put spark-jars.zip /user/centos/data/spark/

## Start Spark shell
spark-shell --master yarn --deploy-mode client

## Submit job
spark-submit --master yarn --deploy-mode cluster

## View YARN logs
yarn logs -applicationId <application_id>
```

---

## Conclusion

Using a custom Spark version on YARN is possible in both HDP and CDP by:

1. Downloading a compatible Spark version
2. Packaging Spark jars into an archive
3. Uploading the archive to HDFS
4. Setting proper Hadoop environment variables
5. Configuring Spark to use the uploaded archive

This approach allows you to run any Spark version independent of the cluster-installed Spark version.
