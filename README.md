# Where this repo is coming from
This repo contains an application - which runs the TPCH/TPCDS/TPCx-BB benchmark queries - that you can submit to Apache Spark. You can run any specific query for any number of iterations in one launch (e.g. run Q3 for three times), and the application can generate a report telling you how much time each iteration spent (works for both TPCH/TPCDS); or run any sequence of queris for any number of iterations (e.g. run the sequence {Q1, Q4, Q1-Q22} for 3 times)(only works for TPCH so far). Furthermore, it can help you convert "raw" tables generated for TPCH/TPCDS into Parquet or Orc format. 

This package was orginally developed by NVIDIA in this repo: https://github.com/NVIDIA/spark-rapids. It was a GPU accelerator driver for Apache Spark, and BenchmarkRunner was only a small part of it. After branch-0.3, they dropped the code for BenchmarkRunner (for the reason, see their issue #1846). 

I decided to strip that part of code out from NVIDIA/spark-rapid and do customized modifications to it. This is where this repo comes from. 

An example on how to use this repo/application can be found in this repo: https://github.com/barnes88/rapids-bench. Aaron is my colleague at Purdue AALP lab, both of us are students of Prof. Timothy Rogers. 

# Usage

## Converting TPCH/TPCDS tables to Parquet
❗ This feature is not tested yet. 

Please refer to this repository: https://github.com/barnes88/rapids-bench. These two commands are what you need to run in order to generate Parquet tables for TPCH/TPCDS (in that repository's folder):
```
source setup_env.sh
./gen_parquet_tables.sh -s 1 -c none -m 8G
```

At the heart of it, this is the application that Spark runs:

```
$SPARK_HOME/bin/spark-submit \
    --master local[*] \
    --driver-memory ${mem} \
    --conf spark.sql.parquet.compression.codec="$compress" \
    --jars $SPARK_RAPIDS_PLUGIN_JAR,$CUDF_JAR,$SCALLOP_JAR \
    --class com.nvidia.spark.rapids.tests.tpch.ConvertFiles \
    $SPARK_RAPIDS_PLUGIN_INTEGRATION_TEST_JAR \
    --input $DBGEN_ROOT \
    --output $TBL_ROOT \
    --output-format parquet
```

Here are the CLI configurations for com.nvidia.spark.rapids.tests.tpch.ConvertFiles:
```
class FileConversionConf(arguments: Seq[String]) extends ScallopConf(arguments) {
  val input = opt[String](required = true)
  val output = opt[String](required = true)
  val outputFormat = opt[String](required = true)
  val coalesce = propsLong[Int]("coalesce")
  val repartition = propsLong[Int]("repartition")
  verify()
  BenchUtils.validateCoalesceRepartition(coalesce, repartition)
}
```


## Running TPCH queries
This should also apply to TPCDS/TPCx-BB queries, with a change to the `benchmark` and `input` options.

```
export SCALLOP_JAR=<jar file for package org.rogach.scallop>
export $SPARK_RAPIDS_PLUGIN_INTEGRATION_TEST_JAR=<jar file for this package>

$SPARK_HOME/bin/spark-submit \
... \
<other Spark configs> \
... \
--jars $SCALLOP_JAR \
--class com.nvidia.spark.rapids.tests.BenchmarkRunner \
$SPARK_RAPIDS_PLUGIN_INTEGRATION_TEST_JAR \
--benchmark tpch \
--query q3 \
--input ./tpch-tables/1_none/ \
--input-format parquet \
--summary-file-prefix tpch-cpu \
--iterations 1
```

To run more than one query (e.g. Q3 + Q4) in one launch, simply append the query names one after another, for example: `-query q3 q4`. When there are more than one query names in this option, "`all`" gets expanded to "`q1 q2 q3 q4 ... q22`" (this only works for TPCH now).

The CLI configurations for BenchmarkRunner are here:
```
class BenchmarkConf(arguments: Seq[String]) extends ScallopConf(arguments) {
  val myStrConverter = listArgConverter[String](_.toString)

  val benchmark = opt[String](required = true)
  val input = opt[String](required = true)
  val inputFormat = opt[String](required = true)
  val appendDat = opt[Boolean](required = false, default = Some(false))
  val query = opt[List[String]](required = true)(myStrConverter)
  val iterations = opt[Int](default = Some(3))
  val output = opt[String](required = false)
  val outputFormat = opt[String](required = false)
  val summaryFilePrefix = opt[String](required = false)
  val gcBetweenRuns = opt[Boolean](required = false, default = Some(false))
  val uploadUri = opt[String](required = false)
  verify()
}
```
