# Message Server

The message server is in charge of message propagation through the network and between the clients and the backend.

## Cluster Identification

Currently the message server system will identify itself as being a part of a cluster by using the application root
package name and version. However in environment where multiple instances of the same application and version are run,
there will be conflicts with the messaging system. (e.g. inside single box which houses multiple test environments).

To prevent pollution and contamination of messages, the `server.serviceName` configuration entry needs to be set, to
give each environment a unique identifier.

Example:

```yaml
server:
    serviceName: applicationName-environmentID
```

## Subsystems

The systems that make the message server and their configuration are described below.

### MMRP

The message server uses the MMRP library to ensure communication between the different MAGE instances on the network.
In order to find these instances, you need to have [service discovery](#service-discovery) set up.

To use MMRP, please configure it to bind to a particular port and pick a network on which it is expected to communicate.

Example:

```yaml
server:
    mmrp:
        bind:
            host: "*"  # asterisk for 0.0.0.0, or a valid IP address
            port: "*"  # asterisk for any port, or a valid integer
        network:
            - "192.168.2"  # formatted according to https://www.npmjs.com/package/netmask
```

For more information, please read the [MMRP documentation](#mmrp).


### Message stream

The messages sent by the server inevitably make their way to a client through a message stream.

For more information, please read the [Message stream documentation](#message-stream).


## Sending messages

It is recommended to use the [State class](../state/Readme.md) to send and broadcast messages to users, but if you want
more control over or a better understanding about how messages are sent, please keep on reading.

Keep in mind that the system makes no guarantees that the message will actually be delivered, although as long as the
address and clusterId you pass are valid, and that user remains logged in, the message should be received.

### Sending a single message

You can send a message directly to a user using the following call.

```javascript
mage.core.msgServer.send(address, clusterId, message);
```

Where:

- `address` is the recipient's session key.
- `clusterId` refers to the physical location in the network where that session's messages are being stored for delivery.
- `message` is a string or Buffer containing valid JSON, or an array of strings/Buffers if you want to send a batch.


### Broadcasting a message to all logged-in users

You can send a message to all logged-in users, by using the following call.

```javascript
mage.core.msgServer.broadcast(message);
```

Where:

- `message` is a string or Buffer, or when you want to send multiple messages, an array of strings/Buffers.


## Receiving messages on a web client

The MAGE client will receive messages and emit them as events. You can listen for them through `mage.eventManager`.

```javascript
mage.eventManager.on('event.name', function (message) {
  /* use the message */
});
```

For now, the only way to receive these messages is to make sure messages are sent or broadcast with the following format:

```json
[
  ["event.name", {"key":5,"another":"value"}],
  ["another.event", {"etc":false}],
]
```

## Testing your application with AerisCloud and Marathon

When using AerisCloud to test your application across a cluster of servers using the Marathon service, please make sure
to include the following configuration in `config/marathon.yaml`. Over time, the addresses below may change, so always
stay in touch with your friendly local system administrator to make sure these make sense.

```yaml
server:
    clientHost:
        expose: null
        bind:
            port: PORT0
    serviceDiscovery:
        engine: "zookeeper"
        options:
            hosts: "192.168.2.21:2181"
    mmrp:
        bind:
            port: PORT1
        network:
            - "192.168.2"
```

Your Makefile needs to replace the magical constants PORT0 and PORT1 in this file when running. Here's an example of
the marathon Make target:

```make
.PHONY: marathon

marathon:
	source /opt/nvm/nvm.sh && \
	sed -i "s/PORT0/${PORT0}/" config/marathon.yaml && \
	sed -i "s/PORT1/${PORT1}/" config/marathon.yaml && \
	sed -i "s/PORT/${PORT}/" config/marathon.yaml && \
	nvm use 0.10.33 && \
	NODE_ENV=production,marathon HOME=${MESOS_DIRECTORY} make deps && \
	NODE_ENV=production,marathon node .
```

When pushing the application to Marathon, you need to select ports for PORT0 and PORT1. An example:

```shell
aeriscloud marathon office/jp push -p 80 16001
```

For more information about using Marathon, please read the [documentation](https://github.com/Wizcorp/AerisCloud/blob/master/docs/walkthrough/marathon.md)