# Perserve Message Ordering

## Problem statement

Design a system where messages consumed by your system should perserve the order of the messages coming in such that older messages do not overwrite newer messages.

### Extra considerations
System needs to have scalability capabilities (support multi-thread/multi-server environment) while perserving integrity of message order

## Requirements specification

### User requirements
- User can insert a new 'message' into db
- User can update an existing message into db

### Quality requirements
- Message order integrity needs to be perserved (older messages cannot overwrite newer ones)
- System must be scalable to handle concurrent requests (must not violate message order)
- System downtime should be minimized / should not disturb message order
- Messages that 'fail' must be handled (don't overwrite newer messages)
- Logging capabilities to track down events and identify issuess
### Risks and assumptions
We will assume that a message is 'recieved' into the system at the moment the kafka record of it is made.  
Since it is possible that two requests made from the client at the same time might arrive at different times

'Partial' update will refer to an object that is 'incomplete' ie includes `ID`, `Data` fields, but does not include `Enable` field

Message syntax
```
message: {
    id,
    enabled,
    data,
}
```

## Design choices & considerations

### Design Choice: Use Kafka
- Must use some form of 'message broker' to queue requests and maintain order that requests arrived in.
- This message broker must implement the Pub/Sub pattern with some extra considerations.
- Must be able to maintain several identifable queues
- Must provide capability of selecting which queue line a message is published to
- Must provide capability of assigning queue lines to specific consumers
- Kafka maintains the order of requests that are published to its topic
- allows for publishers and consumers 
- topics can have identifiable partitions (queue lines)
- partition can have assigned consumer IDs such that only one of a multiple of consumers subscribed to the topic will consume from that partition


### Design Choice: Requests are grouped by message id and published to a consistent parition in the topic such that all the messages of a specific id can be found in only one partition
- System needs to ensure that requests made with same id are processed in the order that they arrived

### Design Choice: Can concurrently process requests of different ids with multiple consumers but same ids must run in sequence
- Quality requirement of performance requires capability of scaling the system up
- Quality requirement that related messages must process in order requires running those in sequence and in order

### Design Choice: Only 1 consumer can be subscribed to a partition
- To ensure that the requests are processing in order, the system needs to ensure that only 1 consumer is subscribed to a partition
- A consumer will process requests in its topic partition in sequence by oldest one first
- This ensures that no related messages are processed concurrently and eliminates chance of newer messags being overwritten by older ones



### Design Choice: Max number of consumers is the number of partitions available in kafka
- Since a partition can only have one consumer assigned to it, the max number of consumers the system can handle is at most the number of partitions in the kafka topic
- Ie, two consumers can't read off the same partition since that will mean two requests with the same ID might run concurrently

### Design Choice: A consumer can have multiple assigned partitions
- A partition can have only one consumer listening to it, but a consumer can listen to multiple partitions, since the ordering and sequential processing of the requests of the same ID is maintained.

### Design Choice: Partitions can have many publishers assigned to them
- Many publishers can publish to the topic in the appropriate parition, the requests will be processed in the order they arrive into the topic

### Design Choice: System infrastructure will be set up within a cloud environment
- Set up system using a cloud service platform (AWS, IBM Cloud, etc) 
- Can use Kubernetes to handle deployments, restart failing pods, update pods using rollouts, handle secrets, handle endpoint access, etc
- Can utilize CI/CD pipeline with integrated testing and multi environment deployment
- Essential third party services available (Kafka, databases)



## Software Architecture
### Overall system
![overall_design](/overall_design.png)
1. Clients connect to Kafka instance, publish requests in topic. A hash_id is generated to ensure conistent partition destination is for the same message id.
2. Subscriber assigned to the partition will read in requets sequentiallty
3. Can handle multiple consumers (<= number of partitions available) that will run concurrently
4. On successful processing of request (updating db), consumer can update kafka record and move onto the next request in the queue

## Subsystem designs
### Event stream (Kafka)
![kafka_design](/kafka-design.png)
- kafka can have 1 or more partitions on the `upsert` topic
- This will allow us to run our system concurrently with multiple consumers while still enforcing message order
- `upsert` topic can have 1 to many publishers, but number of consumers are enforced to at most the number of partitions available (because two consumers cannot work on the same partition and enforce order. This limitation ensurs message order consitency)
### Publisher (Client subsystem)
- Will publish client requests (such as `upsert`) to the kafka topic
- Will perform hashing function to group ids so that it can publish topic to consistent partitions
- To know which partition the message should be created to, perform some form of hashing function on the id.

 `id % number_of_active_partitions = partition_number`
 
This will identify to the client which partition in the topic it should publish the record to.

Example

A request is made by the client
```
function identifyPartitionID(message, number_of_partitions){
    return message.id % number_of_partitions
}
```

This will ensure that every request with the ID `x` will be published to partition `y`
```
message.id = 30
number_of_partitions = 3
partition_id = 30 % 3 = 0
```

By enforcing this id / partition grouping, we allow our system to perserve the order of messages. Since same IDs will be put in one partition and will run sequentially, multiple consumer/partition pairs will run this process concurrently

### Consumer (Backend subsystem)
![backend-design](/backend-design.png)
- Each partition will have at most 1 consumer subscribed to it. This will ensure that all requests of the same id is handled by only one consumer.
- Consumer will sequentially process the requests found in the partition queue by oldest first.
- A consumer can listen to multiple partitions and ensure order integrity. This will allow the system to continue to function if
    1. A consumer dies and a living one has to take it's place
    1. Can run the entire system sequentially (1 consumer) or concurrently (multiple consumers) without needing to modify partitions
- consumers can run concurrently without fear of overwriting messages

### Database
![db_design](/db_design.png)
- Will contain a single table `Messages` with fields `id`, `data`, `enabled`, etc

## Edge case considerations

### Should /insert and /update be decoupled?
Depending on the user requirements, we might want to seperate the insert and update functionalities, what if the user wanted to only update with no intention to create, or vice versa?

### What if we want to shut down the system?
We can still receive flow of requests by having a backup / fallover kafka topic. This will enable shutting down main kafka topic for partition / other updates, and still recieve requests/maintain the order. When system is coming back online make sure to migrate backup topic requests into main topic

### What if a subscriber dies? In the middle of processing the request?
If a `subscriber x `dies, allow another `subscriber y` to take its place, ie becoming subscribed to that partition. When `subscriber x` is added back, ensure that `subsriber y` stops taking new requests from target partition, process any remaining requests, and then rebalance partition / consumer grouping. Only remove a request from the kafka record after it has been fully processed, if dies in the middle of processing, the request will simply be reprocessed from the beginning

### What if we want to increase # of subscribers?
A limitation of this system is that number of subscribers cannot exceed number of partitions, if `subscriber_count` < `partition_count`, spin up a new consumer, and redelegate consumer partition grouping (make sure consumer record consumption is paused). If need to increase number of partitions to increase consumers, activate backup kafka topic to keep backlog of requests, run updates, migrate backup requests into new kafka topic/partitions, then resume processing

### What if we want to decrease # of subscribers?
If we want to decrease # of subscribers, we will need the system to 'rebalance' partition/consumer grouping such that another consumer will be subscribed to that partition

### What if message processing fails?
If a message fails to be processed (inserted / updated the db), we can have a certain amount of retry rates, we can log the failure, and we can eventually disregard the message so the consumer does not hang on 1 message

### What if timestamps are unreliable?
- Can store them at the consumer level, so that our control over the environment can guarentee time stamp consistency
- Timestamp is not necessary for the integrity of order of this system. Our message broker system (Kafka) will enforce order integrity of same ID requests

### What if messages coming in have partial updates?
- Treat them how you would with any other valid message (enforce sequence and process) Simpler

### What if multiple messages with same ID are identical?
- treat them how you would with any other valid message. Simpler
- To save time and ignore unnecessary db updates, filter message data with stored data so that only changes to new values are made. Efficiency

### IDs with all empty fields?
- only process fields that are valid (not empty, valid value)
- if empty ignore

### Invalid message recieved/sent?
- only process fields that are valid (not empty, valid value)
- if empty ignore, if fail validation, client errors out before sending
- if publisher, validate messages recieved, invalidated messages are logged and discarded

## Alternatives considered
### Having multiple consumers somehow split processing requests for the same id
- Although this will increase the efficiency of our system, especially if requests being made heavily favour a specific id (which means those requests would have to run sequentially), we just cannot gaurentee order enforcement anymore, since a concurrently running consumer may finish a newer request before others
- Given the use cases of this system however I think that there will be a fair amount of distribution in terms of request ids made.

### Deleting requests that are identical
- We could check other requests in the partition that match the one the consumer is currently looking at, decreasing processing time for unecessary actions, however this presents additional complexity to our system that does not pay off in terms of performance
- additionally we can check identical fields when performing the db update, and filter them out

### Combining partial requests together to form full ones if can
- To make fewer db calls, we can combine partial requests together if the fields do not conflict with one another, however this increases the complexity of our system
- can just treat these requests as normal, complete ones
### Timestamping
- While timestamping can help enforce ordering, often times different systems may have inconsistent times
- Our design does not depend on accurate timestamping, kafka (message broker) will act as the agent enforcing request queueing / maintaining order of requests made


