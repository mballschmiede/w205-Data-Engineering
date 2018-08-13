# Project 3 
## Michael Ballschmiede

### Assignment 12 - In this assignment we are initializing a web server running a simple web API service. The goal of this server is to service web API calls by writing them to a Kafka topic. We will utilize the 'curl' utility to make API calls to our web service, manually consuming the Kafka topic we create to verify our web service is working. We are then adding a Hadoop container to the cluster and writing Python Spark code to subscirbe to the Kafka topic and write the results into Parquet format in HDFS. We will introduce the batch Python Spark interface called Spark-submit to submit our Python files (as opposed to using Pyspark like we did before. We will run a couple of variations of these Python files: one to transform the events and another to separate the events. Finally, we will add the Apache Bench utility to test our web API instead of using curl. We will edit our Spark code to allow for multiple schema type. We will add parameters to our web API calls and change the corresponding Spark code to process them. We will use Jupyter Notebook to write Pyspark code designed to read Parquet files, register them as temporary tables, and execute Spark SQL against them.

---

Before starting assignment 12, run these commands in the droplet (but not in a container) to update our Docker images:
```
docker pull confluentinc/cp-zookeeper:latest
docker pull confluentinc/cp-kafka:latest
docker pull midsw205/cdh-minimal:latest
docker pull midsw205/spark-python:0.0.5
docker pull midsw205/base:0.1.9
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ vi docker-compose.yml 
After changing to the proper directory, our first step is to create a docker-compose.yml file containing the following:
```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  mids:
    image: midsw205/base:0.1.8
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    expose:
      - "5000"
    ports:
      - "5000:5000"
    extra_hosts:
      - "moby:127.0.0.1"
 ```
 
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose up -d
Spin up our cluster
```
Starting assignment11mballschmiede_zookeeper_1
Starting assignment11mballschmiede_mids_1
Starting assignment11mballschmiede_kafka_1
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose logs -f kafka
Check the Kafka logs as they arise. The -f tells it to keep checking the file for any new additions to the file and print them. Use control-C to stop this command.

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a Kafka topic called events.
```
Created topic "events".
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ vi game_api.py
We are trying to develop of mobile game in which users can perform various actions such as purchasing swords, purchasing knives, and joining guilds. To process said actions, this mobile app makes API calls to a web-based server. We will use the Python Flask module to write a simple API server. We create a .py file with the following code to do so:
```
@app.route("/")
def default_response():
    event_logger.send(events_topic, 'default'.encode())
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_knife():
    # business logic to purchase knife
    return "\nKnife Purchased!\n"

@app.route("/join_a_guild")
def join_guild():
    # business logic to join guild
    return "\nGuild Joined!\n"
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-11-mballschmiede/game_api.py flask run
Run the Python script we just created. The 'env' command runs our program in a modified environment. Note that this will tie up our command window.
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
We exec into our mids container and utilize the 'curl' utility in another linux command line window to make web API calls. These calls are automatically assigned to our local host. Note that TCP port 5000 is the port we are using.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

Switching back to our command line window running Flask, we notice that we see output from our Python program here as we make our web API calls. We can infer from our '200' messages that our API calls were successful. We can also see here that we are using http1.1, which allows us to log in and remain connected while making repeated requests. We stop this program with control-C.
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
127.0.0.1 - - [08/Jul/2018 22:21:35] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [08/Jul/2018 22:21:38] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [08/Jul/2018 22:21:41] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [08/Jul/2018 22:21:47] "GET /join_a_guild HTTP/1.1" 200 -
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ rm game_api.py
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ vi game_api.py
Edit our Python Flask script to publish to our 'events' Kafka topic in addition to writing to standard output:
```
#!/usr/bin/env python
from kafka import KafkaProducer
from flask import Flask
app = Flask(__name__)
event_logger = KafkaProducer(bootstrap_servers='kafka:29092')
events_topic = 'events'

@app.route("/")
def default_response():
    event_logger.send(events_topic, 'default'.encode())
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    # log event to kafka
    event_logger.send(events_topic, 'purchased_sword'.encode())
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_knife():
    # business logic to purchase knife
    # log event to kafka
    event_logger.send(events_topic, 'purchased_knife'.encode())
    return "\nKnife Purchased!\n"

@app.route("/join_a_guild")
def join_guild():
    # business logic to join guild
    # log event to kafka
    event_logger.send(events_topic, 'guild_joined'.encode())
    return "\nGuild Joined!\n"
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-11-mballschmiede/game_api.py flask run
Run our latest and greatest Python script. Again note that this will tie up our command window. This command execs into the MIDS container, sets environment variables for the Flask app, feeds in our Python file, and runs Flask. Note that Flask is running on our local host.

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
After switching to a different command line window, we use 'curl' to make more web API calls.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
Utilize the Kafkacat utility to consume the messages that our web service wrote to the kafka topic. We connect to our Kafka broker on port 29092 and subscribe to our event topics from beginning to end. By writing these events to Kafka, we can potentially run our data through a lambda architecture, a speed layer and a batch layer, and ultimately do analytics on our web-based server
```
default
purchased_sword
purchased_knife
guild_joined
guild_joined
guild_joined
% Reached end of topic events [0] at offset 6: exiting
```

Return to our main window and use control-C to stop Flask.

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ vi game_api_with_json_events.py
Now let's create a a new Python file called game_api_with_json_events.py with the following Python code. Note that this is very similar to our above game_api.py file. We are still using the Python kafka module's class KafkaProducer (but we are renaming this from 'event_logger' to 'producer') and the Flask module's class Flask. Recall that we previously installed the kafka and flask modules. We are also importing JSON and moving our Kafka-logging code to a standalone function called log_to_kakfa. This function takes the 'event' object, encodes it, and turns it into a string before being sent and logged to the Kafka topic 'events'.
```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')


