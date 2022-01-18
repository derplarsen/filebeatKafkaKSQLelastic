
# Overview

This repository is to show how to output filebeat to kafka, instantiate a schema in Schema Registry, flatten the schema in ksqlDB and save into another kafka topic as AVRO format, then output to Elasticsearch.

### Run Kafka / Connect / ksqlDB / Elasticsearch 

The easiest way to do this is run it via kafka-docker-playground.

 1. Make sure you don't have any Confluent Platform components currently running.
 2. Clone this: https://github.com/vdesabou/kafka-docker-playground
 3. Change directory into **kafka-docker-playground/connect/connect-elasticsearch-sink**
 4. `chmod +x elasticsearch.sh && ./elasticsearch.sh`
 
### Set up Filebeat

1. Install filebeat and modify filebeat.yml in the appropriate location.
2. In "*filebeat.inputs:*" section, disable **type: log**
3. In "*filebeat.inputs:*" section, enable **type: filestream** and set it to process whatever log file you want, I used `/var/log/system.log`
4. In outputs, comment out Elasticsearch and any other output
5. add the following output:

<pre>  output.kafka:
          hosts: ["broker:9092"]
          topic: filebeat_in
          partition.round_robin:
                reachable_only: false
          required_acks: 1
          compression: gzip
          max_message_bytes: 1000000</pre>

(Note **broker:9092**, you likely will need to add a host record to 127.0.0.1 for `broker`)

6. Run `filebeat -e`
7. Now if you open the Confluent Control Center at `localhost:9021` you should see a new topic that got automatically created called **filebeat_in** (per the topic property in the filebeat configuration), take a look at the data by setting offset to 0 and inspect the data in plain JSON using the button with 4 solid lines on the top right of the data output.

### How do I sink this data to Elastic?
This docker playground instance does send sample data into a topic which gets sinked down into elasticsearch using the Elasticsearch Sink Connector, but we want to sink down to a *new* index with *new* data.

Now that you've got the json data in the filebeat_in topic, we could try to add that topic in the connector configuration and restart the connector, but the problem is that by default it's going to fail because it doesn't have a known schema. It is possible work around this with an additional configuration made, but if we know the schema we can do stream processing to it! 

So first, let's set up a schema in the Confluent Schema Registry and use ksqlDB!

### Create a STREAM

In order to instantiate a schema, we can use ksqlDB to do this. The first thing we need to do is initialize with a new stream. Go back to Confluent Control Center at localhost:9021 and go to ksqlDB then click on the ksqlDB instance. You will be presented with an editor.

FIRST, click on the drop down for Latest, and choose Earliest. We want to be able to see any data, including what's already been sent to topics.

Below is a sample payload from one of my log messages:
<pre>
{
  "@timestamp": "2022-01-18T22:44:22.975Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "_doc",
    "version": "7.15.1"
  },
  "log": {
    "offset": 684257,
    "file": {
      "path": "/var/log/system.log"
    }
  },
  "message": "Jan 18 17:44:21 localhost com.apple.xpc.launchd[1] (com.apple.mdworker.shared.01000000-0700-0000-0000-000000000000[85810]): Service exited due to SIGKILL | sent by mds[155]",
  "input": {
    "type": "filestream"
  },
  "ecs": {
    "version": "1.11.0"
  },
  "host": {
    "ip": [
      "fe80::aede:48ff:fe00:1122",
      "fe80::1c3a:bee8:78cd:e3d7",
      "192.168.86.239"
    ],
    "mac": [
      "3a:f9:d3:d6:68:94",
      "ac:de:48:00:11:23",
      "38:f9:d3:d3:68:93"
    ],
    "name": "localhost",
    "hostname": "localhost",
    "architecture": "x86_64",
    "os": {
      "platform": "darwin",
      "version": "10.15.7",
      "family": "darwin",
      "name": "Mac OS X",
      "kernel": "19.6.0",
      "build": "19H1519",
      "type": "macos"
    },
    "id": "DDDBD54A-B228-55F1-9CBE-52ED95649F8E"
  },
  "agent": {
    "ephemeral_id": "87217691-0ab2-479d-b208-79d7c66a7b33",
    "id": "4b4b1c48-80ff-4d18-8cba-177290156e18",
    "name": "localhost",
    "type": "filebeat",
    "version": "7.15.1",
    "hostname": "localhost"
  }
}
</pre>
So after analyzing the schema, I know all the subsequent schema will be the same, I can create this statement, which initializes a STREAM, declaring the schema for the payload we're getting from filebeat. 

