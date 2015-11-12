MicroMsg2
=====================

Micromessage2 is an application layer network protocol which intention is to be lightweight but flexible. Reliable means that it can make sure that all messages are delivered (if you want to). Flexible means that it supports extensions out of the box (using `EXTENSION` frames).

The protocol follows semantic versioning which means that client/servers that speak the same major version should be able to communicate, even if the minor version is not matching. But with reduced functionality.

## Client/Server architecture

This library is based on the assumption that the one connecting acts as a client and the one accepting a connection acts as a server. However, both sides may receive connections. Hence the term server/client is only applied for the current connection and not the applications.

The `MESSAGE` frames must only be sent from the server to the client.

## Reliable

The protocol supports reliable messaging. The model is based on message ACKs instead of transactions. ACKs are disabled by default and must be activated in the handshake negotiation.

ACKs can either be accumulate (`batch-ack` extension) or one ack per message (`ack` extension). The sender MUST NOT send any more messages until the pre-configured number of messages have been ACKed.

For instance, to activate `batch-ack` the client might send `batch-ack:count=10;json;dotnet` as it's required extensions. That means that the sender must stop sending messages when there are 10 pending messages (i.e. an ack not have been sent by the receiver for those messages).

## Extensions

Functionality is added by requiring extensions in the handshake. As mentioned above, ACKs is an extension. pub/sub too.

Extensions may or may not provide additional frames. For instance the "json" extension only defines that MESSAGE/SEND payloads should be serialized as JSON while the "dotnet" extension requires that an extension frame is sent before each message.

Official extensions are only introduced when a new minor version are released. Vendor specific extensions may exist, they must however start with `x-` as prefix.

# Handshake

The purpose of the handshake is to make sure that the client and server are compatible. The handshake should be initiated by the client and can be considered to be a negotiation.

If versions differ the lowest version should be used. If the client or the server doesn't speak the lower version it should send an `ERROR` frame and then disconnect.

Both the server and the client sends the extensions they require to work properly. If the server doesn't implement the extensions requested by the client it must send and `ERROR` frame and disconnect directly after. If the client doesn't implement the extensions requested by the server it must send and `ERROR` frame and disconnect directly after. 

1. Client: I require A, B and what I suggest X, Y
2. Server: I support A, B, but I also require C. I suggest Y like you but also Z.
3. Client: I support C and we'll use the optional Y and Z.

## Extension negotiation

The `RequiredExtensions` and `OptionalExtensions` fields are used to determine which extensions should be used for this connection.

Each extension can contain a name and properties (key/value). These fields must follow the following restrictions.

* Name: FIELD
* Property names: FIELD
* Property values: Values must be URL ENCODED.

Each extension must be followed by a semicolon (except the last one).

Properties are not required.

### Example format

    //with two properties
    extensionName1:propName1=propValue2,propName2=Hello%20%25%20mother%20fucker!;

	// just extension names
	json;xml;protobuf

	//one extension with property, the rest is only definitions
	batch-ack:max-count=10;json;microauth

## Client request

A handshake from the client looks like this:

Name | Number of octets | Description
-----|------------------|----------------
Major Version | 1 | Should be 1.
Minor version | 1 | Should be 0.
Flags | 1 | A set of flags that can indicate certain features (reserved)
Identity length | 1 | Length of the identity field
Identity | * | ASCII string used to allow the server to identity this connection. An opaque string.
RequiredExtensionsLength | 2 | Length of the next string field (unsigned short)
RequiredExtensions | * | Semi-colon separated string identifying all extensions that the server must support. Extensions are described later in this chapter.
OptionalExtensionsLength | 2 | Length of the next string field (unsigned short)
OptionalExtensions | * | A list of extensions that the client would like to activate.

The `RequiredExtensionsLength` and `OptionalExtensionsLength` can be zero, which means that the string field is not present. Continue with the next length header.

### Byte per byte example

The following example doesn't require any optional extensions.

    Version  Flags  Idlen   ID        RequiredLen  Required                 OptionalLen
    1 0      0      4       h u l k   0 10           j s o n ; d o t n e t  0 0


## Server reply

Name | Number of octets | Description
-----|------------------|---------------
Major Version | 1 | Should be 1.
Minor version | 1 | Should be 0.
Flags | 1 | A set of flags that can indicate certain features (reserved)
Identity length | 1 | Length of the identity field
Identity | * | ASCII string used to allow the client to identity this connection. An opaque string. Typically server name and version.
RequiredExtensionsLength | 2 | Length of the next string field (unsigned short)
RequiredExtensions | * | Semi-colon separated string identifying all extensions that the client must support (i.e. requested by the server). Extensions are described later in this chapter.
OptionalExtensionsLength | 2 | Length of the next string field (unsigned short)
OptionalExtensions | * | A list of extensions that the client would like to activate.