def log_to_kafka(topic, event):
    producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife'}
    log_to_kafka('events', purchase_knife_event)
    return "\nKnife Purchased!\n"
    

@app.route("/join_a_guild")
def join_a_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nGuild Joined!\n"
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-11-mballschmiede/game_api_with_json_events.py flask run --host 0.0.0.0
Let's run our new Python flask script in the mids container of our docker cluster. This will run and print output to the command line each time we make a web API call. It will hold the command line until we exit it with a control-C, so you will need another command line prompt:
```
 * Serving Flask app "game_api_with_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
Run our various commands in a different window
```
This is the default response!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
Run the kafkacat utility in the mids container of our docker cluster to consume the topic:
```
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "purchase_knife"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
% Reached end of topic events [0] at offset 6: exiting
```

Let's now switch back to our command line window running Flask
```
127.0.0.1 - - [16/Jul/2018 01:26:04] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:26:07] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:26:09] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:26:12] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:26:32] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:26:34] "GET /join_a_guild HTTP/1.1" 200 -
```

Use control-C to exit the flask web server. Now we will enhance the previous activity by adding more key-value attributes to the JSON objects we are publishing to kafka. We will use the pyspark python to spark interface to read the json objects into a data frame and process them.

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ vi game_api_with_extended_json_events.py 
We create a new python file game_api_with_extended_json_events.py with the following python code to add detail to our event logging. We add a line to each function which utilizes the Python request module to add additional key-value attributes to each of our events.
```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')

def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())

@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife'}
    log_to_kafka('events', purchase_knife_event)
    return "\nKnife Purchased!\n"
    
@app.route("/join_a_guild")
def join_a_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nGuild Joined!\n"
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-11-mballschmiede/game_api_with_extended_json_events.py flask run --host 0.0.0.0
Let's try once again running our Python flask script in the mids container of our docker cluster. This will run and print output to the command line each time we make a web API call. Again note that Flask will occupy the command line until we exit it with a control-C, so we will use another command line prompt to make our 'curl' calls:
```
 * Serving Flask app "game_api_with_extended_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
Make API calls via our 'curl' command, just as we did before
```
Sword Purchased!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
```
This is the default response!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
Run the kafkacat utility in the mids container of our docker cluster to consume the topic, just as we did before
```
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "purchase_knife"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_knife", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic events [0] at offset 10: exiting
```

We still see our six events from earlier. These have not changed. However, we can also see that our four most recent events have additional key-value attributes. We can now see that our events are decorated with 'host', 'accept', and 'user-agent' keys.

Switching back to our main command line prompt, we note that our Flask output looks consistent to what we saw before adding the enhancements. Let's again exit Flask with a control-C.
```
127.0.0.1 - - [16/Jul/2018 01:31:40] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:31:43] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:31:46] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [16/Jul/2018 01:31:49] "GET /join_a_guild HTTP/1.1" 200 -
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec spark pyspark
Using yet another command line window, we start a Pyspark shell.
```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.2.0
      /_/

