# Project 3 Setup

- You're a data scientist at a game development company.  
- Your latest mobile game has two events you're interested in tracking: 
- `buy a sword` & `join guild`...
- Each has metadata

## Project 3 Task
- Your task: instrument your API server to catch and analyze these two
event types.
- This task will be spread out over the last four assignments (9-12).

---

# Assignment 09

## Follow the steps we did in class 
- for both the simple flask app and the more complex one.

### Turn in your `/assignment-09-<user-name>/README.md` file. It should include:
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

### Assignment 09 - In this assignment we are initializing a web server running a simple web API service. The goal of this server is to service web API calls by writing them to a Kafka topic. We will utilize the 'curl' utility to make API calls to our web service, manually consuming the Kafka topic we create to verify our web service is working.

---

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ vi docker-compose.yml 
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
 
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose up -d
Spin up our cluster
```
Starting assignment09mballschmiede_zookeeper_1
Starting assignment09mballschmiede_mids_1
Starting assignment09mballschmiede_kafka_1
```

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose logs -f kafka
Check the Kafka logs as they arise. The -f tells it to keep checking the file for any new additions to the file and print them. Use control-C to stop this command.

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a Kafka topic called events.
```
Created topic "events".
```

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ vi game_api.py
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

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-09-mballschmiede/game_api.py flask run
Run the Python script we just created. The 'env' command runs our program in a modified environment. Note that this will tie up our command window.
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 ```
 
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
We exec into our mids container and utilize the 'curl' utility in another linux command line window to make web API calls. These calls are automatically assigned to our local host. Note that TCP port 5000 is the port we are using.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
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

# science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ rm game_api.py
# science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ vi game_api.py
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

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids env FLASK_APP=/w205/assignment-09-mballschmiede/game_api.py flask run
Run our latest and greatest Python script. Again note that this will tie up our command window. This command execs into the MIDS container, sets environment variables for the Flask app, feeds in our Python file, and runs Flask. Note that Flask is running on our local host.

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/
After switching to a different command line window, we use 'curl' to make more web API calls.
```
This is the default response!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Sword Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/purchase_a_knife
```
Knife Purchased!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```
### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Guild Joined!
```

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
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

### science@w205s4-crook-0:~/w205/assignment-09-mballschmiede$ docker-compose down
After returning to our Flask window and stopping Flask with control-C, tear down our cluster.
```
Stopping assignment09mballschmiede_kafka_1 ... done
Stopping assignment09mballschmiede_mids_1 ... done
Stopping assignment09mballschmiede_zookeeper_1 ... done
Removing assignment09mballschmiede_kafka_1 ... done
Removing assignment09mballschmiede_mids_1 ... done
Removing assignment09mballschmiede_zookeeper_1 ... done
Removing network assignment09mballschmiede_default
```

### science@w205s4-crook-0:~$ docker ps -a
Verify our cluster is properly down
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
 
