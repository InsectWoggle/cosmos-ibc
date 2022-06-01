# Router Specification

### Synopsis

The following document specifies the interfaces and state machine logic that IBC implementations must implement if they wish to serve as a Router chain.

### Motivation

// TODO

### Desired Properties

// TODO

## Technical Specification

### Data Structures

### Store Paths

### Channel Handshake

```typescript
// srcConnectionHops is the list of identifiers routing back to the initializing chain
// destConnectionHops is the list of identifiers routing to the TRY chain
// NOTE: since srcConnectionHops is routing in the opposite direction, it will contain all the counterparty connection identifiers from the connection identifiers specified by the initializing chain up to this point.
// For example, if the route specified by the initializing chain is "connection-1/connection-3"
// Then `routeChanOpenTry` may be called on the router chain with srcConnectionHops: "connection-4", destConnectionHops: "connection-3"
// where connection-4 is the counterparty connectionID on the router chain to connection-1 on the initializing chain
// and connection-3 is the connection on the router chain to the next hop in the route which in this case is the TRY chain.
function routeChanOpenTry(
    srcConnectionHops: [Identifier],
    destConnectionHops: [Identifier],
    provingConnectionIdentifier: Identifier,
    counterpartyPortIdentifier: Identifier,
    counterpartyChannelIdentifier: Identifier,
    initChannel: ChannelEnd,
    proofInit: CommitmentProof,
    proofHeight: Height
) {
    abortTransactionUnless(len(srcConnectionHops) != 0)
    abortTransactionUnless(len(destConnectionHops) != 0)
    route = join(append(srcConnectionHops, destConnectionHops...), "/")
    included = false
    for ch in initChannel.connectionHops {
        if route == ch {
            included = true
        }
    }
    abortTransactionUnless(included)

    connection = getConnection(provingConnectionIdentifier)

    // verify that proving connection is counterparty of the last src connection hop
    abortTransactionUnless(srcConnectionHops[len(srcConnectionHops-1)] == connection.counterpartyConnectionIdentifier)

    if srcConnectionHops > 1 {
        // prove that previous hop stored channel under channel path and prefixed by srcConnectionHops[0:len(srcConnectionHops)-1]
        path = append(srcConnectionHops[0:len(srcConnectionHops)-1], channelPath(portIdentifier, channelIdentifier))
        client = queryClient(connection.clientIdentifier)
        value = protobuf.marshal(initChannel)
        verifyMembership(clientState, proofHeight, 0, 0, proofInit, path, value)
    } else {
        // prove that previous hop (original source) stored channel under channel path
        verifyChannelState(connection, proofHeight, proofInit, counterpartyPortIdentifier, counterpartyChannelIdentifier, initChannel)
    }

    // store channel under channelPath prefixed by srcConnectionHops
    path = append(srcConnectionHops, channelPath(counterpartyPortIdentifier, counterpartyChannelIdentifier))
    store.set(path, initChannel)

    // store srcConnectionHops -> src identifiers
    store(srcConnectionHops, counterpartyPortIdentifier, counterpartyChannelIdentifier)
}

// srcConnectionHops is the connection Hops from the TRY chain to the executing router chain
// destConnectionHops is the connection hops from the executing router chain to the ACK chain.
function routeChanOpenAck(
  srcConnectionHops: [Identifier],
  destConnectionHops: [Identifier],
  provingConnectionIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  tryChannel: ChannelEnd,
  proofTry: CommitmentProof,
  proofHeight: Height) {
    abortTransactionUnless(len(srcConnectionHops) != 0)
    abortTransactionUnless(len(destConnectionHops) != 0)

    connection = getConnection(provingConnectionIdentifier)
    
    // verify that proving connection is counterparty of the last src connection hop
    abortTransactionUnless(srcConnectionHops[len(srcConnectionHops-1)] == connection.counterpartyConnectionIdentifier)

    if srcConnectionHops > 1 {
        // prove that previous hop stored channel under channel path and prefixed by srcConnectionHops[0:len(srcConnectionHops)-1]
        path = append(srcConnectionHops[0:len(srcConnectionHops)-1], channelPath(portIdentifier, channelIdentifier))
        client = queryClient(connection.clientIdentifier)
        value = protobuf.marshal(tryChannel)
        verifyMembership(clientState, proofHeight, 0, 0, proofTry, path, value)
    } else {
        // prove that previous hop (original source) stored channel under channel path
        verifyChannelState(connection, proofHeight, proofTry, counterpartyPortIdentifier, counterpartyChannelIdentifier, tryChannel)
    }

    // store channel under channelPath prefixed by srcConnectionHops
    path = append(srcConnectionHops, channelPath(counterpartyPortIdentifier, counterpartyChannelIdentifier))
    store.set(path, tryChannel)

    // store srcConnectionHops -> src identifiers
    store(srcConnectionHops, counterpartyPortIdentifier, counterpartyChannelIdentifier)
}

function routeChanOpenConfirm(
    srcConnectionHops: [Identifier],
    destConnectionHops: [Identifier],
    provingConnectionIdentifier: Identifier,
    portIdentifier: Identifier,
    channelIdentifier: Identifier,
    ackChannel: ChannelEnd,
    proofAck: CommitmentProof,
    proofHeight: Height
) {
    abortTransactionUnless(len(srcConnectionHops) != 0)
    abortTransactionUnless(len(destConnectionHops) != 0)

    connection = getConnection(provingConnectionIdentifier)

    // verify that proving connection is counterparty of the last src connection hop
    abortTransactionUnless(srcConnectionHops[len(srcConnectionHops-1)] == connection.counterpartyConnectionIdentifier)

    if srcConnectionHops > 1 {
        // prove that previous hop stored channel under channel path and prefixed by srcConnectionHops[0:len(srcConnectionHops)-1]
        path = append(srcConnectionHops[0:len(srcConnectionHops)-1], channelPath(portIdentifier, channelIdentifier))
        client = queryClient(connection.clientIdentifier)
        value = protobuf.marshal(tryChannel)
        verifyMembership(clientState, proofHeight, 0, 0, proofTry, path, value)
    } else {
        // prove that previous hop (original src) stored channel under channel path
        verifyChannelState(connection, proofHeight, proofTry, counterpartyPortIdentifier, counterpartyChannelIdentifier, tryChannel)
    }


}
```

### Packet Handling

```typescript
// if path is prefixed by portId and channelId and it is stored by last
// connection in srcConnectionHops then store under same path prefixed by srcConnectionHops
function routeChannelData(
    srcHops: [Identifier],
    portIdentifier: Identifier,
    channelIdentifier: Identifier,
    path: CommitmentPath
)
