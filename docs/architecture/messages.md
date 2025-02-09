# Message Structure in Danube

A message in Danube represents the fundamental unit of data transmission between producers and consumers. Each message contains both the payload and associated metadata for proper routing and processing.

## StreamMessage Structure

### Core Fields

* request_id (u64): Unique identifier for tracking the message request
* msg_id (MessageID): Complex identifier containing routing and location information
* payload (Vec): The actual message content
* publish_time (u64): Timestamp when the message was published
* producer_name (String): Name of the producer that sent the message
* subscription_name (String): Name of the subscription for consumer acknowledgment routing
* attributes (HashMap<String, String>): User-defined key-value pairs for custom metadata

### MessageID Fields

* producer_id (u64): Unique identifier for the producer within a topic
* topic_name (String): Name of the topic the message belongs to
* broker_addr (String): Address of the broker handling the message
* segment_id (u64): Unique identifier for the topic segment
* segment_offset (u64): Message position within the segment

### Usage

#### Producer Perspective

Producers create messages by setting the payload and optional attributes. The system automatically generates and manages other fields like request_id, msg_id, and publish_time.

#### Consumer Perspective

Consumers receive the complete StreamMessage structure, providing access to both the message payload and all associated metadata for processing and acknowledgment handling.

#### Message Routing

The MessageID structure enables efficient message routing and acknowledgment handling across the Danube messaging system, ensuring messages reach their intended destinations and can be properly tracked.