```

### >>> raw_events = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","events").option("startingOffsets", "earliest").option("endingOffsets", "latest").load() 
### >>> raw_events.cache()
We use Pyspark to consume and cachce our Kafka topic 'events':
```
DataFrame[key: binary, value: binary, topic: string, partition: int, offset: bigint, timestamp: timestamp, timestampType: int]
```

### >>> events = raw_events.select(raw_events.value.cast('string'))
As we have done several times before, we need to convert our binary values into a more more human-readable string format.

### >>> import json
### >>> extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF() 
Let's now extract our string values into individuals JSON objects
```
/spark-2.2.0-bin-hadoop2.6/python/pyspark/sql/session.py:351: UserWarning: Using RDD of dict to inferSchema is deprecated. Use pyspark.sql.Row instead
  warnings.warn("Using RDD of dict to inferSchema is deprecated. "
```

### >>> extracted_events.show()
Take a look at the extracted JSON values. Note here that because some of our events did not have the additional enhancements, we can only see the 'event_type' key consistent to all of them in our data frame. 

```
+--------------+
|    event_type|
+--------------+
|       default|
|purchase_sword|
|purchase_knife|
|    join_guild|
|    join_guild|
|    join_guild|
|purchase_sword|
|purchase_knife|
|       default|
|    join_guild|
+--------------+
```

### >>> extracted_events
Let's take a closer look at our extracted_events object
```
DataFrame[event_type: string]
```
### >>> extracted_events.printSchema()
```
root
 |-- event_type: string (nullable = true)
```

### As an aside, we could have worked around the inconsistent schemas by using the following code in Spark:

### >>> import json
### >>> json_schema = spark.read.json(events.rdd.map(lambda row: row.value)).schema
Read the schema using Python's built-in JSON reading module

### >>> update_extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF(json_schema)
### >>> update_extracted_events.show()
(Note that these event API calls are different than the ones we made before)
```
+------+--------------+-----------+--------------+
|Accept|          Host| User-Agent|    event_type|
+------+--------------+-----------+--------------+
|  null|          null|       null|purchase_knife|
|   */*|localhost:5000|curl/7.47.0|purchase_sword|
|   */*|localhost:5000|curl/7.47.0|purchase_knife|
|   */*|localhost:5000|curl/7.47.0|purchase_knife|
+------+--------------+-----------+--------------+
```

### >>> exit()
Exit Pyspark

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose down
Return to our Flask window and stop Flask with control-C. Then tear down our cluster. 
```
Stopping assignment11mballschmiede_kafka_1 ... done
Stopping assignment11mballschmiede_mids_1 ... done
Stopping assignment11mballschmiede_zookeeper_1 ... done
Removing assignment11mballschmiede_kafka_1 ... done
Removing assignment11mballschmiede_mids_1 ... done
Removing assignment11mballschmiede_zookeeper_1 ... done
Removing network assignment11mballschmiede_default
```

### science@w205s4-crook-0:~$ docker ps -a
Verify our cluster is properly down
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

### science@w205s4-crook-0:~$ docker run -it --rm -v /home/science/w205:/w205 midsw205/base:latest bash
Run docker with the MIDS image in our droplet

### root@e260abaa09b9:~/assignment-11-mballschmiede# rm docker-compose.yml
### root@e260abaa09b9:~/assignment-11-mballschmiede# vi docker-compose.yml
Create a new .yml file with the following code. We are adding the Cloudera Hadoop container to what we have done so far. We are also exposing port 5000 so we can connect to our Flask web API to both the droplet and from our desktop.
```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  cloudera:
    image: midsw205/cdh-minimal:latest
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"
    extra_hosts:
      - "moby:127.0.0.1"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    expose:
      - "8888"
    ports:
      - "8888:8888"
    volumes:
      - "/home/science/w205:/w205"
    command: bash
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera
    extra_hosts:
      - "moby:127.0.0.1"

  mids:
    image: midsw205/base:latest
    stdin_open: true
    tty: true
    expose:
      - "5000"
    ports:
      - "5000:5000"
    volumes:
      - "/home/science/w205:/w205"
    extra_hosts:
      - "moby:127.0.0.1"
```

### root@e260abaa09b9:~/assignment-11-mballschmiede# cp /w205/course-content/11-Storing-Data-III/&ast;.py .
### root@e260abaa09b9:~/assignment-11-mballschmiede# ls
Copy the .py files we will be using and ensure we now have all necessary files in our directory
```
README.md  docker-compose.yml  extract_events.py  game_api.py  separate_events.py  transform_events.py
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose up -d
Spin up our Docker cluster. 
```
Creating network "assignment11mballschmiede_default" with the default driver
Creating assignment11mballschmiede_zookeeper_1
Creating assignment11mballschmiede_cloudera_1
Creating assignment11mballschmiede_mids_1
Creating assignment11mballschmiede_spark_1
Creating assignment11mballschmiede_kafka_1
```

Wait for the cluster to come up. Open a separate linux command line window for each of these:

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose logs cloudera
Cloudera Hadoop may take a while to come up. 
```
Attaching to assignment11mballschmiede_cloudera_1
cloudera_1   | Start HDFS
cloudera_1   | starting datanode, logging to /var/log/hadoop-hdfs/hadoop-hdfs-datanode-deed4a8c798a.out
cloudera_1   |  * Started Hadoop datanode (hadoop-hdfs-datanode): 
cloudera_1   | starting namenode, logging to /var/log/hadoop-hdfs/hadoop-hdfs-namenode-deed4a8c798a.out
cloudera_1   |  * Started Hadoop namenode: 
cloudera_1   | starting secondarynamenode, logging to /var/log/hadoop-hdfs/hadoop-hdfs-secondarynamenode-deed4a8c798a.out
cloudera_1   |  * Started Hadoop secondarynamenode: 
cloudera_1   | Start Components
cloudera_1   | Press Ctrl+P and Ctrl+Q to background this process.
cloudera_1   | Use exec command to open a new bash instance for this instance (Eg. "docker exec -i -t CONTAINER_ID bash"). Container ID can be obtained using "docker ps" command.
cloudera_1   | Start Terminal
cloudera_1   | Press Ctrl+C to stop instance.
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec cloudera hadoop fs -ls /tmp/
Note to check the Hadoop file system to see how eventual consistency works for the two directories; it may take Yarn and Hive a minute to both show up.
```
Found 2 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-27 02:45 /tmp/hive
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose logs -f kafka
Check Kafka logs. Note that sometimes Kafka has to reorganize and it can take a while to come up. Remember to use control-C to exit processes with the -f option.
```
Attaching to assignment11mballschmiede_kafka_1
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a topic in Kafka
```
Created topic "events".
```

### root@e260abaa09b9:~/assignment-11-mballschmiede# vi game_api.py
In a different container, vi into our game_api.py file and add the necessary "purchase knife" & "join guild" logic
```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')

def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())

@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"
    
@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife'}
    log_to_kafka('events', purchase_knife_event)
    return "\nKnife Purchased!\n"

@app.route("/join_a_guild")
def join_a_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nGuild Joined!\n"
```
                                                                                                                                                                                       
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-11-mballschmiede/game_api.py flask run --host 0.0.0.0
Run flask with our game_api.py python code:
```
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

In another linux command line window, use curl to test our web API server (commands below condensed for concision)
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
```This is the default response!```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```Sword Purchased!```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```Knife Purchased!```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```Guild Joined!```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
Read the topic in Kafka to see the generated events, same as we have done before.
```
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_knife", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic events [0] at offset 11: exiting
```

We now review the following Python Spark file extract_events.py. Instead of using pyspark we will now be using spark-submit to run our application. 
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""

import json
from pyspark.sql import SparkSession


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    events = raw_events.select(raw_events.value.cast('string'))
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()

    extracted_events \
        .write \
        .parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec spark spark-submit /w205/assignment-11-mballschmiede/extract_events.py
Submit our extract_events.py file to spark using spark-submit. We are extracting our events from Kafka and writing them to HDFS.

Note that if we try to use this command without first publishing any Kafka events, we will get the following output telling us that our RDD is empty:
```
Traceback (most recent call last):
  File "/w205/spark-from-files/extract_events.py", line 35, in <module>
    main()
  File "/w205/spark-from-files/extract_events.py", line 27, in main
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()
  File "/spark-2.2.0-bin-hadoop2.6/python/lib/pyspark.zip/pyspark/sql/session.py", line 57, in toDF
  File "/spark-2.2.0-bin-hadoop2.6/python/lib/pyspark.zip/pyspark/sql/session.py", line 535, in createDataFrame
  File "/spark-2.2.0-bin-hadoop2.6/python/lib/pyspark.zip/pyspark/sql/session.py", line 375, in _createFromRDD
  File "/spark-2.2.0-bin-hadoop2.6/python/lib/pyspark.zip/pyspark/sql/session.py", line 346, in _inferSchema
  File "/spark-2.2.0-bin-hadoop2.6/python/lib/pyspark.zip/pyspark/rdd.py", line 1364, in first
ValueError: RDD is empty
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec cloudera hadoop fs -ls /tmp/
Let's verify our code wrote to the Hadoop HDFS file system:
```
Found 3 items
drwxr-xr-x   - root   supergroup          0 2018-07-28 01:12 /tmp/extracted_events
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-28 01:02 /tmp/hive
```
### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec cloudera hadoop fs -ls /tmp/extracted_events
```
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-28 01:12 /tmp/extracted_events/_SUCCESS
-rw-r--r--   1 root supergroup       1212 2018-07-28 01:12 /tmp/extracted_events/part-00000-2c4cb4b3-d73d-4583-9124-8b535f35e907-c000.snappy.parquet
```

An aside that the following command that we used earlier:
```docker-compose exec spark spark-submit filename.py```

is short for this:
```
docker-compose exec spark \
  spark-submit \
    --master 'local[*]' \
    filename.py
```

We are running a spark "pseudo-distributed" cluster (aka not really a cluster with a "master" and "workers"). If we run a standalone cluster with a master node and worker nodes, we have to be more specific :
```
docker-compose exec spark \
  spark-submit \
    --master spark://23.195.26.187:7077 \
    filename.py
```

If we were running our spark inside of a hadoop cluster, we would need to submit to yarn which is the resource manager for Hadoop 2:
```
docker-compose exec spark \
  spark-submit \
    --master yarn \
    --deploy-mode cluster \
    filename.py
```

If we were running our spark inside of a mesos cluster, we would need to submit to mesos master:
```
docker-compose exec spark \
  spark-submit \
    --master mesos://mesos-master:7077 \
    --deploy-mode cluster \
    filename.py
```

If we were running our spark inside of a kubernetes cluster, we would need to submit to kubernetes master:
```
docker-compose exec spark \
  spark-submit \
    --master k8s://kubernetes-master:443 \
    --deploy-mode cluster \
    filename.py
```

End of aside. Let's now review the Python file transform_events.py below. Note we change the event "Host" to "moe" and the event "Cache-Control" to "no-cache". We also would like to display our events, which we do using a ".show", and overwrite any Parquet files.
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""

import json
from pyspark.sql import SparkSession


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    events = raw_events.select(raw_events.value.cast('string'))
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()

    extracted_events \
        .write \
        .parquet("/tmp/extracted_events")

if __name__ == "__main__":
    main()
```           

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose exec spark spark-submit /w205/assignment-11-mballschmiede/transform_events.py
Run the Python file transform_events.py. The resulting code is largely unreadable to humans but we do see the following data frame outputted. Notice that all of our events are grouped together.
```
+------+-------------+----+-----------+--------------+--------------------+
|Accept|Cache-Control|Host| User-Agent|    event_type|           timestamp|
+------+-------------+----+-----------+--------------+--------------------+
|   */*|     no-cache| moe|curl/7.47.0|       default|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|       default|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|purchase_knife|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|    join_guild|2018-07-28 01:10:...|
|   */*|     no-cache| moe|curl/7.47.0|    join_guild|2018-07-28 01:10:...|
|   */*|     no-cache| moe|curl/7.47.0|    join_guild|2018-07-28 01:10:...|
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-28 01:10:...|
+------+-------------+----+-----------+--------------+--------------------+
```

Finally, let's check out our separate_events.py Python file. Here we seperate our four events: default, sword purchases, knife purchases, & guild joins.
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe" # silly change to show it works
    event['Cache-Control'] = "no-cache"
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()
    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))

    extracted_events = munged_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.munged))) \
        .toDF()

    sword_purchases = extracted_events \
        .filter(extracted_events.event_type == 'purchase_sword')
    sword_purchases.show()
    # sword_purchases \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/sword_purchases")

    default_hits = extracted_events \
        .filter(extracted_events.event_type == 'default')
    default_hits.show()
    # default_hits \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/default_hits")
    knife_purchases = extracted_events \
        .filter(extracted_events.event_type == 'purchase_knife')
    knife_purchases.show()
    # knife_purchases \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/knife_purchases")

    guild_joins = extracted_events \
        .filter(extracted_events.event_type == 'join_guild')
    guild_joins.show()
    # guild_joins \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/guild_joins")


if __name__ == "__main__":
    main()
```

Run our separate_events.py Python file and review the results. Again, the output is largely unreadable to humans but the main takeaway here is that our events are split up by type. i.e. the knife purchases are separated from the default events and the guild joins are separated from the sword purchases. (Note that the following output is truncated)
```
+------+-------------+----+-----------+----------+--------------------+
|Accept|Cache-Control|Host| User-Agent|event_type|           timestamp|
+------+-------------+----+-----------+----------+--------------------+
|   */*|     no-cache| moe|curl/7.47.0|   default|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|   default|2018-07-28 01:09:...|
+------+-------------+----+-----------+----------+--------------------+

+------+-------------+----+-----------+--------------+--------------------+
|Accept|Cache-Control|Host| User-Agent|    event_type|           timestamp|
+------+-------------+----+-----------+--------------+--------------------+
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-28 01:09:...|
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-28 01:10:...|
+------+-------------+----+-----------+--------------+--------------------+

+------+-------------+----+-----------+--------------+--------------------+
|Accept|Cache-Control|Host| User-Agent|    event_type|           timestamp|
+------+-------------+----+-----------+--------------+--------------------+
|   */*|     no-cache| moe|curl/7.47.0|purchase_knife|2018-07-28 01:09:...|
+------+-------------+----+-----------+--------------+--------------------+

+------+-------------+----+-----------+----------+--------------------+
|Accept|Cache-Control|Host| User-Agent|event_type|           timestamp|
+------+-------------+----+-----------+----------+--------------------+
|   */*|     no-cache| moe|curl/7.47.0|join_guild|2018-07-28 01:10:...|
|   */*|     no-cache| moe|curl/7.47.0|join_guild|2018-07-28 01:10:...|
|   */*|     no-cache| moe|curl/7.47.0|join_guild|2018-07-28 01:10:...|
+------+-------------+----+-----------+----------+--------------------+
```

### science@w205s4-crook-0:~/w205/assignment-11-mballschmiede$ docker-compose down
Tear down our cluster
```
Stopping assignment11mballschmiede_kafka_1 ... done
Stopping assignment11mballschmiede_spark_1 ... done
Stopping assignment11mballschmiede_zookeeper_1 ... done
Stopping assignment11mballschmiede_cloudera_1 ... done
Stopping assignment11mballschmiede_mids_1 ... done
Removing assignment11mballschmiede_kafka_1 ... done
Removing assignment11mballschmiede_spark_1 ... done
Removing assignment11mballschmiede_zookeeper_1 ... done
Removing assignment11mballschmiede_cloudera_1 ... done
Removing assignment11mballschmiede_mids_1 ... done
Removing network assignment11mballschmiede_default
```

---

###  Now let's start over, implementing both Cloudera & Jupyter Notebooks.
Let's review and edit our .yml file, which can be seen below. Note that we are adding Cloudera.
```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
       expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  cloudera:
    image: midsw205/cdh-minimal:latest
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"
    extra_hosts:
      - "moby:127.0.0.1"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
         expose:
      - "8888"
    ports:
      - "8888:8888"
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera
    extra_hosts:
      - "moby:127.0.0.1"
    command: bash

  mids:
    image: midsw205/base:0.1.9
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    expose:
      - "5000"
    ports:
      - "5000:5000"
    extra_hosts:
      - "moby:127.0.0.1"
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose up -d
Spin up our cluster
```
Creating network "assignment12mballschmiede_default" with the default driver
Creating assignment12mballschmiede_mids_1
Creating assignment12mballschmiede_cloudera_1
Creating assignment12mballschmiede_zookeeper_1
Creating assignment12mballschmiede_kafka_1
Creating assignment12mballschmiede_spark_1
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a Kafka topic called 'events'
```
Created topic "events".
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-12-mballschmiede/game_api.py flask run --host 0.0.0.0
Let's run our Python Flask code for our web API server as we have done before. Note that if desired, we can review the code seen in our game_api.py Python file above.
```
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning
In a separate command line window, let's now run Kafkacat in continuous mode so we can see events in real-time as they come through. We do this by leaving off the "-e" to give the endpoint. Note that we shouldn't expect any output at this point as we have yet to publish any events.

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/
Apache Bench is a utility designed to stress test web servers using a high volume of data in a short amount of time. While this is a great way to test web servers, it is important to note that even the best testing can not replicate a real-time production environment where an extraordinary number of requests can come from all around the world at any given time. We will use Apache Bench as shown below to generate multiple requests of the same thing. The "-n" option is used below to specify 10 of each. 

There are several takeaways from the output below. We can see that the server hostname is "localhost" and the server port is "5000." This should not be surprising given our command. More interestingly, we can see that the document length was 31 bytes and a total of 0.029 seconds was taken to run 10/10 complete & successful requsts. A total of 1860 bytes was transferred, 310 of which was HTML. We can also see some statistics about the individual requests. The mean time was 2.864 ms, 66% of the requests were made in 3 ms or less, and the longest request was served in 6 ms.
```
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient).....done


Server Software:        Werkzeug/0.14.1
Server Hostname:        localhost
Server Port:            5000

Document Path:          /
Document Length:        31 bytes

Concurrency Level:      1
Time taken for tests:   0.029 seconds
Complete requests:      10
Failed requests:        0
Total transferred:      1860 bytes
HTML transferred:       310 bytes
Requests per second:    349.16 [#/sec] (mean)
Time per request:       2.864 [ms] (mean)
Time per request:       2.864 [ms] (mean, across all concurrent requests)
Transfer rate:          63.42 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   1.0      0       3
Processing:     2    2   0.7      3       3
Waiting:        0    1   1.1      1       3
Total:          2    3   1.4      3       6
WARNING: The median and mean for the processing time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      4
  80%      4
  90%      6
  95%      6
  98%      6
  99%      6
 100%      6 (longest request)
 ```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/purchase_a_sword
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/purchase_a_sword
Let's now run three similar, but different commands. Output is suppressed but is very much analogous to the above.

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ vi just_filtering.py 
Last time we wrote Spark code using Python and submitted it using Spark-submit. This code can be seen above in the separate_events.py Python file and should be reviewed before moving onto our latest and greatest code. Note that the separate_events.py code can only handle one schema for events and would break if we tried to give it different schemas for different events. Also note that we need to alter the is_purchase logic to appropriately handle both knife and sword purchases.
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] in ('purchase_sword', 'purchase_knife'):
        return True
    return False


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()


