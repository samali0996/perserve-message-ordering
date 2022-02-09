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
- Must use some form of 'message broker' to queue requests and maintain order that requests arrived in.
- This message broker must implement the Pub/Sub pattern with some extra considerations.
- Must be able to maintain several identifable queues
- Must provide capability of selecting which queue line a message is published to
- Must provide capability of assigning queue lines to specific consumers

Design Choce: Use Kafka
- Kafka maintains the order of requests that are published to its topic
- allows for publishers and consumers 
- topics can have identifiable partitions
- partition can have assigned consumer IDs such that only one of the multiple consumers subscribed to the topic will consume from that partition


- System needs to ensure that requests made of with same id are processed in the order that they arrived


Design Choice: Requests are grouped by message id and will be grouped to a single parition in the topic such that all the messages of a specific id can be found in only one partition


- Quality requirement of performance requires capability of scaling the system up
- Quality requirement that related messages must process in order requires running requests grouped by id in sequence and in order

Design Choice: Can concurrently process requests of different ids with multiple consumers

- To ensure that the requests are processing in order, the system needs to ensure that only 1 consumer is subscribed to a partition
- A consumer will process requests in its topic partition in sequence by oldest one first

Design Choice: Only 1 consumer can be subscribed to each topic 

Design Choice: Max number of consumers is the number of partitions available in kafka

Design Choice: A consumer can have multiple assigned partitions

Design Choice: Partitions can have many publishers assigned to them

## Software Architecture
### Overall system

## Subsystem designs
### Event stream (Kafka)
- kafka can have 1 or more partitions on the `upsert` topic
- This will allow us to run our system concurrently with multiple consumers while still enforcing message order
- `upsert` topic can have 1 to many publishers, but number of consumers are enforced to at most the number of partitions available (because two consumers cannot work on the same partition and enforce consist. This satisfieDesign ensuring message order consitency)
- 
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

By enforcing this id / partition grouping, we allow our system to perserve the order of messages

### Consumer (Backend subsystem)
- Each partition will have at most 1 consumer subscribed to it. This will ensure that all requests are grouped uniquely by id. Will run in sequence
- A consumer can listen to multiple partitions. This will allow the system to continue to function if
    1. A consumer dies and 
    1. ds  

### Database

### Infrastructure
Kubernetes to handle deployments, restart failing pods, update system,
Can use cloud technology
- secrets store
- handle 

### CI/CD

## Userflows 
### User adds message
### User updates message
### Two users 'upsert' same ID


#### Infrastructure choices  
Using a cloud service (AWS, IBM Cloud, etc)
## Tradeoffs
Added complexity

## Edge case considerations

Should /insert and /update be decoupled?


What if we want to shut down the system?

We can still receive flow of requests by having a backup / falloever kafka topic. This will enable shutting down main kafka topic for partition / other updates.

What if a subscriber dies?

We can have another subscriber take over that partition. As long as we 

What if we want to increase # of subscribers?

What if we want to decrease # of subscribers?

What if message processing fails?

What if timestamps are unreliable?
- Can store them at the subscriber level, so that our control over the environment can guarentee 
- Timestamp is not necessary for the integrity of order. Our message broker system (Kafka) will enforce order integrity of same ID requests

What if messages coming in have partial updates?
- Treat them how you would with any other valid message (enforce sequence and process) Simpler

What if multiple messages with same ID are identical?
- treat them how you would with any other valid message. Simpler
- To save time and ignore unnecessary db updates, filter message data with stored data so that only changes to new values are made. Efficiency

IDs with all empty fields?
- only process fields that are valid (not empty, valid value)
- if empty ignore

Invalid message recieved/sent?
- only process fields that are valid (not empty, valid value)
- if empty ignore, if fail validation, client errors out before sending
- if publisher, validate messages recieved, invalidated messages are logged and discarded

## Alternatives considered
- Having multiple consumers somehow split work from the same id
- Deleting requests that are identical
- Combining partial requests together to form full ones if can
- Timestamping