(Note, some filebeat versions are different, I'm using 7.15 at the time of this writing, so the **schema definition can change**, double check use yours!). 

<pre>
CREATE STREAM FILEBEAT_STREAM_INIT (
	"@timestamp" VARCHAR,
	"@metadata" STRUCT<
		beat VARCHAR,
		type VARCHAR,
		version VARCHAR
	>,
	input STRUCT<
    type VARCHAR
  >,
  host STRUCT<
    name VARCHAR,
    mac ARRAY<VARCHAR>,
    hostname VARCHAR,
    architecture VARCHAR,
    os STRUCT<
      build VARCHAR,
      type VARCHAR,
      platform VARCHAR,
      version VARCHAR,
      family VARCHAR,
      name VARCHAR,
      kernel VARCHAR
    >,
    id VARCHAR,
    ip ARRAY<VARCHAR>
  >,
  agent STRUCT<
    version VARCHAR,
    hostname VARCHAR,
    ephemeral_id VARCHAR,
    id VARCHAR,
    name VARCHAR,
    type VARCHAR
  >,
  ecs STRUCT<
    version VARCHAR
  >,
  log STRUCT<
    offset BIGINT,
    file STRUCT<
      path VARCHAR
    >
  >,
  message VARCHAR
)
WITH (
    KAFKA_TOPIC = 'filebeat_in',
    VALUE_FORMAT = 'JSON'
  );
</pre>

Then hit the **Run Query** button. At this point, you should be able to see a new stream created under "STREAMS" in your ksqlDB editor UI. If you click on it and click Query Stream you should see the following appear: `select * from FILEBEAT_STREAM_INIT EMIT CHANGES;` (again, make sure "Earliest" is chosen, vs Latest) next to the auto.offset.reset property). You should see data coming in!

Now let's create a CSAS (Create Stream as Select) statement to instantiate a new stream with flattened output in AVRO format, that we'll then use to sink down to Elasticsearch:

<pre>
CREATE STREAM FILEBEAT_STREAM_OUTPUT
WITH (
  value_format='AVRO'
)
AS
select
    MESSAGE,
	  INPUT->TYPE as FILEBEAT_INPUT_TYPE,
    HOST->NAME as Host_name,
    HOST->MAC as MacAddr,
    HOST->HOSTNAME as Hostname,
    HOST->Architecture as Architecture,
    HOST->os->build as OS_Build,
    HOST->os->type as OS_Type,
    HOST->os->platform as OS_Platform,
    HOST->os->version as OS_Version,
    HOST->os->family as OS_Family,
    HOST->os->name as OS_Name,
    HOST->os->kernel as OS_Kernel,
    HOST->id as Host_ID,
    HOST->ip as Host_IP_Array,
    agent->version as Agent_version,
    agent->hostname as Agent_hostname,
    agent->ephemeral_id as Agent_ephemeral_id,
    agent->id as Agent_id,
    agent->name as Agent_name,
    agent->type as Agent_type,
    ecs->version as ECS_version,
    log->offset as Log_Offset,
    log->file->path as Log_Filepath
from FILEBEAT_STREAM_INIT;
</pre>

In the above statement, note that the first field "MESSAGE" is just what it is - a single arbitrary field. The rest of them are STRUCTs, nested JSON, so I can flatten by traversing the hierarchy to get the fields I want with the arrow notation. 

(See ksqlDB.io for usage documentation)

Because this is a CSAS statement (a CTAS would do the same), an output topic is created where the results get produced. Go to **FILEBEAT_STREAM_OUTPUT** and inspect the records.  You can also just query it in ksqlDB like "select * from FILEBEAT_STREAM_OUTPUT emit changes;" and as long as "Earliest" is chosen for for the auto.offset.reset you should see all the data from the beginning of Kafka receiving data from Filebeat.

### Now we can sink down to Elasticsearch!

Go into the Connect cluster registered as "myconnect" in the Confluent Control Center and bring up the Elasticsearch Sink Connector that was created and should be in a Running state. Click on that "elasticsearch-sink" connector instance and click on Settings. Simply add the topic called **FILEBEAT_STREAM_OUTPUT** by clicking into the box where the existing topic is, and add another topic, you should see it hinted in a drop down.

Then scroll all the way down to hit Next, then Launch, and you should see the "elasticsearch-sink" connector get back to a "Running" state.

### Check to see the new index is created with the topic name

### Profit