if __name__ == "__main__":
    main()
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec spark spark-submit /w205/assignment-12-mballschmiede/just_filtering.py
Use this code to run our above just_filtering.py using Spark-submit. As before, most of the output is unreadable to humans but we do see that our data frame of purchase events (20 total - 10 sword purchases via Comcast and 10 via AT&T) was returned.
```
+------+-----------------+---------------+--------------+--------------------+
|Accept|             Host|     User-Agent|    event_type|           timestamp|
+------+-----------------+---------------+--------------+--------------------+
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-28 16:44:...|
+------+-----------------+---------------+--------------+--------------------+
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ vi game_api.py
Let's play around with our flask web API server. Stop the flask web API server. Add a new attribute to our purchase_knife event. 
```
@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife',
                            'description': 'very sharp knife'}
    log_to_kafka('events', purchase_knife_event)
    return "Knife Purchased!\n"
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-12-mballschmiede/game_api.py flask run --host 0.0.0.0
Restart the flask web API server.
```
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ vi filtered_writes.py
Modify our spark code to handle this new event attribute. We want our code to write out using massively parallel processing to Hadoop HDFS in Parquet format. Previously, we got an error if the directory already existed and had to delete it or pick a new name for the directory. We will now use the overwrite option. Remember that we want to do it this way so we can read it back in quickly if it's a large data set. 

A couple notes about the below file:

-we have two user-defined functions: the first is a boolean function which returns whether or not our event is defined as a "purchase event." As the name suggests, purchase_sword and purchase_knife events fall under this umbrella. Our second UDF, munge_event, is included so our code doesn't break when we try to feed it multiple schemas. After adding the "description: very sharp knife" attribute to our purchase_knife event, this event schema is left with one more value than the others. Trying to run code without the string UDF munge_event would result in an error message such as the following:
```
IllegalStateException: Input row doesn't have expected number of values required by the schema. 5 fields are required while 6 values are provided.
```
Including this UDF ensures consistent schemas across events by adding the "description: none" attribute to those events without a given "description" attribute.

-we output our results at several points throughout the code to help trouble-shoot and better understand what each chunk of code is doing. We include the ```extracted_purchase_events.show(100)``` tidbit because we want to see our entire data frame, not just the first 20 events.

```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] in ('purchase_sword',  'purchase_knife'):
        return True
    return False

