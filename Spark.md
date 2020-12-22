


## Spark-submit locally
Start hdfs, yarn.
```
spark-submit \
--class=simplifiedTrending.findAverageRank 
--conf "spark.eventlog.enabled=false"  \
target/scala-2.11/simplifiedtrending_2.11-0.1.jar sample_data_large.csv output

```

## Google Cloud DataProc 

#### Command clipboard:
gsutil cp tracktrendsovertime_2.11-0.1.jar gs://d8trends


gcloud dataproc jobs submit spark \
--cluster=cluster-d8trends \
--class=queries.SampleQueries \
--jars=gs://d8trends/simplified-trending_2.11-0.1.jar \
--region=us-central1 \
-- #LifeGoesOnWithBTS gs://d8trends/input/ gs://d8trends/output2

---

#### Problem:
Tried to run a jar on Dataproc and got this error: Job failed with message [java.lang.ClassNotFoundException

#### Solution: 
Found this github [issue](https://github.com/googleapis/java-logging/issues/276): Dataproc java.lang.ClassNotFoundException: io.grpc.internal.BaseDnsNameResolverProvider #276
which suggested adding these dependencies:
```
libraryDependencies += "com.google.cloud" % "google-cloud-logging" % "1.102.0" exclude("io.grpc", "grpc-alts")
libraryDependencies += "io.grpc" % "grpc-alts" % "1.29.0"
```

---

#### Problem: Java.IO.Exception: Unexpected Exception type
[error log](https://gist.github.com/dannyhlee/7f605cfeb5d8f5e68d898efea8c5d5a5)

#### Solution:
from this [SO post](https://stackoverflow.com/questions/54893850/apache-spark-throwing-deserialization-error-when-using-take-method-on-rdd)

|  | was | changed to |
|:---:|:---:|:---:|
|Scala | 2.12.10 | 2.11.12 |
|SparkSQL | 3.0.1 | 2.2.1 |
|SBT | 1.3.13 | 1.2.0 |

---


## Write file to HDFS from Spark

#### Resolves Exception:
`Exception in thread "main" org.apache.hadoop.security.AccessControlException: Permission denied: user="spark", access=WRITE, inode="/user/spark":spark:spark:drwxrwxr-x`

---

#### Get HDFS port:
```
$ hdfs getconf -confKey fs.defaultFS
hdfs://localhost:9000
```

#### Create user's directory on HDFS
```
$ hdfs dfs -mkdir /user/<username>
$ hdfs dfs -chmod g+w /user/<username>
```

#### Scala code (eg: Runner.scala)
Write file:
```
df.rdd.saveAsTextFile("hdfs://localhost:9000/user/<username>/<folder or file name>")
```
Read File:
```
//RDD
val rdd = spark.sparkContext.textFile("hdfs://localhost:9000/user/<filename>")
val rdd = spark.sparkContext.wholeTextFiles("hdfs://localhost:9000/user/<filename>)
val df = spark.read.text("hdfs://localhost:9000/user/<filename>")
val ds = spark.read.textFile("hdfs://localhost:9000/user/<filename>")
```

#### Add environment variable 
**warning**: do not use quotes around `<username>`

##### In Intellij Run Menu (->Configurations->Environment variables)
```HADOOP_USER_NAME=<username>```
 
##### As unix shell environment variable
```export HADOOP_USER_NAME=<username>```

---

## Delete HDFS file or directory from Spark

```
import java.net.URI
import org.apache.hadoop.fs.{FileSystem, Path}

 val fs = FileSystem.get(new URI("hdfs://localhost:9000/"), spark.sparkContext.hadoopConfiguration)
 println(spark.sparkContext.hadoopConfiguration)
 // output:
 // Configuration: core-default.xml, core-site.xml, mapred-default.xml, mapred-site.xml, yarn-default.xml, yarn-site.xml, hdfs-default.xml, hdfs-site.xml, __spark_hadoop_conf__.xml

 val fsStatus = fs.listStatus(new Path("hdfs://localhost:9000/"))
 fsStatus.foreach(x=> println(x.getPath))
 // output:
 // hdfs://localhost:9000/tmp
 // hdfs://localhost:9000/user

 val outputPath = new Path("/user/spark/trends")
 println(fs.exists(outputPath), outputPath)
 // output: (on success, ie. file existed)
 // (true,/user/spark/trends)
 
 if (fs.exists(outputPath))
   fs.delete(outputPath, true)
```
---

## Create SparkSQL Schema for Nested JSON objects using StructType (DataFrames)

##### Spark generated schema  `.printSchema()`
```
     root  
 |-- as_of: string (nullable = true)  
 |-- created_at: string (nullable = true)  
 |-- locations: array (nullable = true)  
 |    |-- element: struct (containsNull = true)  
 |    |    |-- name: string (nullable = true)  
 |    |    |-- woeid: long (nullable = true)  
 |-- trends: array (nullable = true)  
 |    |-- element: struct (containsNull = true)  
 |    |    |-- name: string (nullable = true)  
 |    |    |-- promoted_content: string (nullable = true)  
 |    |    |-- query: string (nullable = true)  
 |    |    |-- tweet_volume: long (nullable = true)  
 |    |    |-- url: string (nullable = true)  
```
#### The proper schema for --^
```
import org.apache.spark.sql.types._

val trendSchema = new StructType()  
  .add("as_of", StringType)  
  .add("created_at", StringType)  
  .add("locations", ArrayType(new StructType()  
      .add("name", StringType)  
      .add("woeid", LongType)  
    ))  
    .add("trends", ArrayType(new StructType()  
      .add("name", StringType)  
      .add("promoted_content", StringType)  
      .add("query", StringType)  
      .add("tweet_volume", LongType)  
      .add("url", StringType)  
    ))  
  
// read JSON and convert to DF with schema
val df = spark.read.schema(trendSchema).json("input-old")
```
 
## Spark submit - Example, by William Gentry

#!/bin/sh
# s3a is a connector for k8 - https://cwiki.apache.org/confluence/display/HADOOP2/AmazonS3
# s3a documentation at hadoop.apache.org http://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/index.html
# configuration list at cloudera: https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/bk_cloud-data-access/content/s3-config-parameters.html

SPARK_HOME=/home/dannylee/git/spark-3.0.1-bin-hadoop3.2
SCHEME="file"
APPLICATION_JAR_LOCATION="$SCHEME:///home/dannylee/git/k8/pi/target/scala-2.12/sparkpi_2.12-1.jar"
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET="s3://adam-king-848"
S3_INPUT="$S3_BUCKET/data"

set -e

echo "Running job located at $APPLICATION_JAR_LOCATION"

$SPARK_HOME/bin/spark-submit \
    --master k8s://http://localhost:8001 \
    --name s3-example \
    --deploy-mode cluster \
    --class org.apache.spark.examples.SparkPi \
    --packages com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.7.6 \
    --conf spark.kubernetes.file.upload.path="$S3_BUCKET/spark/" \
    --conf spark.hadoop.fs.s3a.access.key="$AWS_ACCESS_KEY_ID" \
    --conf spark.hadoop.fs.s3a.secret.key="$AWS_SECRET_ACCESS_KEY" \
    --conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
    --conf spark.hadoop.fs.s3a.fast.upload=true \
    --conf spark.driver.extraJavaOptions="-Divy.cache.dir=/tmp -Divy.home=/tmp" \
    --conf spark.driver.log.persistToDfs.enabled=true \
    --conf spark.driver.log.dfsDir="$S3_BUCKET/spark-driver-logs/" \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    --conf spark.executor.instances=3 \
    --conf spark.kubernetes.driver.request.cores=1 \
    --conf spark.kubernetes.executor.request.cores=0 \
    --conf spark.hadoop.fs.s3a.path.style.access=true \
    --conf spark.hadoop.fs.s3.buffer.dir="/tmp" \
    --conf spark.hadoop.fs.s3.impl=org.apache.hadoop.fs.s3native.NativeS3FileSystem \
    --conf spark.hadoop.fs.s3.awsAccessKeyId="$AWS_ACCESS_KEY_ID" \
    --conf spark.hadoop.fs.s3.awsSecretAccessKey="$AWS_SECRET_ACCESS_KEY" \
    --conf spark.executor.request.memory=1g \
    --conf spark.driver.request.memory=1g \
    --conf spark.kubernetes.container.image=855430746673.dkr.ecr.us-east-1.amazonaws.com/adam-king-848-example-spark \
    $APPLICATION_JAR_LOCATION $AWS_ACCESS_KEY_ID $AWS_SECRET_ACCESS_KEY $S3_INPUT