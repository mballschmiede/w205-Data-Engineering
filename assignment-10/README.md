# Project 3 Setup

- You're a data scientist at a game development company.  
- Your latest mobile game has two events you're interested in tracking: 
- `buy a sword` & `join guild`...
- Each has metadata

## Project 3 Task
- Your task: instrument your API server to catch and analyze event types.
- This task will be spread out over the last four assignments (9-12).

## Project 3 Task Options 

- All: Game shopping cart data used for homework 
- Advanced option 1: Generate and filter more types of items.
- Advanced option 2: Enhance the API to accept parameters for purchases (sword/item type) and filter
- Advanced option 3: Shopping cart data & track state (e.g., user's inventory) and filter


---

# Assignment 10

## Follow the steps we did in class 


### Turn in your `/assignment-10-<user-name>/README.md` file. It should include:
1) A summary type explanation of the example. 
  * For example, for Week 6's activity, a summary would be: "We spun up a cluster with kafka, zookeeper, and the mids container. Then we published and consumed messages with kafka."
2) Your `docker-compose.yml`
3) Source code for the flask application(s) used.
4) Each important step in the process. For each step, include:
  * The command(s) 
  * The output (if there is any).  Be sure to include examples of generated events when available.
  * An explanation for what it achieves 
    * The explanation should be fairly detailed, e.g., instead of "publish to kafka" say what you're publishing, where it's coming from, going to etc.

# Project 3 
## Michael Ballschmiede

### Assignment 10 - In this assignment we are initializing a web server running a simple web API service. The goal of this server is to service web API calls by writing them to a Kafka topic. We will utilize the 'curl' utility to make API calls to our web service, manually consuming the Kafka topic we create to verify our web service is working. 

---

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ vi docker-compose.yml 
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
 
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose up -d
Spin up our cluster
```
Starting assignment10mballschmiede_zookeeper_1
Starting assignment10mballschmiede_mids_1
Starting assignment10mballschmiede_kafka_1
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose logs -f kafka
Check the Kafka logs as they arise. The -f tells it to keep checking the file for any new additions to the file and print them. Use control-C to stop this command.

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a Kafka topic called events.
```
Created topic "events".
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ vi game_api.py
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-10-mballschmiede/game_api.py flask run
Run the Python script we just created. The 'env' command runs our program in a modified environment. Note that this will tie up our command window.
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
We exec into our mids container and utilize the 'curl' utility in another linux command line window to make web API calls. These calls are automatically assigned to our local host. Note that TCP port 5000 is the port we are using.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ rm game_api.py
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ vi game_api.py
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-10-mballschmiede/game_api.py flask run
Run our latest and greatest Python script. Again note that this will tie up our command window. This command execs into the MIDS container, sets environment variables for the Flask app, feeds in our Python file, and runs Flask. Note that Flask is running on our local host.

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
After switching to a different command line window, we use 'curl' to make more web API calls.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ vi game_api_with_json_events.py
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-10-mballschmiede/game_api_with_json_events.py flask run --host 0.0.0.0
Let's run our new Python flask script in the mids container of our docker cluster. This will run and print output to the command line each time we make a web API call. It will hold the command line until we exit it with a control-C, so you will need another command line prompt:
```
 * Serving Flask app "game_api_with_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
Run our various commands in a different window
```
This is the default response!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ vi game_api_with_extended_json_events.py 
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-10-mballschmiede/game_api_with_extended_json_events.py flask run --host 0.0.0.0
Let's try once again running our Python flask script in the mids container of our docker cluster. This will run and print output to the command line each time we make a web API call. Again note that Flask will occupy the command line until we exit it with a control-C, so we will use another command line prompt to make our 'curl' calls:
```
 * Serving Flask app "game_api_with_extended_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
Make API calls via our 'curl' command, just as we did before
```
Sword Purchased!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
```
This is the default response!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
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

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose exec spark pyspark
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

** See Update below **

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

### >>> exit()
Exit Pyspark

### science@w205s4-crook-0:~/w205/assignment-10-mballschmiede$ docker-compose down
Return to our Flask window and stop Flask with control-C. Then tear down our cluster. 
```
Stopping assignment10mballschmiede_kafka_1 ... done
Stopping assignment10mballschmiede_mids_1 ... done
Stopping assignment10mballschmiede_zookeeper_1 ... done
Removing assignment10mballschmiede_kafka_1 ... done
Removing assignment10mballschmiede_mids_1 ... done
Removing assignment10mballschmiede_zookeeper_1 ... done
Removing network assignment10mballschmiede_default
```

### science@w205s4-crook-0:~$ docker ps -a
Verify our cluster is properly down
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```




## Update: Upon review, we see we could have worked around the inconsistent schemas by using the following code in Spark:

### >>> import json
### >>> json_schema = spark.read.json(events.rdd.map(lambda row: row.value)).schema
Read the schema using Python's built-in JSON reading module

### >>> update_extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF(json_schema)
### >>> update_extracted_events.show()
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


