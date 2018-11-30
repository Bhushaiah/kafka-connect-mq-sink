# Kafka Connect sink connector for IBM MQ
kafka-connect-mq-sink is a [Kafka Connect](http://kafka.apache.org/documentation.html#connect) sink connector for copying data from Apache Kafka into IBM MQ.

The connector is supplied as source code which you can easily build into a JAR file.

**Note**: A source connector for IBM MQ is also available on [GitHub](https://github.com/ibm-messaging/kafka-connect-mq-source).

## Contents

 - [Building the connector](#building-the-connector)
 - [Running the connector](#running-the-connector)
 - [Data formats](#data-formats)
 - [Security](#security)
 - [Configuration](#configuration)
 - [Troubleshooting](#troubleshooting)
 - [Support](#support)
 - [Issues and contributions](#issues-and-contributions)
 - [License](#license)

## Building the connector
To build the connector, you must have the following installed:
* [git](https://git-scm.com/)
* [Maven 3.0 or later](https://maven.apache.org)
* Java 8 or later

Clone the repository with the following command:
```shell
git clone https://github.com/ibm-messaging/kafka-connect-mq-sink.git
```

Change directory into the `kafka-connect-mq-sink` directory:
```shell
cd kafka-connect-mq-sink
```

Build the connector using Maven:
```shell
mvn clean package
```

Once built, the output is a single JAR `target/kafka-connect-mq-sink-<version>-jar-with-dependencies.jar` which contains all of the required dependencies.


## Running the connector

For step-by-step instructions, see the following guides for running the connector:
 - connecting to Apache Kafka [running locally](UsingMQwithKafkaConnect.md)
 - connecting to an installation of [IBM Event Streams](https://ibm.github.io/event-streams/connecting/mq/sink)

To run the connector, you must have:
* The JAR from building the connector
* A properties file containing the configuration for the connector
* Apache Kafka 2.0.0 or later, either standalone or included as part of an offering such as IBM Event Streams
* IBM MQ v8 or later, or the IBM MQ on Cloud service

The connector can be run in a Kafka Connect worker in either standalone (single process) or distributed mode. It's a good idea to start in standalone mode.

You need two configuration files, one for the configuration that applies to all of the connectors such as the Kafka bootstrap servers, and another for the configuration specific to the MQ sink connector such as the connection information for your queue manager. For the former, the Kafka distribution includes a file called `connect-standalone.properties` that you can use as a starting point. For the latter, you can use `config/mq-sink.properties` in this repository.

The connector connects to MQ using either a client or a bindings connection. For a client connection, you must provide the name of the queue manager, the connection name (one or more host/port pairs) and the channel name. In addition, you can provide a user name and password if the queue manager is configured to require them for client connections. If you look at the supplied `config/mq-sink.properties`, you'll see how to specify the configuration required. For a bindings connection, you must provide provide the name of the queue manager and also run the Kafka Connect worker on the same system as the queue manager.

To run the connector in standalone mode from the directory into which you installed Apache Kafka, you use a command like this:

``` shell
bin/connect-standalone.sh connect-standalone.properties mq-sink.properties
```


## Data formats
Kafka Connect is very flexible but it's important to understand the way that it processes messages to end up with a reliable system. When the connector encounters a message that it cannot process, it stops rather than throwing the message away. Therefore, you need to make sure that the configuration you use can handle the messages the connector will process.

Each message in Kafka Connect is associated with a representation of the message format known as a *schema*. Each Kafka message actually has two parts, key and value, and each part has its own schema. The MQ sink connector does not currently use message keys, but some of the configuration options use the word *Value* because they refer to the Kafka message value.

When the MQ sink connector reads a message from Kafka, it is processed using a *converter* which chooses a schema to represent the message format and creates a Java object containing the message value. The MQ sink connector then converts this internal format into the message it sends to MQ using a *message builder*.

There are three converters built into Apache Kafka. The following table shows which converters to use based on the incoming message encoding.

| Incoming Kafka message | Converter class                                        |
| ---------------------- | ------------------------------------------------------ |
| Any                    | org.apache.kafka.connect.converters.ByteArrayConverter |
| String                 | org.apache.kafka.connect.storage.StringConverter       |
| JSON, may have schema  | org.apache.kafka.connect.json.JsonConverter            |

There are three message builders supplied with the connector, although you can write your own. The basic rule is that if you're using a converter that uses a very simple schema, the default message builder is probably the best choice. If you're using a converter that uses richer schemas to represent complex messages, the JSON message builder is good for generating a JSON representation of the complex data. The following table shows some likely combinations.

| Converter class                                        | Message builder class                                              | Outgoing MQ message    |
| ------------------------------------------------------ | ------------------------------------------------------------------ | ---------------------- |
| org.apache.kafka.connect.converters.ByteArrayConverter | com.ibm.eventstreams.connect.mqsink.builders.DefaultMessageBuilder | **Binary data**        |
| org.apache.kafka.connect.storage.StringConverter       | com.ibm.eventstreams.connect.mqsink.builders.DefaultMessageBuilder | **String data**        |
| org.apache.kafka.connect.json.JsonConverter            | com.ibm.eventstreams.connect.mqsink.builders.JsonMessageBuilder    | **JSON, no schema**    |

When you set *mq.message.body.jms=true*, the MQ messages are generated as JMS messages. This is appropriate if the applications receiving the messages are themselves using JMS.

There's no single configuration that will always be right, but here are some high-level suggestions.

* Message values are treated as byte arrays, pass byte array into MQ message
```
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
```
* Message values are treated as strings, pass string into MQ message
```
value.converter=org.apache.kafka.connect.storage.StringConverter
```

### The gory detail
The messages received from Kafka are processed by a converter which chooses a schema to represent the message and creates a Java object containing the message value. There are three basic converters built into Apache Kafka.

| Converter class                                        | Kafka message encoding | Value schema        | Value class        |
| ------------------------------------------------------ | ---------------------- | ------------------- | ------------------ |
| org.apache.kafka.connect.converters.ByteArrayConverter | Any                    | OPTIONAL_BYTES      | byte[]             |
| org.apache.kafka.connect.storage.StringConverter       | String                 | OPTIONAL_STRING     | java.lang.String   |
| org.apache.kafka.connect.json.JsonConverter            | JSON, may have schema  | Depends on message  | Depends on message |

The MQ sink connector uses a message builder to build the MQ messsages from the schema and value. There are three built-in message builders.

The DefaultMessageBuilder is best when the schema is very simple, such as when the ByteArrayConverter or StringConverter are being used.

| Value schema    | Value class      | Outgoing message format | JMS message type | Outgoing message body           |
| --------------- | ---------------- | ----------------------- | ---------------- | ------------------------------- |
| null            | Any              | MQFMT_STRING            | TextMessage      | Java Object.toString() of value |
| BYTES           | byte[]           | MQFMT_NONE              | BytesMessage     | Byte array                      |
| STRING          | java.lang.String | MQFMT_STRING            | TextMessage      | String                          |
| Everything else | Any              | MQFMT_STRING            | TextMessage      | Java Object.toString() of value |

If you use the JsonConverter with the DefaultMessageBuilder, the output message will not be JSON; it will be a Java string representation of the value instead. That's why there's a JsonMessageBuilder too which behaves like this:

| Value schema    | Value class      | Outgoing message format | JMS message type | Outgoing message body            |
| --------------- | ---------------- | ----------------------- | ---------------- | -------------------------------- |
| Any             | Any              | MQFMT_STRING            | TextMessage      | JSON representation of the value |

To make the differences clear, here are some examples.

| Input message | Converter       | Value schema      | Message builder       | Output message body | Comment                            |
| ------------- | --------------- | ----------------- | --------------------- | ------------------- | ---------------------------------- |
| ABC           | StringConverter | STRING            | DefaultMessageBuilder | ABC                 | OK                                 |
| ABC           | StringConverter | STRING            | JsonMessageBuilder    | "ABC"               | Quotes added to give a JSON string |
| "ABC"         | JsonConverter   | STRING            | DefaultMessageBuilder | ABC                 | Quotes removed, not a JSON string  |
| "ABC"         | JsonConverter   | STRING            | JsonMessageBuilder    | "ABC"               | OK                                 |
| {"A":"B"}     | JsonConverter   | Compound (STRUCT) | DefaultMessageBuilder | STRUCT{A=B}         | Probably not helpful               |
| {"A":"B"}     | JsonConverter   | Compound (STRUCT) | JsonMessageBuilder    | {"A":"B"}           | OK                                 |

Note that the order of JSON structures is not fixed and fields may be reordered.

To handle the situation in which you already have a Kafka converter that you want to use to build the MQ message payload, the ConverterMessageBuilder is the one to use. Then you would end up using two Converters - one to convert the Kafka message to the internal SinkRecord, and the second to convert that into the MQ message. Since the Converter might also have its own configuration options, you can specify them using a prefix of `mq.message.builder.value.converter`. For example, the following configuration gets the ConverterMessageBuilder to work the same as the JsonMessageBuilder.

```
mq.message.builder=com.ibm.eventstreams.connect.mqsink.builders.ConverterMessageBuilder
mq.message.builder.value.converter=org.apache.kafka.connect.json.JsonConverter
mq.message.builder.value.converter.schemas.enable=false
```

### Key support and partitioning
By default, the connector does not use the keys for the Kafka messages it reads. It can be configured to set the JMS correlation ID using the key of the Kafka records. To configure this behavior, set the `mq.message.builder.key.header` configuration value.

| mq.message.builder.key.header | Key schema | Key class | Recommended value for key.converter                    |
| ----------------------------- |----------- | --------- | ------------------------------------------------------ |
| JMSCorrelationID              | STRING     | String    | org.apache.kafka.connect.storage.StringConverter       |
| JMSCorrelationID              | BYTES      | byte[]    | org.apache.kafka.connect.converters.ByteArrayConverter |

In MQ, the correlation ID is a 24-byte array. As a string, the connector represents it using a sequence of 48 hexadecimal characters. The Kafka key will be truncated to fit into this size.


## Security
The connector supports authentication with user name and password and also connections secured with TLS using a server-side certificate and mutual authentication with client-side certificates.

### Setting up TLS using a server-side certificate
To enable use of TLS, set the configuration `mq.ssl.cipher.suite` to the name of the cipher suite which matches the CipherSpec in the SSLCIPH attribute of the MQ server-connection channel. Use the table of supported cipher suites for MQ 9.1 [here](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.1.0/com.ibm.mq.dev.doc/q113220_.htm) as a reference. Note that the names of the CipherSpecs as used in the MQ configuration are not necessarily the same as the cipher suite names that the connector uses. The connector uses the JMS interface so it follows the Java conventions.

You will need to put the public part of the queue manager's certificate in the JSSE truststore used by the Kafka Connect worker that you're using to run the connector. If you need to specify extra arguments to the worker's JVM, you can use the EXTRA_ARGS environment variable.

### Setting up TLS for mutual authentication
You will need to put the public part of the client's certificate in the queue manager's key repository. You will also need to configure the worker's JVM with the location and password for the keystore containing the client's certificate.

### Troubleshooting
For troubleshooting, or to better understand the handshake performed by the IBM MQ Java client application in combination with your specific JSSE provider, you can enable debugging by setting `javax.net.debug=ssl` in the JVM environment.


## Configuration
The configuration options for the Kafka Connect sink connector for IBM MQ are as follows:

| Name                               | Description                                                | Type    | Default       | Valid values                      |
| ---------------------------------- | ---------------------------------------------------------- | ------- | ------------- | --------------------------------- |
| topics or topics.regex             | List of Kafka source topics                                | string  |               | topic1[,topic2,...]               |
| mq.queue.manager                   | The name of the MQ queue manager                           | string  |               | MQ queue manager name             |
| mq.connection.mode                 | The connection mode - bindings or client                   | string  | client        | client, bindings                  |
| mq.connection.name.list            | List of connection names for queue manager                 | string  |               | host(port)[,host(port),...]       |
| mq.channel.name                    | The name of the server-connection channel                  | string  |               | MQ channel name                   |
| mq.queue                           | The name of the target MQ queue                            | string  |               | MQ queue name                     |
| mq.user.name                       | The user name for authenticating with the queue manager    | string  |               | User name                         |
| mq.password                        | The password for authenticating with the queue manager     | string  |               | Password                          |
| mq.ccdt.url                        | The URL for the CCDT file containing MQ connection details | string  |               | URL for obtaining a CCDT file     |
| mq.message.builder                 | The class used to build the MQ message                     | string  |               | Class implementing MessageBuilder |
| mq.message.body.jms                | Whether to generate the message body as a JMS message type | boolean | false         |                                   |
| mq.time.to.live                    | Time-to-live in milliseconds for messages sent to MQ       | long    | 0 (unlimited) | [0,...]                           |
| mq.persistent                      | Send persistent or non-persistent messages to MQ           | boolean | true          |                                   |
| mq.ssl.cipher.suite                | The name of the cipher suite for TLS (SSL) connection      | string  |               | Blank or valid cipher suite       |
| mq.ssl.peer.name                   | The distinguished name pattern of the TLS (SSL) peer       | string  |               | Blank or DN pattern               |
| mq.message.builder.key.header      | The JMS message header to set from the Kafka record key    | string  |               | JMSCorrelationID                  |
| mq.message.builder.value.converter | The class and prefix for message builder's value converter | string  |               | Class implementing Converter      |


### Using a CCDT file
Some of the connection details for MQ can be provided in a [CCDT file](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.1.0/com.ibm.mq.con.doc/q016730_.htm) by setting `mq.ccdt.url` in the Kafka Connect sink connector configuration file. If using a CCDT file the `mq.connection.name.list` and `mq.channel.name` configuration options are not required.

### Externalizing secrets
[KIP 297](https://cwiki.apache.org/confluence/display/KAFKA/KIP-297%3A+Externalizing+Secrets+for+Connect+Configurations) introduced a mechanism to externalize secrets to be used as configuration for Kafka connectors.

#### Example: externalizing secrets with FileConfigProvider

Given a file `mq-secrets.properties` with the contents:
```
secret-key=password
```

Update the worker configuration file to specify the FileConfigProvider which is included by default:

```
# Additional properties for the worker configuration to enable use of ConfigProviders
# multiple comma-separated provider types can be specified here
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
```

Update the connector configuration file to reference `secret-key` in the file:

```
mq.password=${file:mq-secret.properties:secret-key}
```

## Troubleshooting

### Unable to connect to Kafka

You may receive an `org.apache.kafka.common.errors.SslAuthenticationException: SSL handshake failed` error when trying to run the MQ Sink Connector using SSL to connect to your Kafka cluster. In the case that the error is caused by the following exception: `Caused by: java.security.cert.CertificateException: No subject alternative DNS name matching XXXXX found.`, Java may be replacing the IP address of your cluster with the corresponding hostname in your `/etc/hosts` file. For example, to push Docker images to a custom Docker repository, you may add an entry in this file which corresponds to the IP of your repository e.g. `123.456.78.90    mycluster.icp`. To fix this, you can comment out this line in your `/etc/hosts` file.

## Support
A commercially supported version of this connector is available for customers with a support entitlement for [IBM Event Streams](https://www.ibm.com/cloud/event-streams).

## Issues and contributions
For issues relating specifically to this connector, please use the [GitHub issue tracker](https://github.com/ibm-messaging/kafka-connect-mq-sink/issues). If you do submit a Pull Request related to this connector, please indicate in the Pull Request that you accept and agree to be bound by the terms of the [IBM Contributor License Agreement](CLA.md).


## License
Copyright 2017, 2018 IBM Corporation

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    (http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.The project is licensed under the Apache 2 license.
