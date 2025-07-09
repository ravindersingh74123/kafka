# Apache Kafka Notes

## What is Kafka?

Apache Kafka is a distributed event streaming platform that allows you to build systems where one process produces events that can be consumed by multiple consumers. You can scale Kafka horizontally by adding more nodes that run your Kafka brokers.

**Official Website:** https://kafka.apache.org/

## Use Cases

- **Distributed Event Streaming**
- **Payment Notifications**
- **Real-time Data Processing**

## Key Terminology (Jargon)

### Cluster and Broker
- **Kafka Cluster**: A group of machines running Kafka
- **Broker**: Each individual machine in the cluster

### Producers
- Used to **publish** data to a topic
- Send messages to Kafka topics

### Consumers
- **Consume** messages from a topic
- Keep track of their position using **offsets**
- Offsets represent the position of the last consumed message
- Kafka can manage offsets automatically or allow manual management

### Topics
- A logical channel to which producers send messages
- Consumers read messages from topics
- Have configurable **retention policies** determining how long data is stored before deletion
- Allow both real-time processing and historical data replay

### Partitions and Consumer Groups
- Will be covered in detail later in the notes

## Getting Started Locally

### Using Docker

```bash
# Start Kafka container
docker run -p 9092:9092 apache/kafka:3.7.1

# Get shell access to container
docker ps
docker exec -it container_id /bin/bash
cd /opt/kafka/bin
```

### Basic Kafka Commands

#### Create a Topic
```bash
./kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
```

#### Publish to the Topic
```bash
./kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
```

#### Consuming from the Topic
```bash
./kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```

## Kafka in Node.js

### Setup

```bash
# Initialize project
npm init -y
npx tsc --init
```

### Package.json Configuration

```json
{
  "scripts": {
    "start": "tsc -b && node dist/index.js",
    "produce": "tsc -b && node dist/producer.js",
    "consume": "tsc -b && node dist/consumer.js",
    "produce:user": "tsc -b && node dist/producer-user.js"
  }
}
```

Update tsconfig.json:
```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

### Install KafkaJS

```bash
npm install kafkajs
```

**Reference:** https://www.npmjs.com/package/kafkajs

### Basic Implementation

#### Combined Producer and Consumer (src/index.ts)

```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["localhost:9092"]
});

const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: "my-app3" });

async function main() {
  await producer.connect();
  await producer.send({
    topic: "quickstart-events",
    messages: [{
      value: "hi there"
    }]
  });

  await consumer.connect();
  await consumer.subscribe({
    topic: "quickstart-events", 
    fromBeginning: true
  });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      console.log({
        offset: message.offset,
        value: message?.value?.toString(),
      });
    },
  });
}

main();
```

### Separate Producer and Consumer Scripts

#### Producer (producer.ts)

```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["localhost:9092"]
});

const producer = kafka.producer();

async function main() {
  await producer.connect();
  await producer.send({
    topic: "quickstart-events",
    messages: [{
      value: "hi there"
    }]
  });
}

main();
```

#### Consumer (consumer.ts)

```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["localhost:9092"]
});

const consumer = kafka.consumer({ groupId: "my-app3" });

async function main() {
  await consumer.connect();
  await consumer.subscribe({
    topic: "quickstart-events", 
    fromBeginning: true
  });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      console.log({
        offset: message.offset,
        value: message?.value?.toString(),
      });
    },
  });
}

main();
```

## Consumer Groups and Partitions

### Consumer Groups

A **consumer group** is a group of consumers that coordinate to consume messages from a Kafka topic.

**Purpose:**
- **Load Balancing**: Distribute processing load among multiple consumers
- **Fault Tolerance**: If one consumer fails, Kafka automatically redistributes partitions to remaining consumers
- **Parallel Processing**: Consumers can process different partitions in parallel, improving throughput and scalability

### Partitions

**Partitions** are subdivisions of a Kafka topic. Each partition is an ordered, immutable sequence of messages appended to by producers.

**Benefits:**
- Enable horizontal scaling
- Allow parallel processing of messages
- When a message is produced, it's assigned to a specific partition using:
  - Round-robin method
  - Hash of the message key
  - Custom partitioning strategy

**Best Practice:** Use **user ID** as the message key so all messages from the same user go to the same consumer, preventing a single user from overwhelming all partitions.

## Working with Partitions

### Create Topic with Multiple Partitions

```bash
# Create topic with 3 partitions
./kafka-topics.sh --create --topic payment-done --partitions 3 --bootstrap-server localhost:9092

# Verify partition count
./kafka-topics.sh --describe --topic payment-done --bootstrap-server localhost:9092
```

### Producer with Partitions

```typescript
async function main() {
  await producer.connect();
  await producer.send({
    topic: "payment-done",
    messages: [{
      value: "hi there",
      key: "user1"
    }]
  });
}
```

### Consumer with Partitions

```typescript
await consumer.subscribe({
  topic: "payment-done", 
  fromBeginning: true
});
```

### Monitor Consumer Groups

```bash
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-app3
```

## Partition Assignment Scenarios

### 1. Equal Number of Partitions and Consumers
- Each consumer gets exactly one partition
- Optimal load distribution

### 2. More Partitions than Consumers
- Some consumers handle multiple partitions
- Still efficient processing

### 3. More Consumers than Partitions
- Some consumers remain idle
- Inefficient resource usage

## Partitioning Strategy

### Using Message Keys

When producing messages, assign a key that uniquely identifies the event. Kafka will hash this key to determine the partition, ensuring all messages with the same key go to the same partition.

**Why use the same partition for the same user?**
If a single user generates too many notifications, this approach ensures they only affect one partition instead of choking all partitions.

### Producer with User Key (producer-user.ts)

```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "my-app",
  brokers: ["localhost:9092"]
});

const producer = kafka.producer();

async function main() {
  await producer.connect();
  await producer.send({
    topic: "payment-done",
    messages: [{
      value: "hi there",
      key: "user1"
    }]
  });
}

main();
```

### Testing Partitioning

1. Start 3 consumers: `npm run consume`
2. Run producer with user key: `npm run produce:user`
3. Notice all messages reach the same consumer (same partition)

## Key Concepts Summary

- **Horizontal Scaling**: Add more brokers to increase capacity
- **Event Streaming**: Real-time data processing and historical replay
- **Consumer Groups**: Load balancing and fault tolerance
- **Partitions**: Parallel processing and scalability
- **Message Keys**: Deterministic partition assignment
- **Offsets**: Track consumer position in topics