@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    if 'description' not in event:
        event['description'] = "none" # silly change to show it works
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'), raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))
        
    munged_events.show()
    
    purchase_events = munged_events \
        .select(munged_events.munged.cast('string').alias('raw'),
                munged_events.timestamp.cast('string')) \
                        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    
    extracted_purchase_events.show(100)

    extracted_purchase_events \
        .write \
        .mode('overwrite') \
        .parquet('/tmp/purchases')


if __name__ == "__main__":
    main()
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/purchase_a_knife
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/join_a_guild
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/purchase_a_knife
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/join_a_guild
Let's use Apache Bench to make more API calls to make things more interesting.

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec spark spark-submit /w205/assignment-12-mballschmiede/filtered_writes.py
Submit to Spark-submit as we have done before.
```
root
 |-- Accept: string (nullable = true)
 |-- Host: string (nullable = true)
 |-- User-Agent: string (nullable = true)
 |-- description: string (nullable = true)
 |-- event_type: string (nullable = true)
 |-- timestamp: string (nullable = true)
 ```
 ```
 +------+-----------------+---------------+----------------+--------------+--------------------+
|Accept|             Host|     User-Agent|     description|    event_type|           timestamp|
+------+-----------------+---------------+----------------+--------------+--------------------+
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|user1.comcast.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|            none|purchase_sword|2018-08-03 00:39:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|    user2.att.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:43:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
|   */*|user1.comcast.com|ApacheBench/2.3|very sharp knife|purchase_knife|2018-08-03 00:44:...|
+------+-----------------+---------------+----------------+--------------+--------------------+
```
```
{
  "type" : "struct",
  "fields" : [ {
    "name" : "Accept",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  }, {
    "name" : "Host",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  }, {
    "name" : "User-Agent",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  }, {
    "name" : "description",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  }, {
    "name" : "event_type",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  }, {
    "name" : "timestamp",
    "type" : "string",
    "nullable" : true,
    "metadata" : { }
  } ]
}
```

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec cloudera hadoop fs -ls /tmp/
Check HDFS to ensure it's there.
```
Found 3 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-28 16:43 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-07-28 17:01 /tmp/purchases
```
### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose exec cloudera hadoop fs -ls /tmp/purchases/
```
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-28 17:01 /tmp/purchases/_SUCCESS
-rw-r--r--   1 root supergroup       1646 2018-07-28 17:01 /tmp/purchases/part-00000-c812bcd8-c76f-4b74-b4ea-dc8188e36c1d-c000.snappy.parquet
```