The `RequiredExtensionsLength` and `OptionalExtensionsLength` can be zero, which means that the string field is not present. Continue with the next length header.

### Byte per byte example

The following handshake suggests that acks should be used.

    Version  Flags  Idlen   ID          RequiredLen  OptionalLen Optional
    1 0      0      5       s r v e r   0 0          0 9         b a t c h - a c k


## End of handshake

Finally the client must send it's message again, but with all required and optional frames that should be used.

If the client do not support the extensions required by the server it MUST send an `ERROR` frame and disconnect.

### Negotiation of optional frames

(Part of each handshake)

    // the client want to use any of the specified serialization extensions
    Client: json;xml;protobuf;
 
    // the server wants to use json and also some sort of acks
    Server: json;ack;batch-ack

    // the client do not want to use acks
    Client: json

As none sent an error frame nor disconnected the handshake is now complete. The client can now send command frame like `SUBSCRIBE` or use an extension frame.

## Handshake completed

Once the handshake is completed the extensions are indexed using the order in the final handshake message.

That means that the following order are used for the examples above (one-based index):

1. json
2. dotnet
3. batch-ack

The index is used in the `EXTENSION` frames to specify which extension the frame is for.

### Complete handshake example

	//Client -> server
    Version  Flags  Idlen   ID        RequiredLen  Required                 OptionalLen
    1 0      0      4       h u l k   0 10           j s o n ; d o t n e t  0 0

	// server -> client
    Version  Flags  Idlen   ID          RequiredLen  OptionalLen Optional
    1 0      0      5       s r v e r   0 0          0 9         b a t c h - a c k

	// client -> server
    Version  Flags  Idlen   ID        RequiredLen    Required               OptionalLen Optional
    1 0      0      4       h u l k   0 10           j s o n ; d o t n e t  0 9         b a t c h - a c k

# Frame prefix

All frames are prefixed with a standard header. The header specifies how the rest of the frame will look like.

Name | Number of octets | Description
-----|------------------|---------------
Flags | 1 | Controls the behavior of this frame.

**Flag values**

The flags value is contains bit flags. The default type of frame is MESSAGE/SEND (depending on the direction) unless flag 1, 2 or 4 is specified.

* 1 = COMMAND frame: The rest of the frame follows the COMMAND definition
* 2 = EXTENSION frame: The rest of the frame follows the EXTENSION definition
* 4 = ERROR frame:
* 8 = LARGE_PAYLOAD. The length header will be 4 bytes instead of 1.
* 16 = Continued: There are more frames for the same payload.

# MESSAGE frame

Message frames can be sent both by the client and the server. However, they are always pushed and not pulled. It basically means that both end points should either use asynchronous reads or have a dedicated thread for the reading.

The amount of messages that are being pushed can be controlled by the `ack` or `batch-ack` extensions. The both add reliability where the former limits the amount of messages being pushed to one (until an ACK have been sent) while the latter can send a configurable amount of messages before waiting on an ack.

* If no ACK is used the pusher will push messages as soon as they are arrived. 
* If a single-message ack is used the pusher will only push a new message once the previous one have been ACKed
* If batch-acks are used the pusher will push messages continuously until the `batch-max` amount of messages have been sent without receiving an ACK from the client.


Name | Number of octets | Description
-----|------------------|---------------
Sequence number | 2 | Sequence number which wraps once reaching 65535 (unsigned short)
DestinationLength | 1 | Length of the next field
Destination | * | Destination, for instance a queue name "commands" or a topic "topic/domain-events"
PropertiesLength | 2 | Length of the next field (unsigned short)
Properties | * | Properties that can be used in filtering. Key/Value pairs. Key = FIELD, Value = FILTER_VALUE (see below)
Payload length | 1 or 4 | 1 byte unless LARGE_PAYLOAD is set in the frame flags
Payload | * | Actual message. Can contain anything (binary/text/json/xml etc etc)

Multiple frames should be used if the payload in a MESSAGE frame is larger than 1 MiB data. The bit "CONTINUED" should be set in all but the last frame to be able to receive everything.

Properties can be used to attach data to a frame. For message queuing they are typically used to provide information that the receivers can use to filter messages. that allows the server to apply filters without having to know what the message content is.

The format of the destination is specified by the extension that defines it.

## Example properties:

    first_name:jonas;twitter:jgauffin;


## COMMAND frame

Commands are used to define how the connection should be used. It can for instance be a SUBSCRIBE command (pub/sub extension).

Name | Number of octets | Description
-----|------------------|---------------
Payload length | 1 or 4 | 1 byte unless bit 8 is set in the flags
payload | * | data related to the extension

The payload for commands are a NAME and key/value pairs.

    CommandName;key=value,key=value,key=value

Restrictions:

* Command Name: FIELD
* Key: FIELD
* Value: Values must be URL ENCODED.


## EXTENSION frame

