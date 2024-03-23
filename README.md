# Flatten SMT Research

First we start our cluster:

```shell
docker compose up -d
```

## Mongodb

You can access the mongo express interface on http://localhost:18081 with user/password admin/pass. Create database named demo.

## Connect

You can check the connector plugins available by executing:

```bash
curl localhost:8083/connector-plugins | jq
```

As you see we only have source connectors:

```text
[
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "7.6.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "7.6.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "7.6.0-ce"
  }
]
```

Let's install mongodb/kafka-connect-mongodb connector plugin for sink.

For that we will open a shell into our connect container:

```bash
docker compose exec -it connect bash
```

Once inside the container we can install a new connector from confluent-hub:

```bash
confluent-hub install mongodb/kafka-connect-mongodb:latest
```

(Choose option 2 and after say yes to everything when prompted.)

Do the same for:

```bash
confluent-hub install confluentinc/connect-transforms:latest
```

Now we need to restart our connect:

```bash
docker compose restart connect
```

Now if we list our plugins again we should see two new ones corresponding to the Mongo connector.

## Create and populate our topic

For creating our topic we execute:

```shell
kafka-topics --bootstrap-server localhost:29092 --topic testAvro --create --partitions 1 --replication-factor 1
```

We will be using the following schema:

```
[
  {
    "type": "record",
    "name": "InnerField",
    "namespace": "demo",
    "fields": [
      {
        "name": "id",
        "type": "int"
      },
      {
        "name": "innerField",
        "type": "string"
      }
    ]
  },
  {
    "type": "record",
    "name": "testAvro",
    "namespace": "demo"
    "fields": [
      {
        "name": "id",
        "type": "int"
      },
      {
        "name": "outerField",
        "type": "string"
      },
      {
        "name": "innerField",
        "type": "InnerField"
      }
    ]
  }
]
```

Let's register our schemas:

```shell
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "[\n  {\n    \"type\": \"record\",\n    \"name\": \"InnerField\",\n    \"namespace\": \"demo\",\n    \"fields\": [\n      {\n        \"name\": \"id\",\n        \"type\": \"int\"\n      },\n      {\n        \"name\": \"innerField\",\n        \"type\": \"string\"\n      }\n    ]\n  },\n  {\n    \"type\": \"record\",\n    \"name\": \"testAvro\",\n    \"namespace\": \"demo\"\n    \"fields\": [\n      {\n        \"name\": \"id\",\n        \"type\": \"int\"\n      },\n      {\n        \"name\": \"outerField\",\n        \"type\": \"string\"\n      },\n      {\n        \"name\": \"innerField\",\n        \"type\": \"InnerField\"\n      }\n    ]\n  }\n]"}' \
  http://localhost:8081/subjects/testavro-value/versions
```




Now we insert in our topic a couple of messages with schema:

```shell
kafka-avro-console-producer --broker-list localhost:29092 --topic testAvro --property parse.key=true --property key.separator=, --property schema.registry.url=http://localhost:8081 --property value.schema='{'type':'record','name':'myrecord','fields':[{'name':'f1','type':'string'}]}'
```

## Clean up

```shell
docker compose down -v
```