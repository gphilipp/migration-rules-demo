

I've exported my environment variables to make it easier to run the commands:
```shell
export SCHEMA_REGISTRY_URL=xxxxx
export SCHEMA_REGISTRY_API_KEY=xxxxx
export SCHEMA_REGISTRY_API_SECRET=xxxxx
export BASIC_AUTH_USER_INFO=$SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET
export BOOTSTRAP_SERVERS=SASL_SSL://xxxxx.xxxxx.aws.confluent.cloud:9092
export KAFKA_API_KEY=xxxxx
export KAFKA_API_SECRET=xxxxx
export SASL_JAAS_CONFIG="org.apache.kafka.common.security.plain.PlainLoginModule required username='$KAFKA_API_KEY' password='$KAFKA_API_SECRET';"
```

## delete subjects (in two steps, first soft, then hard)

```shell
confluent schema-registry schema delete --subject memberships-value --version all --force  
confluent schema-registry schema delete --permanent --subject memberships-value --version all --force 
confluent kafka topic delete memberships --force
```

Setup the environment:
```shell
confluent environment list
confluent environment use env-xq231x
confluent kafka cluster list
confluent kafka cluster use lkc-jv153q
confluent api-key store $KAFKA_API_KEY $KAFKA_API_SECRET
confluent api-key use $KAFKA_API_KEY
``` 

# START HERE

Let's see how Migration Rules work.   

We start by creating the memberships topic with the Confluent CLI.

```shell
confluent kafka topic create memberships
```

We're going to register the first version of the schema, the one with `start_date` and `end_date` at the top level.
```shell 
confluent schema-registry schema create --subject memberships-value --schema membership_v1.avsc --type avro
```

We create the first compatibility group by setting the major_version property to 1.
```shell
jq -n --rawfile schema membership_v1.avsc '{schema: $schema, metadata: { properties: { major_version: 1 }}}' | \
curl $SCHEMA_REGISTRY_URL/subjects/memberships-value/versions \
--basic --user $SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET \
--json @-
# {"id":100036,"version":2,"metadata":{"properties":{"major_version":"1"}},"schema":"{\"type\":\"record\",\"name\":\"Membership\",\"fields\":[{\"name\":\"user_id\",\"type\":\"string\"},{\"name\":\"start_date\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}},{\"name\":\"end_date\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}}]}"}%
```

Let's start a consumer which is pinned to only use major_version 1. 
We do that by passing the property called `use.latest.with.metadata`. The writer schema id is displayed after each message.

```shell
docker run -it --rm confluentinc/cp-schema-registry:7.5.0 \
  kafka-avro-console-consumer \
  --topic memberships \
  --from-beginning \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer-property security.protocol=SASL_SSL \
  --consumer-property sasl.mechanism=PLAIN \
  --consumer-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username='$KAFKA_API_KEY' password='$KAFKA_API_SECRET';" \
  --property schema.registry.url=$SCHEMA_REGISTRY_URL \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$BASIC_AUTH_USER_INFO \
  --property print.schema.ids=true \
  --property use.latest.with.metadata=major_version=1
```

We're also going to start a producer using the `major_version` 1 format.
```shell
confluent kafka topic produce memberships --value-format avro --schema membership_v1.avsc
```

Let's produce a few messages:
```json lines
{ "user_id": "jane_doe", "start_date": 19697, "end_date": 20062 }
{ "user_id": "john_smith","start_date": 18601, "end_date": 19090 }
```

Ok, the consumer receives the messages in the same format as the producer.

If we try to register the new version of the schema which has a breaking change, it's denied because we haven't told the schema registry to use compatibility groups.
```shell
jq -n --rawfile schema membership_v2.avsc '{schema: $schema, metadata: { properties: { major_version: 2 }}}' | \
curl $SCHEMA_REGISTRY_URL/subjects/memberships-value/versions \
--basic --user $SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET \
--json @-
  
  # we get this error:
  # {"error_code":409,"message":"Schema being registered is incompatible with an earlier schema for subject \"memberships-value\", details: [{errorType:'READER_FIELD_MISSING_DEFAULT_VALUE', description:'The field 'validity_period' at path '/fields/1' in the new schema has no default value and is missing in the old schema', additionalInfo:'validity_period'}, {oldSchemaVersion: 1}, {oldSchema: '{\"type\":\"record\",\"name\":\"Membership\",\"fields\":[{\"name\":\"user_id\",\"type\":\"string\"},{\"name\":\"start_date\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}},{\"name\":\"end_date\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}}]}'}, {compatibility: 'BACKWARD'}]"}%
```

