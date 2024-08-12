# Message

It is the basic unit, it is what producers publish to topics and what consumers then consume from topics. It includes metadata about the message as well as the actual payload.

## Sent message - by Producer

The user is able to send through producer the following data:

### payload (bytes)

* `Description`: The actual payload of the message. This field contains the data that the consumer will process.
* `Usage`: This is the core content of the message that is intended to be consumed by the receiving party.

### attributes (map(string, string) )

* `Description`: A map of user-defined properties or attributes associated with the message. This can include custom metadata or tags.
* `Usage`: This field provides additional flexibility for including extra context or metadata that might be required for specific processing or filtering needs.

## Received message - by Consumer

### payload ( bytes)

* `Description`: The actual payload of the message. This field contains the data that the consumer will process.
* `Usage`: This is the core content of the message that is intended to be consumed by the receiving party.

### metadata (MessageMetadata)

* `Description`: Contains additional metadata related to the message. This can include information about the producer and the message itself.
* `Usage`: This field provides context and additional information that can be used for processing, tracking, and analyzing messages.

#### MessageMetadata

The `MessageMetadata` message includes supplementary information about the message itself, which is useful for managing message processing and analytics. It contain the following fields:

**producer_name (string)**:

* `Description`: The name of the producer that sent the message. This can be used for identifying the source of the message.
* `Usage`: This field is optional and can be used for logging, tracing, or any other purposes where the producer's identity is needed.

**sequence_id (uint64)**:

* `Description`: Represents the sequence ID of the message within the topic. It helps in maintaining the order of messages.
* `Usage`: This field is crucial for ordering messages correctly when received by consumers. It helps in ensuring that messages are processed in the correct sequence.

**publish_time (uint64)**:

* `Description`: Indicates the time when the message was published. It is typically a timestamp value.
* `Usage`: This field can be used for time-based processing, such as determining the age of a message or for scheduling purposes.

**attributes (map(string, string))**:

* `Description`: A map of user-defined properties or attributes associated with the message. This can include custom metadata or tags.
* `Usage`: This field provides additional flexibility for including extra context or metadata that might be required for specific processing or filtering needs.
