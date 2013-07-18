# CDK End-to-End Demo

This module provides an example of logging application events from a webapp to Hadoop
via Flume (using log4j as the logging API), extracting session data from the events using
Crunch, running the Crunch job periodically using Oozie, and analyzing the
session data with SQL using Hive.

## Pre-requisites

Before trying this example, you need to have installed the CDK event serializer module in
Flume (this is explained in the `logging` example).

Next, start a Flume agent on the QuickStart VM. You can do this via Cloudera Manager by
selecting "View and Edit" under the Flume service Configuration tab, then clicking on the
"Agent (Default)" category, and pasting the contents of the `flume.properties` file in
this project into the text area for the "Configuration File" property.

If you are running this example from you machine and not from a QuickStart VM login,
then make sure you change the value of the `proxyUser` setting in the agent
configuration to the user that you are logged in as. Save changes,
then start the Flume agent.

For Oozie you need to have Oozie's sharelib installed (which is taken care of already in
the QuickStart VM) and the Oozie service must be running - so start it using Cloudera
Manager.

Finally add the HCatalog Core JAR to the Hive Oozie sharelib,
by logging in to the VM and running:

```bash
sudo -u oozie hadoop fs -put \
  /usr/lib/hcatalog/share/hcatalog/hcatalog-core-0.5.0-cdh4.3.0.jar \
  /user/oozie/share/lib/hive
```

## Building

To build the project, type

```bash
mvn package
```

This creates the following artifacts:

* a JAR file containing the compiled Avro specific schema `session.avsc` (in `demo-core`)
* a JAR file containing classes for creating datasets (in `demo-admin`)
* a WAR file for the webapp that logs application events (in `demo-webapp`)
* a JAR file for running the Crunch job to transform events into sessions (in
`demo-crunch`)
* an Oozie application for running the Crunch job on a periodic basis (in `demo-oozie`)

## Running

### Create the datasets

First we need to create the datasets: one called `events` for the raw events,
and `sessions` for the derived sessions.

We store the raw events metadata in HDFS so Flume can find the schema (it would be nice
if we could store it using HCatalog, so we may lift this restriction in the future).
The sessions dataset metadata is stored using HCatalog, which will allow us to query it
via Hive.

```bash
java -cp demo-admin/target/*:demo-admin/target/jars/* com.cloudera.cdk.examples.demo.CreateStandardEventDataset
java -cp demo-admin/target/*:demo-admin/target/jars/* com.cloudera.cdk.examples.demo.CreateSessionDataset
```

You can check that the data directories were created, using Hue (login as `cloudera` if
 you are logged in to the VM, or as your host login if you are running from your
 machine): [`/tmp/data/events`](http://localhost:8888/filebrowser/#/tmp/data/events),
 [`/tmp/data/sessions`](http://localhost:8888/filebrowser/#/tmp/data/sessions).

### Create events

Next we can run the webapp. It can be used in a Java EE 6 servlet
container; for this example we'll start an embedded Tomcat instance using Maven:

```bash
cd demo-webapp
mvn tomcat7:run
```

Navigate to [http://localhost:8080/demo-webapp/](http://localhost:8080/demo-webapp/),
which presents you with a very simple web page for sending messages.

The message events are sent to the Flume agent
over IPC, and the agent writes the events to the HDFS file sink.

Rather than creating lots of events manually, it's easier to simulate two users with
a script as follows:

```bash
./bin/simulate-activity.sh 1 10 > /dev/null &
./bin/simulate-activity.sh 2 10 > /dev/null &
```

### Generate the derived sessions

Wait about 30 seconds for Flume to flush the events to the
[filesystem](http://localhost:8888/filebrowser/#/tmp/data/events),
then run the Crunch job to generate derived session data from the events:

```bash
(cd demo-crunch; mvn cdk:run-tool)
```

The `cdk:run-tool` Maven goal executes the `run` method of the `Tool`,
in this case `CreateSessions`, which launches a Crunch job on the cluster.

The `Tool` class to run, as well as the cluster settings, are found from the configuration
of the `cdk-maven-plugin`.

When it's complete you should see a file in [`/tmp/data/sessions`]
(http://localhost:8888/filebrowser/#/tmp/data/sessions).

### Run session analysis

The `sessions` dataset is now populated with data, which you can analyze using SQL
using the [Hive UI (Beeswax)](http://localhost:8888/beeswax/) in Hue. Try these queries:

```
DESCRIBE sessions
```

```
SELECT * FROM sessions
```

```
SELECT AVG(duration) FROM sessions
```

### Use Oozie to create derived sessions periodically

Oozie is a workflow management system for running jobs on a Hadoop cluster. Rather than
launching jobs from the developer console, Oozie applications are deployed to the
cluster then launched from there.

A quick note on terminology: an Oozie _application_ is all the packaged code and
configuration. There are three types of Oozie application: workflow applications that
describe a workflow of actions; coordinator applications that run workflows based on
time and data triggers; and bundle applications that run batches of coordinator
applications. An Oozie _job_ is the running instantiation of an Oozie _application_.

The `cdk-maven-plugin` provides Maven goals for packaging, deploying,
and running Oozie applications.

Oozie applications are stored in HDFS to allow Oozie to access and run them, so
the first thing we do is deploy the Oozie application to HDFS.

```bash
cd demo-oozie
mvn cdk:deploy-app
```

The filesystem to deploy to is specified by the `deployFileSystem` setting for the
`cdk-maven-plugin`. By default, Oozie applications are stored in the
`/user/<user>/apps` directory on
HDFS. You can navigate to this location using the
[web interface](http://localhost:8888/filebrowser/#/user) to see if the
application has been successfully deployed.

Before running an Oozie coordinator application, let's run a one-off workflow. The
Oozie server to use is specified by `oozieUrl` in the plugin configuration.

```bash
mvn cdk:run-job -Dcdk.applicationType=workflow
```

Monitor the workflow job using the [Oozie application in Hue](http://localhost:8888/oozie/list_oozie_workflows/).
You can click through to see the underlying MapReduce jobs (just one in this case)
that are run by Crunch.

Next, let's run a coordinator application that runs the workflow application once
every minute. Build the coordinator version of the app:

```bash
mvn package cdk:deploy-app -Dcdk.applicationType=coordinator
```

We need to create events continuously, which we do by running the
user simulation script again. This time we don't specify a limit on the number
of events to create, so it runs indefinitely.

```bash
./bin/simulate-activity.sh 1
```

Now we can run the Oozie coordinator application.

```bash
mvn cdk:run-job -Dcdk.applicationType=coordinator -Dstart="$(date -u +"%Y-%m-%dT%H:%MZ")"
```

Monitor the coordinator and resulting workflow jobs using the [Oozie application in Hue](http://localhost:8888/oozie/list_oozie_coordinators).

After a minute or two when a workflow job has completed, you should see new files appear
in the `sessions` dataset, in
[`/tmp/data/sessions`](http://localhost:8888/filebrowser#/tmp/data/sessions).
When you see new files appear, then try running the session analysis from above.

When you have finished, stop the user simulation script by killing the process
(with Ctrl-C). Kill the Oozie coordinator job through the Hue web interface.