Let's tell the Schema Registry to use the `major_version` field to enable Compatibility Groups:
```shell
curl $SCHEMA_REGISTRY_URL/config/memberships-value \
--basic --user $SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET \
-X PUT --json '{ "compatibilityGroup": "major_version" }'
 # {"compatibilityGroup":"major_version"}%
```

If we try to register again, it's now successful and the new schema is assigned to a new group with `major_version` set to 2:
```shell
jq -n --rawfile schema membership_v2.avsc '{schema: $schema, metadata: { properties: { major_version: 2 }}}' | \
curl $SCHEMA_REGISTRY_URL/subjects/memberships-value/versions \
--basic --user $SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET \
--json @-

  # {"id":100023,"version":3,"metadata":{"properties":{"major_version":"2"}},"schema":"{\"type\":\"record\",\"name\":\"Membership\",\"fields\":[{\"name\":\"user_id\",\"type\":\"string\"},{\"name\":\"validity_period\",\"type\":{\"type\":\"record\",\"name\":\"ValidityPeriod\",\"fields\":[{\"name\":\"from\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}},{\"name\":\"to\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}}]}}]}"}%
```

I've written two migration rules in the JSON file: 
an UPGRADE rule to convert the data from `major_version` 1 to `major_version` 2 
and a DOWNGRADE rule for the other way around.
```shell
cat migration_rule_upgrade_and_downgrade.json | jq ".ruleSet.migrationRules"
```

Let's register them.
```shell
curl $SCHEMA_REGISTRY_URL/subjects/memberships-value/versions \
--basic --user $SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET \
--json @migration_rule_upgrade_and_downgrade.json

# {"id":100024,"version":4,"metadata":{"properties":{"major_version":"2"}},"ruleSet":{"migrationRules":[{"name":"move_start_and_end_date_to_validity_period","kind":"TRANSFORM","mode":"UPGRADE","type":"JSONATA","expr":"$merge([$sift($, function($v, $k) {$k != 'start_date' and $k != 'end_date'}), {'validity_period': {'from':start_date,'to':end_date}}])","disabled":false},{"name":"move_validity_period_to_start_date_and_end_date","kind":"TRANSFORM","mode":"DOWNGRADE","type":"JSONATA","expr":"$merge([$sift($, function($v, $k) {$k != 'validity_period'}), {'start_date': validity_period.from, 'end_date': validity_period.to}])","disabled":false}]},"schema":"{\"type\":\"record\",\"name\":\"Membership\",\"fields\":[{\"name\":\"user_id\",\"type\":\"string\"},{\"name\":\"validity_period\",\"type\":{\"type\":\"record\",\"name\":\"ValidityPeriod\",\"fields\":[{\"name\":\"from\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}},{\"name\":\"to\",\"type\":{\"type\":\"int\",\"logicalType\":\"date\"}}]}}]}"}%
```

Let's start a consumer which will use `major_version` 2.
```shell
docker run -it --rm confluentinc/cp-schema-registry:7.5.0 \
  kafka-avro-console-consumer \
  --topic memberships \
  --from-beginning \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer-property security.protocol=SASL_SSL \
  --consumer-property sasl.mechanism=PLAIN \
  --consumer-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username='$KAFKA_API_KEY' password='$KAFKA_API_SECRET';" \
  --property schema.registry.url=$SCHEMA_REGISTRY_URL \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$BASIC_AUTH_USER_INFO \
  --property print.schema.ids=true \
  --property use.latest.with.metadata=major_version=2
```

If we go to the Producer V1 window and produce messages in the old format... 
```json lines

{ "user_id": "morgan_johnson", "start_date": 19366, "end_date": 20655 }
{ "user_id": "casey_williams", "start_date": 18988, "end_date": 19345 }
```

we can see that consumer V2 receives them in the new format. The transformation is done by the UPGRADE migration rule.

Let's start a new producer which use the v2 schema and produce a few messages.
```shell
confluent kafka topic produce memberships --value-format avro --schema membership_v2.avsc
```

```json lines
{ "user_id": "jane_doe", "validity_period": { "from": 19697, "to": 20443}}
{ "user_id": "lara_wilson", "validity_period": { "from": 17498, "to": 17701}}
```

We can see that consumer v1 can receive these v2 messages in the old format. The transformation is done by the DOWNGRADE migration rule.

Both consumers can automatically receive messages in their preferred format. 

You can now evolve your schemas as much as you like without the stress of breaking anything! 