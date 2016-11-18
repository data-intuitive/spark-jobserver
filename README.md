This is a (minor) fork of the [Spark Jobserver](https://github.com/spark-jobserver/spark-jobserver) project. For the original [readme, please see here](https://github.com/spark-jobserver/spark-jobserver/blob/master/README.md).

This is the second forked version we created. Version numbers are taken from the Spark Jobserver original snapshot version `0.x.y-SNAPSHOT` and translated to `.x.y`__`a`__:

- <https://github.com/data-intuitive/spark-jobserver/tree/v0.6.2a>
- <https://github.com/data-intuitive/spark-jobserver/tree/v0.7.0a>

# Introduction

I am a big fan of the [Spark Jobserver](https://github.com/spark-jobserver/spark-jobserver) project. I currently have two projects ([one](https://github.com/data-intuitive/LuciusAPI) and [two](https://github.com/tverbeiren/tetraitesapi)) that use test or synthetic data for frontend development in combination with Spark Jobserver. The synthetic data has an identical format to the real data, but is smaller in size and is, of course, synthetic.

Although Spark Jobserver is a very powerful tool, it is made to connect to a running Spark cluster. And who has a running Spark cluster running around?

So, here comes [Docker](https://www.docker.com/)! And luckily Spark Jobserver comes with functionality to create a Docker container from _source_. And you should take the _source_ aspect literally. I almost burned myself touching my laptop in the process!


# Preparation

You will need a few things to get started.

## CLI and other Tools

You will need at a minimum [`curl`](xxx) or similar. What follows is a description for a general UNIX system (which includes a Mac).


## Docker

You will need a running Docker service and the corresponding tools to manage containers. More information can be found on the relevant [Docker pages](https://www.docker.com/products/overview).


## Location / Directory

You will also need a directory to put the input data (as most data applications require data) and the code for the API. Let us say, for the sake of the argument, that we use `/tmp/api`:

```
mkdir /tmp/api
```

In it, you store the data (ideally in another subdirectory `data` and the assembly jar with the application logic compiled for Spark Jobserver.

```
mkdir /tmp/data
```

## Starting the Docker container

The following command should be sufficient to run the Spark Jobserver as a container:

```
docker run -d -p 8090:8090 -v /tmp/api/data:/app/data tverbeiren/jobserver
```

The image will be downloaded from [Docker Hub](https://hub.docker.com/r/tverbeiren/jobserver/) and started as a daemon. Please note that we map port 8090, which is the default.

In order to test your setup, point your browser to the following url after a minute or so:

```
http://localhost:8090
```

Alternatively, you can issue the following command from the CLI:

```
curl localhost:8090/jobs
```

The result of the latter should be an empty array: `[]%`.

__Remark__: At the time of writing, Spark Jobserver is configured to use 4G of RAM. Make sure your Docker preferences reflect that. This is especially so for people using Docker on a Mac.



# Example based on tests in Spark Jobserver

I have prepared the examples bundled with Spark-Jobserver as a [download here](https://bintray.com/tverbeiren/maven/download_file?file_path=spark%2Fjobserver%2Fjob-server-tests_2.11%2F0.7.0a%2Fjob-server-tests_2.11-0.7.0a.jar). Store this file under `/tmp/api`.

In order to start the example, we need to upload the application to the Spark Jobserver API:

```bash
curl --data-binary @/tmp/api/job-server-tests_2.11-0.7.0a.jar localhost:8090/jars/myApp
```

The response should be

```json
{
  "status": "SUCCESS",
  "result": "Jar uploaded"
}%
```

Now, run the following:

```bash
curl -d '{input.string = "a few words to count takes us a long way with a few possible mistakes"}' \
	'localhost:8090/jobs?appName=myApp&classPath=spark.jobserver.WordCountExample&sync=true'
```

The result should read:

```json
{
  "jobId": "24aaa3e9-761e-489c-81ac-9a469ae9b533",
  "result": {
    "d": 1,
    "e": 1,
    "a": 4,
    "b": 1
  }
}%
```

Please note that the syntax for the POST config parameters resembles JSON, but not quite. In this example, the POST input parameters can be put inline. In more advanced situations, however, this will not be practical anymore. In order to illustrate that, create a file `/tmp/api/wordcount.conf` with the following contents:

```json
{
    "input" : {
        "string" : "a few words to count takes us a long way with a few possible mistakes"
        }
}
```

This, in contrast to the earlier inline version, is valid JSON. In order to use `curl` with this file, issue the following command:

```bash
curl --data-binary @/tmp/api/wordcount.conf \
	'localhost:8090/jobs?appName=myApp&classPath=spark.jobserver.WordCountExample&sync=true'
```

More information concerning the config syntax to use can be found [here](https://github.com/typesafehub/config) with some [specific examples](https://github.com/typesafehub/config#examples-of-hocon).

These examples start a new Spark Context with every run, which is inefficient but also does not allow for keeping data cached across runs. Spark Jobserver allows for the creation of a context (`myContext`) by means of an empty POST request:

```
curl -d '' 'localhost:8090/contexts/myContext?num-cpu-cores=1&memory-per-node=512'
```

The call used earlier to execute the word count example now becomes:

```bash
curl --data-binary @/tmp/api/wordcount.conf \
	'localhost:8090/jobs?appName=myApp&context=myContext&classPath=spark.jobserver.WordCountExample&sync=true'
```

This concludes the example shipped with Spark Jobserver.




