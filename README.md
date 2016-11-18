# Spark-Jobserver

## Introduction

Spark-Jobserver is a crucial component in the stack. It allows us to:

- Abstract the cluster processing away and present all the distributed logic as a [RESTful API](https://en.wikipedia.org/wiki/Representational_state_transfer).
- Connect to the same data or `SparkContext` from different endpoint, i.e., users. Otherwise every user would have its own `SparkContext` running, eating up precious memory capacity.
- Focus on implementation of the REST API itself, rather than the service.


## Versions

The current version available via a binary distribution is Spark-Jobserver version 0.6.1. The interesting developments, however, are currently work in progress on the master branch and evolve towards 0.6.2. The latter version is thus not yet available via Maven or any other standard form.

The `master` branch of the spark-jobserver project is not backward compatible with the version of Spark we _currently_ use on the cluster at IMEC. We have therefore [created a `spark-1.4.1` branch](https://github.com/data-intuitive/spark-jobserver) that removes the one file that causes the build to fail:

```
git clone https://github.com/data-intuitive/spark-jobserver -b spark-1.4.1
```

The binary distribution in this directory (`Spark-Jobserver-v0.6.2-SNAPSHOT-Spark-v1.4.1.tar.gz`) is based on the a snapshot of the 0.6.2 release. We keep it here for compatibility. The instructions on how to build it [can be found later in this document]().

In combination with Spark v1.6.0, we can use the master as per the current (March 2016) documentation [on the Github page](https://github.com/spark-jobserver/spark-jobserver). The approach to that is explained below, the result can be found as `Spark-Jobserver-v0.6.2-SNAPSHOT-Spark-v1.6.0.tar.gz`.


## Building and deployment

Make sure you have a working Spark cluster running and that the Spark binary distribution can be found under `~/spark`.

Two options exist: Docker and `tar.gz`, both are based on the same `sbt` build process.


### Spark-Jobserver v0.6.2 and Spark v1.6.0

__TODO__: update with tag v0.6.2a

We manually create the binaries. We check out the latest version from Github (based on our fork at <https://github.com/data-intuitive/spark-jobserver>):

```
git clone https://github.com/data-intuitive/spark-jobserver
cd spark-jobserver
```

Set the correct environment variables and compile (this is for Mac):

```
export SPARK_VERSION=1.6.0
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.7.0_45.jdk/Contents/Home"
sbt clean
sbt compile
sbt assembly
```

This can be used as a test. When a new binary package is created (see next section), the project is built again anyway.

The major things to be configured in `<environment>.sh` when cross-compiling on a Mac are:

- Set correct Spark version: `export SPARK_VERSION=1.4.1`
- Set correct Java version: `export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.7.0_45.jdk/Contents/Home"`

In order to see if building the package will likely work or for testing the configuration, you can always run the service manually:

```
sbt
reStart <.../configfile>
```


### Binary Package

Scripts are available for building the necessary deployment bundles. We will create 2 configurations, one to test and one to effectively rollout a binary distribution.

First, copy `config/local.sh.template` to `config/local.sh` and edit as appropriate for you system:

```
DEPLOY_HOSTS="localhost"
APP_USER=toni
#APP_GROUP=spark
# optional SSH Key to login to deploy server
#SSH_KEY=/path/to/keyfile.pem
INSTALL_DIR=~/jobserver
LOG_DIR=~/jobserver/logs
PIDFILE=spark-jobserver.pid
JOBSERVER_MEMORY=1G
SPARK_VERSION=1.6.0
SPARK_HOME=~/spark
SPARK_CONF_DIR=$SPARK_HOME/conf
SPARK_MASTER_IP="localhost"
SCALA_VERSION=2.10.4 # or 2.11.6
```

Secondly, copy this file to a config file for the target system, in our case `awsabiirl1123`:

```
# Environment and deploy file
# For use with bin/server_deploy, bin/server_package etc.
DEPLOY_HOSTS="awsabiirl1123"

APP_USER=tverbeir
#APP_GROUP=ubuntu
# optional SSH Key to login to deploy server
#SSH_KEY=/path/to/keyfile.pem
INSTALL_DIR=~/jobserver
LOG_DIR=~/jobserver/logs
PIDFILE=spark-jobserver.pid
JOBSERVER_MEMORY=1G
SPARK_VERSION=1.6.0
SPARK_HOME=~/spark
SPARK_CONF_DIR=$SPARK_HOME/conf
SPARK_MASTER_IP="awsabiirl1123"
SCALA_VERSION=2.10.4 # or 2.11.6
```

We need one more configuration file: `local.conf` and `awsabiirl1123.conf` are configured like this, with only the Spark master being different:

```
spark {
  master = "spark://awsabiirl1123:7077"

  job-number-cpus = 1

  jobserver {
    port = 8090
    jar-store-rootdir = /tmp/jobserver/jars

    jobdao = spark.jobserver.io.JobFileDAO

    filedao {
      rootdir = /tmp/jobserver/filedao/data
    }
  }

  context-settings {
    num-cpu-cores = 1
    memory-per-node = 512m
  }

  home = "~/spark"
}
```

Now, run `bin/server_package.sh local`. This should yield something like this:

```
...
/tmp/job-server ~/Dropbox/_Janssen/ComPass/Architecture/Spark-Jobserver/spark-jobserver
a kill-process-tree.sh
a log4j-server.properties
a manager_start.sh
a server_start.sh
a server_stop.sh
a setenv.sh
a settings.sh
a spark-job-server.jar
~/Dropbox/_Janssen/ComPass/Architecture/Spark-Jobserver/spark-jobserver
Created distribution at /tmp/job-server/job-server.tar.gz
```

Unpacking `/tmp/job-server/job-server.tar.gz` in a directory is sufficient for the next step.



### Binary Deployment

The `tgz` package built under `/tmp/job-server/job-server.tar.gz` can unpackaged on the master node of the Spark cluster. If all configuration above has been accurate, one should be able to start the service immediately.

If you're sure about the settings and you can log into the remote deployment host, a faster way is to use:

```
> bin/server_deploy.sh local
[success] Total time: 8 s, completed Mar 14, 2016 3:46:54 PM
spark-job-server.jar                                                                               100%   23MB  22.5MB/s   00:00
server_start.sh                                                                                    100% 1899     1.9KB/s   00:00
server_stop.sh                                                                                     100%  607     0.6KB/s   00:00
kill-process-tree.sh                                                                               100% 1626     1.6KB/s   00:00
manager_start.sh                                                                                   100% 1137     1.1KB/s   00:00
setenv.sh                                                                                          100% 1450     1.4KB/s   00:00
/Users/toni/Dropbox/_Janssen/ComPass/Architecture/Spark-Jobserver/spark-jobserver/config/local.conf: No such file or directory
config/shiro.ini: No such file or directory
log4j-server.properties                                                                            100%  710     0.7KB/s   00:00
/Users/toni/Dropbox/_Janssen/ComPass/Architecture/Spark-Jobserver/spark-jobserver/config/local.conf: No such file or directory
local.sh                                                                                           100%  659     0.6KB/s   00:00
```

In this case, we use the `local` configuration to be deployed. This can obviously be done for every configuration you have in mind.

Please note that you can always edit the configuration and startup files on the remote host before actually starting the service as well.


### Docker Deployment

Spark-Jobserver comes with an option to be deployed using Docker. Be careful, though, there are some caveats:

- The binary still version needs to be compiled wrt the version of Spark you use
- host network access is required, unless you use Mesos as a cluster manager (untested!)

We use the project and follow up its development closely: <https://github.com/spark-jobserver/spark-jobserver>.


#### Spark-Jobserver v0.6.0 (and v0.6.1) binary distribution via Docker

The existing Docker build is based on `java:7-jre` ([Docker Hub](https://hub.docker.com/r/library/java/)), which in turn is based on `buildpack-jessie-curl` (see [here](https://github.com/docker-library/buildpack-deps/blob/a0a59c61102e8b079d568db69368fb89421f75f2/jessie/curl/Dockerfile)). This is Debian based. So there is some work to be done to convert it to CentOS.

Deployment via Docker is not really necessary, because Spark-Jobserver supports the creation of a tar-ball with all the necessary ingredients. This could potentially also be used for deployment on Mesos.


#### Building a Docker container via `sbt`

Similarly as building a tarball manually, it's possible to build a docker container manually. The question remains if this is beneficial in this case.

This `Dockerfile` works, but it does not yet support `NamedObject`s:

```
from velvia/spark-jobserver:0.6.0
add local.conf /app/docker.conf
add log4j-server.properties /app/log4j-server.properties
```

If one of the distributions suite your needs, you're fine. Otherwise, the process is similar to compiling a `tar.gz` distribution as mentioned earlier:

```
sbt docker
```


#### Security

The Spark process in the Docker container needs to know the AWS S3 credentials/keys in order to fetch the data.

This is something to think about.




## Running Spark-Jobserver

### Binary Distribution

Starting the service (given that the settings above are correct) is as easy as:

```
bin/server_start.sh
```

Using `jps` you should be able to see the service is started. You can access the service on port 8090 of the server you are running it on.



### Docker

The current version of Spark we use on the IMEC cluster is 1.4.1 (version 2.0 is in progress), which requires spark-jobserver version 0.6.0 in for the binary compatibility to be guaranteed. Please note that we have to provide the `--net=host` argument just like with Spark-Notebook.

```
docker run -d --net=host -p 8090:8090 velvia/spark-jobserver:0.6.0
```

This command starts the jobserver, but it is not aware of the correct configuration, for instance for the Spark Master. We will create a custom docker build when a newer version of Spark or Jobserver is required:

```
from velvia/spark-jobserver:0.6.0
add local.conf /app/docker.conf
add log4j-server.properties /app/log4j-server.properties
```

Now, build the project:

```
docker build -t jobserver .
```

This works, but we also need the AWS keys inside the container in order to access S3:

```
docker run -d --net=host -p 8090:8090 -e AWS_ACCESS_KEY_ID=<access_key> -e AWS_SECRET_ACCESS_KEY=<secret_key> jobserver
```

Mount the cluster spark distribution inside the container:

```
docker run -d --net=host -p 8090:8090 -v /home/toniv/spark-1.4.1:/home/toniv/spark-1.4.1 -e AWS_ACCESS_KEY_ID=... -e AWS_SECRET_ACCESS_KEY=... jobserver-3
```

In the future, we could create our own `Dockerfile` from scratch, which can be done using `sbt`.



## Compile against Spark-Jobserver

In order to compile an API (see LuciusAPI), you need to have the binary distribution available. But as mentioned, no binary distribution is yet available for 0.6.2. We therefore created one ourselves.

The following takes care of that

```
sbt compile
sbt publish-local
```

This publishes the necessary `jar` files in your Ivy repository. In the `sbt` project you need to resolve this version, the following should be included in the `build.sbt` file:

```
libraryDependencies += "spark.jobserver" %% "job-server-api" % "0.6.2-SNAPSHOT" % "provided"

libraryDependencies += "spark.jobserver" %% "job-server-extras" % "0.6.2-SNAPSHOT" % "provided"
```



## FAQ

### Mac

Build using `sbt`, but make sure Java versions on client and server match. On a Mac, the following can used to force the correct Java version:

```
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.7.0_45.jdk/Contents/Home"
```

### Troubleshooting

When starting the server using the `server-start.sh` scripts does not work, try the following for troubleshooting:

1. Add the following to the startup parameters at the bottom of the script: `--verbose`
2. Set the shell variable `$JOBSERVER_FG`: `export JOBSERVER_FG=1`

In one case, on an EC2 micro instance, there was not enough memory to start the jobserver with 512MB. By changing this in `settings.sh`, the problem was quickly resolved.


### Building for Spark v1.4.1

There is a problem if we want to use the latest version of Spark-Jobserver with an earlier version of Spark. Version 0.6.2 can not be compiled against Spark v1.4.1 because the required `mllib` dependencies are not available:

```
[info] Compiling 11 Scala sources to /Users/toni/_Scratch/spark-jobserver/job-server-extras/target/scala-2.10/classes...
[error] /Users/toni/_Scratch/spark-jobserver/job-server-extras/src/spark.jobserver/KMeansExample.scala:5: object clustering is not a member of package org.apache.spark.ml
[error] import org.apache.spark.ml.clustering.KMeans
[error]                            ^
[error] /Users/toni/_Scratch/spark-jobserver/job-server-extras/src/spark.jobserver/KMeansExample.scala:67: not found: type KMeans
[error]       val kmeans = new KMeans()
[error]                        ^
[error] two errors found
[error] (job-server-extras/compile:compileIncremental) Compilation failed
[error] Total time: 65 s, completed Feb 29, 2016 9:58:16 AM
```

Now, this is only for the _extras_ subpackage, so we might get away with it. Please note that this is a transient problem, it will be solved automatically when migrating to later versions of Spark.

For now, let's take a look at what goes wrong precisely. The error is raised on the following imports in [`job-server-extras/src/spark.jobserver/KMeansExample.scala`](https://github.com/spark-jobserver/spark-jobserver/blob/master/job-server-extras/src/spark.jobserver/KMeansExample.scala):

```
import org.apache.spark.ml.clustering.KMeans
import org.apache.spark.ml.feature.{StandardScaler, VectorAssembler}
```

The versions are incompatible. As a temporary workaround, we remove this example from the source tree. This corresponds to [the branch](https://github.com/data-intuitive/spark-jobserver/tree/spark-1.4.1). As an alternative, we added a tag that corresponds to the version. It can be checkout as such:

```
git clone -b spark-1.4.1 https://github.com/data-intuitive/spark-jobserver
```


### Spark-Jobserver config file

We include a default `local.conf` config file which is largely unedited:

```
# Template for a Spark Job Server configuration file
# When deployed these settings are loaded when job server starts
#
# Spark Cluster / Job Server configuration
spark {
  # spark.master will be passed to each job's JobContext
  master = "spark://ly-1-09:7077"
  # master = "mesos://vm28-hulk-pub:5050"
  # master = "yarn-client"

  # Default # of CPUs for jobs to use for Spark standalone cluster
  job-number-cpus = 4

  jobserver {
    port = 8090
    jar-store-rootdir = /tmp/jobserver/jars

    jobdao = spark.jobserver.io.JobFileDAO

    filedao {
      rootdir = /tmp/spark-job-server/filedao/data
    }
  }

  # predefined Spark contexts
  # contexts {
  #   my-low-latency-context {
  #     num-cpu-cores = 1           # Number of cores to allocate.  Required.
  #     memory-per-node = 512m         # Executor memory per node, -Xmx style eg 512m, 1G, etc.
  #   }
  #   # define additional contexts here
  # }

  # universal context configuration.  These settings can be overridden, see README.md
  context-settings {
    num-cpu-cores = 2           # Number of cores to allocate.  Required.
    memory-per-node = 512m         # Executor memory per node, -Xmx style eg 512m, #1G, etc.

    # in case spark distribution should be accessed from HDFS (as opposed to being installed on every mesos slave)
    # spark.executor.uri = "hdfs://namenode:8020/apps/spark/spark.tgz"

    # uris of jars to be loaded into the classpath for this context. Uris is a string list, or a string separated by commas ','
    # dependent-jar-uris = ["file:///some/path/present/in/each/mesos/slave/somepackage.jar"]

    # If you wish to pass any settings directly to the sparkConf as-is, add them here in passthrough,
    # such as hadoop connection settings that don't use the "spark." prefix
    passthrough {
      #es.nodes = "192.1.1.1"
    }
  }

  # This needs to match SPARK_HOME for cluster SparkContexts to be created successfully
  home = "/home/toniv/spark-1.4.1"
}

# Note that you can use this file to define settings not only for job server,
# but for your Spark jobs as well.  Spark job configuration merges with this configuration file as defaults.
```



- - - 



