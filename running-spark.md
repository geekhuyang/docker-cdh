
# Spark on CDH

The standalone CDH container is a great way to develop spark applications
that are destined to run on a CDH cluster in production.

It's great to be sure that hdfs paths are set correctly and that your app runs
against the real cdh jars and respects some of the ideosyncrasies of running
spark on a yarn cluster.  It's tough to not learn these things until the first
time your job runs on the production cluster.

One caveat, the image is fat and requires a fairly hefty development
environment...  but really that's a small price to pay for development to more
closely mimic the real world.

This tested out fine in standalone, as well as yarn client and cluster modes,
but I highly recommend running jobs in `yarn-cluster` mode as early as possible
in the development process.  Examples of that are included below.

Note that for all example commands below, you have a choice of either going
old-school

    docker exec -it mycdh bash
    yarn application -list

or running this whole thing a bit more idiomatically to docker

    docker exec -it mycdh yarn application -list

If you're a fan of tight IDE integration, you might hunt around for (or build)
IDE plugins that talk to yarn.  It's then easy enough to just expose and hit
the yarn ports of the CDH container just like you would a remote cluster.
Perhaps even with the same kind of ssh tunnels you'd use for access to 
remote cluster yarn consoles.  The goal is to blur the lines between development
and production so the only difference (besides performance I hope!) is 
permission to run the jobs.


## hive

Here's what worked

    docker exec -it mycdh beeline
    !connect jdbc:hive2://localhost:10000 username password org.apache.hive.jdbc.HiveDriver
    show databases;
    use default;
    show tables;


or equiv

    beeline -u jdbc:hive2://localhost:10000 -n username -p password -d org.apache.hive.jdbc.HiveDriver

or even 

    echo "show tables;" | docker -H galatea:9999 exec cdh beeline -u jdbc:hive2://localhost:10000 -n username -p password -d org.apache.hive.jdbc.HiveDriver
    (with or without the `docker exec -i`... but it doesn't like the `-t` when you hand it a script on stdin)





## yarn


    docker exec -it mycdh bash
    yarn application -list

    yarn logs -applicationId <applicationId>


## spark-shell

    docker exec -it mycdh bash -l
    spark-shell

or 

    spark-shell --master yarn-client

which can then be compressed with the `-c` arg to `bash`

    docker exec -it mycdh bash -l -c spark-shell --master yarn-client


## cluster


Submit jobs using

    docker exec -it mycdh bash -l -c spark-submit --master yarn-cluster


    docker exec -it cdh spark-submit --master yarn-cluster --class org.apache.spark.examples.SparkPi /usr/lib/spark/examples/lib/spark-examples-1.3.0-cdh5.4.3-hadoop2.6.0-cdh5.4.3.jar 1000

    docker exec -it cdh bash -l
    spark-submit --master yarn-cluster --class org.apache.spark.examples.SparkPi /usr/lib/spark/examples/lib/spark-examples-1.3.0-cdh5.4.3-hadoop2.6.0-cdh5.4.3.jar 1000
 

client mode

    spark-submit --master yarn-client --class org.apache.spark.examples.SparkPi /usr/lib/spark/examples/lib/spark-examples-1.3.0-cdh5.4.3-hadoop2.6.0-cdh5.4.3.jar 10

My workflow is slightly different through ssh tunnels, so the literal commands I actually run with this are

    docker -H galatea:9999 exec -it cdh spark-submit --master yarn-cluster --class org.apache.spark.examples.SparkPi /usr/lib/spark/examples/lib/spark-examples-1.3.0-cdh5.4.3-hadoop2.6.0-cdh5.4.3.jar 1000


# misc

some random stuff

    docker -H galatea:9999 exec -it cdh su - hdfs

use your own jars/jobs...

    -v target:/target

or maybe just

    -v target

not sure.

also

    impala-shell
    hive
    hbase shell


need to set up users for these



## filesystem defaults

recommended:

    docker exec -it mycdh bash -l

which slurps `/etc/spark/conf/spark_env.sh` into the environment of the
interactive login session.

The default filesystem for `sc.textFile("/some/path/in/hdfs")` is hdfs.
If you want something from the local filesystem, prefix it with an explicit
`file` protocol in the path URI like `file:///some/path/to/files`.

Alternatively, if you just 

    docker exec -it mycdh bash

spark will default to the local filesystem and requires explicit `hdfs://`
protocols (maybe addresses too?  I haven't tried yarn this way) to get to hdfs.

