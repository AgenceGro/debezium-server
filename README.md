# Debezium Server for NTR: Custom Implementation

This fork contains custom logic to match NTR's specific requirements. Nothing too fancy or useful at large, but you're welcome to use it.

Namely, the custom logic:

* Only "INSERT" events should be published to Pub/Sub
* The topic name should come from the configuration file (custom `topicName` configuration point)
* Published messages should be in binary format and adhere to a Protobuf message described in `proto/event.proto`
* Published messages should only contain the contents *payload*; not the schema and metadata from Debezium

## Building

The following instructions worked for me in a dev container.

You first need to compile the core Debezium project (will take several minutes [~7 @ 8 CPU cores]):

````bash
cd /workspaces
git clone https://github.com/debezium/debezium.git

# cd debezium
# mvn clean install -DskipITs -DskipTests
````

Then compile the Protobuf file:

````bash
cd /workspaces/debezium-server
sudo apt install -y protobuf-compiler
protoc -I=proto --java_out=debezium-server-pubsub/src/main/java proto/event.proto
````

You can then compile Debezium Server (~2 minutes):

````bash
cd /workspaces/debezium-server
mvn clean install -DskipITs -DskipTests
````

To create a distribution:

````bash
mvn clean package -DskipITs -DskipTests -Passembly
````

Built packages can be found in `debezium-server-dist/target`.

Want to run the project? I haven't been able to do that yet. I don't know anything about Maven and whatever "Quarkus" is.
Good luck!
Also, there is no Dockerfile here so I wouldn't even know how to build the Docker image.

------

# Debezium Server

Debezium Server is a standalone Java application built on Quarkus framework.
The application itself contains the `core` module and a set of modules responsible for communication with different target systems.
## Building and Packaging

The following software is required to work with the Debezium codebase and build it locally:

* [Git](https://git-scm.com) 2.2.1 or later
* JDK 11 or later, e.g. [OpenJDK](http://openjdk.java.net/projects/jdk/)
* [Docker Engine](https://docs.docker.com/engine/install/) or [Docker Desktop](https://docs.docker.com/desktop/) 1.9 or later
* [Apache Maven](https://maven.apache.org/index.html) 3.8.4 or later  
  (or invoke the wrapper with `./mvnw` for Maven commands)

See the links above for installation instructions on your platform. You can verify the versions are installed and running:

    $ git --version
    $ javac -version
    $ mvn -version

### Building the code

Debezium Server depends on core Debezium.  You'll need to install the most recent snapshot locally. First obtain the code by cloning the Git repository:

    $ git clone https://github.com/debezium/debezium.git
    $ cd debezium

Then build the code using Maven:

    $ mvn clean install -DskipITs -DskipTests

Then, you can build Debezium Server:

    $ git clone https://github.com/debezium/debezium-server.git
    $ cd debezium-server
    $ mvn clean install -DskipITs -DskipTests

### Creating a Distribution

Debezium Server is normally run by downloading the distribution tar.gz or zip file.  You can generate this file by:

    $ mvn clean package -DskipITs -DskipTests -Passembly

The archives can be found under `debezium-server-dist/target`.

### Building just the artifacts, without running tests, CheckStyle, etc.

You can skip all non-essential plug-ins (tests, integration tests, CheckStyle, formatter, API compatibility check, etc.) using the "quick" build profile:

    $ mvn clean verify -Dquick

This provides the fastest way for solely producing the output artifacts, without running any of the QA related Maven plug-ins.
This comes in handy for producing connector JARs and/or archives as quickly as possible, e.g. for manual testing in Kafka Connect

## Integration Tests

The per-module integration tests depend on the availability of the external services.
It is thus recommended to execute integration tests per-module and set-up necessary pre-requisities beforehand.

Note: running these tests against external infrastructure may incur cost with your cloud provider.
We're not going to pay your AWS/GCP/Azure bill.

### Amazon Kinesis

* Execute `aws configure` as described in AWS CLI [getting started](https://github.com/aws/aws-cli#getting-started) guide and setup the account.
* Create Kinesis stream `aws kinesis create-stream --stream-name testc.inventory.customers --shard-count 1`
* Build the module and execute the tests `mvn clean install -DskipITs=false -am -pl debezium-server-kinesis`
* Remove the stream `aws kinesis delete-stream --stream-name testc.inventory.customers`

### Google Cloud Pub/Sub

* Login into your Google Cloud account using `gcloud auth application-default login` as described in the [documentation](https://cloud.google.com/sdk/gcloud/reference/auth/application-default).
* Build the module and execute the tests `mvn clean install -DskipITs=false -am -pl debezium-server-pubsub`

### Azure Event Hubs

Login into your Azure account and create a resource group, e.g. on the CLI:

```shell
az login
az group create --name eventhubstest --location westeurope
```

### Create an Event Hubs namespace

Create an [Event Hubs namespace](https://docs.microsoft.com/azure/event-hubs/event-hubs-features#namespace). Check the documentation for options on how do this using the [Azure Portal](https://docs.microsoft.com/azure/event-hubs/event-hubs-create#create-an-event-hubs-namespace), [Azure CLI](https://docs.microsoft.com/azure/event-hubs/event-hubs-quickstart-cli#create-an-event-hubs-namespace) etc., e.g. on the CLI:

```shell
az eventhubs namespace create --name debezium-test --resource-group eventhubstest -l westeurope
```

### Create an Event Hub

Create an Event Hub (equivalent to a topic) with 5 partitions. Check the documentation for options on how do this using the [Azure Portal](https://docs.microsoft.com/azure/event-hubs/event-hubs-create#create-an-event-hub), [Azure CLI](https://docs.microsoft.com/azure/event-hubs/event-hubs-quickstart-cli#create-an-event-hub) etc. , e.g. on the CLI:

```shell
`az eventhubs eventhub create` --name debezium-test-hub --resource-group eventhubstest --namespace-name debezium-test --partition-count 5
```

### Build the module

[Get the Connection string](https://docs.microsoft.com/azure/event-hubs/event-hubs-get-connection-string) required to communicate with Event Hubs. The format is: `Endpoint=sb://<NAMESPACE>/;SharedAccessKeyName=<ACCESS_KEY_NAME>;SharedAccessKey=<ACCESS_KEY_VALUE>`.
E.g. on the CLI:

```shell
az eventhubs namespace authorization-rule keys list --resource-group eventhubstest --namespace-name debezium-test --name RootManageSharedAccessKey
```

Set environment variables required for tests:

```shell
export EVENTHUBS_CONNECTION_STRING=<Event Hubs connection string>
export EVENTHUBS_NAME=<name of the Event hub created in previous step>
```

Execute the tests:

```shell
mvn clean install -DskipITs=false -Deventhubs.connection.string=$EVENTHUBS_CONNECTION_STRING -Deventhubs.hub.name=$EVENTHUBS_NAME -am -pl :debezium-server-eventhubs
```

### Examine Events in the Event Hub

E.g. using kafkacat. Create _kafkacat.conf_:

```shell
metadata.broker.list=debezium-test.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanisms=PLAIN
sasl.username=$ConnectionString
sasl.password=Endpoint=sb://debezium-test.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<access key>
```

Start consuming events:

export KAFKACAT_CONFIG=<path to kafkacat.conf>
kafkacat -b debezium-test.servicebus.windows.net:9093 -t debezium-test-hub

### Clean up

Delete the Event Hubs namespace and log out, e.g. on the CLI:

```shell
az group delete -n eventhubstest
az logout
```
