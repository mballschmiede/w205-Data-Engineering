# Assignment 06
## Michael Ballschmiede
  
#### science@w205s4-crook-0:~$ docker run -it --rm -v /home/science/w205:/w205 midsw205/base:latest bash
Run docker with MIDS image & volume mapping in my droplet

#### root@bdf96b70a054:~# git clone https://github.com/mids-w205-crook/assignment-06-mballschmiede.git
Clone the assignment repo via Git command line
```
Cloning into 'assignment-06-mballschmiede'...
Username for 'https://github.com': mballschmiede
Password for 'https://mballschmiede@github.com': 
remote: Counting objects: 10, done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 10 (delta 3), reused 10 (delta 3), pack-reused 0
Unpacking objects: 100% (10/10), done.
Checking connectivity... done.
```

#### root@bdf96b70a054:~# cd assignment-06-mballschmiede/
Change to cloned assignment directory

#### root@bdf96b70a054:~/assignment-06-mballschmiede# git branch assignment
#### root@bdf96b70a054:~/assignment-06-mballschmiede# git checkout assignment
Create and change to branch 'assignment' so we don't alter master branch
```
Switched to branch 'assignment'
```

#### root@d50ae5cd225e:~/assignment-06-mballschmiede# vi docker-compose.yml
Create and vi into docker-compose .yml file (file is uploaded on assignment branch)

#### root@d50ae5cd225e:~/assignment-06-mballschmiede# curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/f5bRm4
Use the curl command line utility to download JSON file from the internet to our current directory
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   391    0   391    0     0   2001      0 --:--:-- --:--:-- --:--:--  2005
100 9096k  100 9096k    0     0  1065k      0  0:00:08  0:00:08 --:--:-- 1453k
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose up -d
Bring up the headless docker cluster specified in our .yml file. We are running our containers headless, or without an interactive connection; if we want to interact with a container, we need to exec processes.
```
Creating network "assignment06mballschmiede_default" with the default driver
Creating assignment06mballschmiede_zookeeper_1
Creating assignment06mballschmiede_mids_1
Creating assignment06mballschmiede_kafka_1
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose logs -f kafka
Look at kafka logs to see kafka open. The "-f" option tells it to hold on to the command line and output new data as it arrives in the log file. (control-c to close)
```
Attaching to assignment06mballschmiede_kafka_1
kafka_1      | 
…
 (io.confluent.support.metrics.submitters.ConfluentSubmitter)
^CERROR: Aborting.
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose ps
Check and see if our cluster is running
```
      Name             Command             State              Ports       
-------------------------------------------------------------------------
assignment06mbal   /etc/confluent/d   Up                 29092/tcp,       
lschmiede_kafka_   ocker/run                             9092/tcp         
1                                                                         
assignment06mbal   /bin/bash          Up                 8888/tcp         
lschmiede_mids_1                                                          
assignment06mbal   /etc/confluent/d   Up                 2181/tcp,        
lschmiede_zookee   ocker/run                             2888/tcp,        
per_1                                                    32181/tcp,       
                                                         3888/tcp    
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose exec kafka kafka-topics --create --topic foo --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
Create a topic called :foo"
```
Created topic "foo".
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose exec kafka kafka-topics --describe --topic foo --zookeeper zookeeper:32181
Check the topic
```
Topic:foo	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: foo	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose exec mids bash -c "cat /w205/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t foo && echo 'Produced 100 messages.'"
Publish some messages to the foo topic in kafka using the kafkacat utility. The linux command line pipeline concatenates the contents of the file to standard output and pipes standard output into standard input of a new process. "jq '.[]'" returns each element of the JSON array returned in the response, one at a time, which are fed into kafkacat. The "-c" option prefixes lines by the count of their number of occurances. 
```
Produced 100 messages.
``` 

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic foo --from-beginning --max-messages 100 
Consume the messages from the foo topic in kafka using the kafka console consumer utility in the kafka container.
```
{"keen_timestamp":"1516717442.735266","max_attempts":"1.0","started_at":"2018-01-23T14:23:19.082Z","base_exam_id":"37f0a30a-7464-11e6-aa92-a8667f27e5dc","user_exam_id":"6d4089e4-bde5-4a22-b65f-18bce9ab79c8","sequences":{"questions":
...{"incomplete":0,"submitted":3,"incorrect":0,"all_correct":true,"correct":3,"total":3,"unanswered":0}},"keen_created_at":"1516234863.2837181","certification":"false","keen_id":"5a5fe86f8b8d730001eec4a6","exam_name":"Refactor a Monolithic Architecture into Microservices"}
Processed a total of 100 messages
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose down
Tear down the cluster
```
Stopping assignment06mballschmiede_kafka_1 ... done
Stopping assignment06mballschmiede_zookeeper_1 ... done
Stopping assignment06mballschmiede_mids_1 ... done
Removing assignment06mballschmiede_kafka_1 ... done
Removing assignment06mballschmiede_zookeeper_1 ... done
Removing assignment06mballschmiede_mids_1 ... done
Removing network assignment06mballschmiede_default
```

#### science@w205s4-crook-0:~/w205/assignment-06-mballschmiede$ docker-compose ps
Verify cluster is properly down