### docker-compose exec spark env PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS='notebook --no-browser --port 8888 --ip 0.0.0.0 --allow-root' pyspark
Startup a jupyter notebook. Remember that to access it from our laptop web browser, we will need to change the IP address to the IP address of our droplet.

In our jupyter notebook, run each of the following in a separate cell.

```
purchases = spark.read.parquet('/tmp/purchases')
purchases.show()
purchases.registerTempTable('purchases')
purchases_by_example2 = spark.sql("select * from purchases where Host = 'user1.comcast.com'")
purchases_by_example2.show()
df = purchases_by_example2.toPandas()
df.describe()
```

Let's discuss the "easy" spark workflow using the "Netflix architecture": we usually receive files in csv or json format, which in a cloud environment, we may want to put in object store (such as AWS S3). We load the file (sequentially - not parallel) into spark. We filter and process the data until we get it into spark tables like we need for analytics. We use SQL as much as we can, go to lambda transforms for things that we cannot, and using special purpose libraries, such as MLLib for machine learning. We save the file out using massively parallel processing to object store. We may also write our results out to object store. At this point our cluster can die and our data in object store will outlive the cluster (similar concept to our docker volume mount outliving the docker container). Next time we need our data, we can read it back in using massively parallel processing, which will be much faster than the original sequential read.

Note: netflix architecture is actually a very specific architecture using object store (AWS S3) and an elastic form of hadoop cluster (AWS EMR - Elastic MapReduce). However, industry slang tends to call any use of object store => load and process in a temporary cluster => save results to object store as "netflix architecture".

### science@w205s4-crook-0:~/w205/assignment-12-mballschmiede$ docker-compose down
Tear down our cluster (as before):
```
Stopping assignment12mballschmiede_spark_1 ... done
Stopping assignment12mballschmiede_kafka_1 ... done
Stopping assignment12mballschmiede_cloudera_1 ... done
Stopping assignment12mballschmiede_zookeeper_1 ... done
Stopping assignment12mballschmiede_mids_1 ... done
Removing assignment12mballschmiede_spark_1 ... done
Removing assignment12mballschmiede_kafka_1 ... done
Removing assignment12mballschmiede_cloudera_1 ... done
Removing assignment12mballschmiede_zookeeper_1 ... done
Removing assignment12mballschmiede_mids_1 ... done
Removing network assignment12mballschmiede_default
```