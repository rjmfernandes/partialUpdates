# Partial Updates Into Mongodb

First we start our cluster:

```shell
docker compose up -d
```

You can check for the cluster to have started by:

```shell
docker compose logs -f
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

We will be using the following schema [./testAvro.avsc](testAvro.avsc).

Now we insert in our topic a couple of messages with schema:

```shell
kafka-avro-console-producer --broker-list localhost:29092 --topic test-avro --property schema.registry.url=http://localhost:8081 --property value.schema.file=./testAvro.avsc --property parse.key=true --property key.schema='{"type":"string"}' --property key.separator=,
```

Use the following:

```
"1",{"id":1,"outer_field":{"string":"value-1"},"inner":{"demo.InnerField":{"id":0,"inner_field":{"string":"inner-value-0"}}}}
```

```
"2",{"id":2,"outer_field":{"string":"value-2"},"inner":{"demo.InnerField":{"id":1,"inner_field":{"string":"inner-value-1"}}}}
```

## Mongodb Sink Connector

In anpther terminal now we can configure the sink connector to sink the data to the mongodb database.

```bash
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink-mongodb/config \
    -d '{
          "connector.class"    : "com.mongodb.kafka.connect.MongoSinkConnector",
          "connection.uri"     : "mongodb://root:example@mongo:27017",
          "topics"             : "test-avro",
          "tasks.max"          : "1",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "database"           : "demo",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter",
          "mongodb.delete.on.null.values": "true",
          "delete.on.null.values": "true",
          "document.id.strategy.overwrite.existing": "true",
          "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInKeyStrategy",
          "transforms":"copyIdToKey,replaceFieldInKey",
          "transforms.copyIdToKey.type" : "org.apache.kafka.connect.transforms.ValueToKey",
          "transforms.copyIdToKey.fields" : "id",
          "transforms.replaceFieldInKey.type": "org.apache.kafka.connect.transforms.ReplaceField$Key",
          "transforms.replaceFieldInKey.renames": "id:_id",
          "writemodel.strategy" : "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneDefaultStrategy",
          "collection"          : "test1"}'
```

You can check the entries sinked in [mongodb](http://localhost:18081/db/demo/test1).

## Partial Values

If we try to use partial values and check how the update goes, insert into the avro producer the entriy:

```
"1",{"id":1,"outer_field":null,"inner":{"demo.InnerField":{"id":0,"inner_field":null}}}
```

If we check now the [mongodb](http://localhost:18081/db/demo/test1) we see the whole document got updated with whole entry passed.

### Custom SMT

Let's try to define a different connector removing the null values with a custom SMT created for a [different project](https://github.com/rjmfernandes/kafkaStreamsRefactor/tree/main/kafkaStreamsRefactor5). So we copy our plugin to the plugins folder and restart connect:

```shell
cp kafkaStreamsRefactor5-0.0.1.jar ./plugins
docker compose restart connect
docker compose logs -f connect
```

And now create a new connector sinking to a different collection for comparison:

```bash
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink-mongodb2/config \
    -d '{
          "connector.class"    : "com.mongodb.kafka.connect.MongoSinkConnector",
          "connection.uri"     : "mongodb://root:example@mongo:27017",
          "topics"             : "test-avro",
          "tasks.max"          : "1",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "database"           : "demo",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter",
          "mongodb.delete.on.null.values": "true",
          "delete.on.null.values": "true",
          "document.id.strategy.overwrite.existing": "true",
          "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInKeyStrategy",
          "transforms":"copyIdToKey,replaceFieldInKey,removeNullFields",
          "transforms.copyIdToKey.type" : "org.apache.kafka.connect.transforms.ValueToKey",
          "transforms.copyIdToKey.fields" : "id",
          "transforms.replaceFieldInKey.type": "org.apache.kafka.connect.transforms.ReplaceField$Key",
          "transforms.replaceFieldInKey.renames": "id:_id",
          "transforms.removeNullFields.type": "io.confluent.developer.transforms.RemoveNullFields",
          "writemodel.strategy" : "com.mongodb.kafka.connect.sink.writemodel.strategy.UpdateOneDefaultStrategy",
          "collection"          : "test2"}'
```

No nulls overwriting now at root if we check [mongodb](http://localhost:18081/db/demo/test2).

Basically the outer_field remained unchanged but the inner field got completely replaced and there is the removal of the field inner_field inside inner cause the RemoveNullFields custom SMT removed the entry for the inner map. This is directly related to the way the MongoSinkConnector works which will replace any fields mentioned on the sink record for the UpdateOneDefaultStrategy.

### Flatten SMT

Let's try to use Flatten first before removing nulls. With the hope that the fact fields to be updated will all be mentioned by their path will guarantee that we  only update the mentioned fields even if they are not at the root. With other unmentioned fields inside inner fields kept unchanged also just as the ones at the root.

Let's create a new connector sinking to the new collection:

```bash
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink-mongodb3/config \
    -d '{
          "connector.class"    : "com.mongodb.kafka.connect.MongoSinkConnector",
          "connection.uri"     : "mongodb://root:example@mongo:27017",
          "topics"             : "test-avro",
          "tasks.max"          : "1",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "database"           : "demo",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter",
          "mongodb.delete.on.null.values": "true",
          "delete.on.null.values": "true",
          "document.id.strategy.overwrite.existing": "true",
          "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInKeyStrategy",
          "transforms":"copyIdToKey,replaceFieldInKey,flatten,removeNullFields",
          "transforms.copyIdToKey.type" : "org.apache.kafka.connect.transforms.ValueToKey",
          "transforms.copyIdToKey.fields" : "id",
          "transforms.replaceFieldInKey.type": "org.apache.kafka.connect.transforms.ReplaceField$Key",
          "transforms.replaceFieldInKey.renames": "id:_id",
          "transforms.flatten.type": "org.apache.kafka.connect.transforms.Flatten$Value",
          "transforms.flatten.delimiter": ".",
          "transforms.removeNullFields.type": "io.confluent.developer.transforms.RemoveNullFields",
          "writemodel.strategy" : "com.mongodb.kafka.connect.sink.writemodel.strategy.UpdateOneDefaultStrategy",
          "collection"          : "test3"}'
```

If we check our values at [mongodb](http://localhost:18081/db/demo/test3) all good. No nulls overriding at all. So now if we try with the following value in the producer:

```
"1",{"id":1,"outer_field":null,"inner":{"demo.InnerField":{"id":3,"inner_field":null}}}
```

It works! So flattening first and after executing the custom RemoveNullFields achieves the desired behaviour.

## Clean up

```shell
docker compose down -v
```

If you also want to remove plugins:

```shell
rm -fr plugins
```