Extension frames have their own layout. They do however share a common header before the actual content. The payload length is required so that proxies/brokers can continue to function (by forwarding messages) without knowing the details of the EXTENSION frames. 


Name | Number of octets | Description
-----|------------------|---------------
ID | 1 | Extension ID from the handshake
Payload length | 1 or 4 | 1 byte unless bit 8 is set in the flags
payload | * | data related to the extension

The batch-ack extension only carries an int which identifies the latest message to identify. Hence an entire frame for that extension would look like:

    Flags Id Length SequenceNumber
    4     2  2      0 4

The length is four since the sequence number (`short`) consists of two bytes.

i.e. it ACKs the messages with sequence numbers 1, 2, 3, 4

## batch-ack

Batch acknowledgements are used to allow the server to push multiple messages to the client before waiting for an ACK. 

The amount of messages to send is specified in the handshake (as defined below).

The server must not send any more messages until the agreed amount of messages have been ACKed. The client can however send and ACK before the agreed number have been received. That means that the server should simply reset the counter.

### Handshake

The number of messages can be negotiated in the handshake. It's optional and the default count is 10 if nothing else have been specified.

**Example**

    //with count specified
    batch-ack:max-count:100;

	//without count
	batch-ack;
	
### Extension frame

The extension frame contains the sequence number to ack. The number specifies that all MESSAGE frames with a sequence number up to the specified one have been ACK:ed.

Name | Number of octets | Description
-----|------------------|---------------
SequenceNumber | 2 | Accumulate sequence number to ack (unsigned short)

Do note that the side that receives the ACK must ACK all messages up to the specified one, no matter if the sequence number have been wrapped or not.

**Example**

Assume that the last ACKed message had sequence number 65533.
We have received 5 messages after that (65534, 65534, 1, 2, 3).

Now we get an ACK extension frame which contains the sequence number 3. The ack receiver must then ACK all five messages including those with higher sequence numbers (65534, 65535). A simplistic approach is to store the LAST received ack sequence number and then ACK and wrap until the specified number have been reached.

Simplistic example:

```csharp
    var numberToAck = lastReceivedNumber == 65535 ? 1 : lastReceivedNumber + 1;
    if (numberToAck == receivedNumber) 
    {
        AckMessage(numberToAck);
        return;
    }

    while (receivedNumber  != numberToAck)
    {
        AckMessage(numberToAck);
        numberToAck = numberToAck == 65535 ? 1 : numberToAck+ 1;
    }
```

## DOTNET

The .NET frame is used to carry the type of the serialized message in the next `MESSAGE` frame. 

The client SHOULD use the type specified for the last MESSAGE frame if none was received for the current MESSAGE frame.

Name | Number of octets | Description
-----|------------------|---------------
Type | * | Assembly qualified type name (with or without version information

# Example frame

    Flags Length  TypeName
    4     50      O n e T r u e E r r o r . C o n t r a c t s . R e p o r t s , O n e T r u e E r r o r . C l i e n t

## PUB/SUB

Publish/subscribe extension. Allows 

### Handshake negotiation

No properties are defined in the handshake. Only request the publish/subscribe extension.

### SUBSCRIBE frame

The subscribe frame is used to subscribe on message from a specific topic.

### Filters

The filter properties allows the server to filter messages for every destination (by using filters configured by each receiving clients).

**FILTER_VALUE**

Filter values can only be the following simple types. All values have a type specified prefix (one byte). All values are represented in text form and not binary form.

Type | Prefix | Example value | Property value (string) | Description
-----|--------|---------------|-------------------------|----------
Number | N | 1 | N1 | Implementations should support `long` values.
Date | D | 2014-10-13 12:00 UTC+1 | D1413198000.0 | UNIX epoch time (UTC) where fractions of a second is behind the dot.
Text | T | Hello "world" | THello%20%22world%22 | URL Encoded ASCII strings

**Filtering operators**

Filtering on values can be done with the following operators. 

Text comparison should be case sensitive. To ignore case simply use lower case for all filter values.

Descriptions are for text comparisons.

Operator | Description
---------|-----------------
= | Equals
< | For text it's "starts with"
<= | Not used for text
> | Text: "Ends with"
!= | Not equals
~ | Do not contain
& | Contains



# Definitions

* MiB: 1 mebibyte = 1,0242 bytes = 1,048,576 bytes;
* FIELD: 7-bit lower case ASCII. letters, digits and hyphen (-)
* NUMBER: Must be transferred using network byte order
* TOPIC: A queue where messages are enqueued to ***all** subscribers. There can be multiple publishers and subscribers for the same topic. The order of messages is not important.
* QUEUE: Messaging where order is important and each message is delivered to ***one*** receiver. There may be multiple consumers, but each message is delivered to the first one reading it.


# To be continued

This is work in progress. Suggestions and feedback is welcome.