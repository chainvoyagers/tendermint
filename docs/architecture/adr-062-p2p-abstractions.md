# ADR 062: P2P Architecture and Abstractions

## Changelog

- 2020-11-09: Initial version (@erikgrinaker)

## Context

In [ADR 061](adr-061-p2p-refactor-scope.md) we decided to refactor the peer-to-peer (P2P) networking stack. The first phase is to redesign and refactor the internal P2P architecture, while retaining protocol compatibility as far as possible.

## Alternative Approaches

Several variations of the proposed design were considered, including e.g. calling interface methods instead of passing messages (like the current architecture), merging channels with streams, exposing the internal peer data structure to reactors, being message format-agnostic via arbitrary codecs, and so on. The final design was chosen because it has very loose coupling, is simpler to reason about and more convenient to use, avoids race conditions and lock contention for internal data structures, gives reactors better control of message ordering and processing semantics, and allows for QoS scheduling and backpressure in a very natural way.

There were also proposals to use LibP2P instead of maintaining our own P2P stack, which was rejected (for now) in [ADR 061](adr-061-p2p-refactor-scope.md).

## Decision

The P2P stack will be redesigned as a message-oriented architecture, primarily relying on Go channels for communication and scheduling. It will use IO stream transports to exchange raw bytes with individual peers, bidirectional peer-addressable channels to send and receive Protobuf messages, and a router to route messages between reactors and peers. Message passing is asynchronous with at-most-once delivery.

## Detailed Design

This ADR is primarily concerned with the architecture and interfaces of the P2P stack, not implementation details. Separate ADRs may be submitted for individual components, since implementation may be non-trivial. The interfaces described here should therefore be considered a rough architecture outline, not a complete and final design.

Primary design objectives have been:

* Loose coupling between components, for a simpler, more robust, and test-friendly architecture.
* Pluggable transports (not necessarily networked).
* Better scheduling of messages, with improved prioritization, backpressure, and performance.
* Centralized peer lifecycle and connection management.
* Better peer address detection, advertisement, and exchange.
* Backwards compatibility with the current P2P network protocols.

The main abstractions in the new stack are:

* `peer`: A node in the network, uniquely identified by a `PeerID` and stored in a `peerStore`.
* `Transport`: An arbitrary mechanism to exchange raw bytes with a single peer using IO streams.
* `Channel`: A bidirectional channel to asynchronously exchange Protobuf messages with peers addressed with `PeerID`.
* `Router`: Maintains transport connections to relevant peers and routes channel messages.
* Reactor: A design pattern loosely defined as "something which listens on a channel and reacts to messages".

These abstractions and related concepts are described in detail below.

### Transports

Transports are arbitrary mechanisms for exchanging raw bytes with a peer. For example, a gRPC transport would connect to a peer over TCP/IP and send data using the gRPC protocol, while an in-memory transport might communicate with a peer running in another goroutine using internal byte buffers. Note that transports don't have a notion of a `peer` as such - instead, they communicate with arbitrary endpoint addresses, to decouple them from the rest of the P2P stack.

Transports must satisfy the following requirements:

* Be connection-oriented, and support both listening for inbound connections and making outbound connections using endpoint addresses.

* Support multiple logical IO streams within a single connection, to take full advantage of protocols with native stream support. For example, QUIC supports multiple independent streams, while HTTP/2 and MConn multiplex logical streams onto a single TCP connection.

* Provide the public key of the peer, and possibly encrypt or sign the traffic as appropriate. This should be compared with known data (e.g. the peer ID) to authenticate the peer and avoid man-in-the-middle attacks.

The initial transport implementation will be a port of the current MConn protocol currently used by Tendermint, and should be backwards-compatible at the wire level.

The `Transport` interface is:

```go
// Transport is an arbitrary mechanism for exchanging bytes with a peer.
type Transport interface {
    // Accept waits for the next inbound connection.
    Accept(context.Context) (Connection, error)

    // Dial creates an outbound connection to an endpoint.
    Dial(context.Context, Endpoint) (Connection, error)

    // Endpoints lists endpoints the transport is listening on. Any endpoint IP
    // addresses do not need to be normalized in any way (e.g. 0.0.0.0 is
    // valid), as they should be preprocessed before being advertised.
    Endpoints() []Endpoint
}
```

How the transport configures listening is transport-dependent, and not covered by the interface. This typically happens during transport construction.

#### Endpoints

`Endpoint` represents a transport endpoint. A connection always has two endpoints: one at the local node and one at the remote peer. An outbound connection to a remote endpoint is made via a `Dial()` call, and inbound connections to listening endpoints are returned via `Accept()`.

The `Endpoint` struct is:

```go
// Endpoint represents a transport connection endpoint, either local or remote.
type Endpoint struct {
    // Protocol specifies the transport protocol, used by the router to pick a
    // transport for an endpoint.
    Protocol Protocol

    // Path is an optional, arbitrary transport-specific path or identifier.
    Path string

    // IP is an IP address (v4 or v6) to connect to. If set, this defines the
    // endpoint as a networked endpoint.
    IP net.IP

    // Port is a network port (either TCP or UDP). If not set, a default port
    // may be used depending on the protocol.
    Port uint16
}

// Protocol identifies a transport protocol.
type Protocol string
```

Endpoints are arbitrary transport-specific addresses, but if they are networked they must use IP addresses, and thus rely on IP as a fundamental packet routing protocol. This enables policies for address discovery, advertisement, and exchange - for example, a private `192.168.0.0/24` IP address should only be advertised to peers on that IP network, while the public address `8.8.8.8` may be advertised to all peers. Similarly, any port numbers if given must represent TCP and/or UDP port numbers, in order to use [UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play) to autoconfigure e.g. NAT gateways.

Non-networked endpoints (without an IP address) are considered local, and will only be advertised to other peers connecting via the same protocol. For example, an in-memory transport might use `Endpoint{Protocol: "memory", Path: "foo"}` as an address for the node "foo", and this should only be advertised to other nodes using `Protocol: "memory"`.

#### Connections and Streams

A connection represents an established transport connection between two endpoints (and thus two nodes), which can be used to exchange bytes via logically distinct IO streams. Connections are set up either via `Transport.Dial()` (outbound) or `Transport.Accept()` (inbound). The caller is responsible for verifying the remote peer's public key as returned by the connection.

Data is exchanged over IO streams created with `Connection.Stream()` using an arbitrary `StreamID`. These implement the standard Go `io.Reader` and `io.Writer` interfaces to read and write bytes. Transports are free to choose how to implement such streams, e.g. by taking advantage of native stream support in the underlying protocol or through multiplexing.

`Connection` and the related `Stream` interfaces are:

```go
// Connection represents an established connection between two endpoints.
type Connection interface {
    // Stream opens a logically distinct IO stream within the connection,
    // using an arbitrary stream ID. The stream must be created before it
    // can receive data, and must be closed before the ID can be reused.
    Stream(StreamID) Stream

    // LocalEndpoint returns the local endpoint for the connection.
    LocalEndpoint() Endpoint

    // RemoteEndpoint returns the remote endpoint for the connection.
    RemoteEndpoint() Endpoint

    // PubKey returns the public key of the remote peer.
    PubKey() crypto.PubKey

    // Close closes the connection.
    Close() error
}

// StreamID is an arbitrary stream ID.
type StreamID uint8

// Stream represents a single logical IO stream within a connection.
type Stream interface {
    io.Reader // Read([]byte) (int, error)
    io.Writer // Write([]byte) (int, error)
    io.Closer // Close() error
}
```

A note on backwards-compatibility: the current MConn protocol takes whole messages expressed as byte slices and splits them up into `PacketMsg` messages, where the final packet of a message has `PacketMsg.EOF` set. In order to maintain wire-compatibility with this protocol, the MConn transport needs to be aware of message boundaries, even though it does not care what the messages actually are. One way to handle this is to break abstraction boundaries and have the transport decode the input's length-prefixed message framing and use this to determine message boundaries. Then at some point in the future when we can break protocol compatibility we either remove the MConn protocol completely or drop the `PacketMsg.EOF` field and rely on the length-prefix framing.

### Peers

Peers are remote network nodes. Each peer is identified by a unique binary `PeerID`, and has a set of `PeerAddress` addresses expressed as URLs that they can be reached via. Addresses are resolved into one or more transport endpoints, e.g. by resolving DNS hostnames into IP addresses (which should be refreshed periodically). Peers should always be expressed as address URLs, and never as endpoints which are a lower-level construct.

```go
// PeerID is a unique peer ID, generally expressed in hex form.
type PeerID []byte

// PeerAddress is a peer address URL. The User field, if set, gives the
// hex-encoded remote PeerID, which should be verified with the remote peer's
// public key as returned by the connection.
type PeerAddress url.URL

// Resolve resolves a PeerAddress into a set of Endpoints, typically by
// expanding out any DNS names given in Host to IP addresses. Fields mapping:
//
//   Scheme → Endpoint.Protocol
//   Host   → Endpoint.IP
//   Port   → Endpoint.Port
//   Path+Query+Fragment,Opaque → Endpoint.Path
//
func (a PeerAddress) Resolve(ctx context.Context) []Endpoint { return nil }
```

The P2P stack needs to track a lot of internal information about peers, such as endpoints, status, priorities, and so on. This is done done in an internal `peer` struct, which should not be exposed outside of the `p2p` package (e.g. to reactors) in order to avoid race conditions and lock contention (other packages should use `PeerID`).

The `peer` struct might look like the following, but is intentionally underspecified and will depend on implementation requirements (for example, it will almost certainly have to track statistics about connection failures and retries):

```go
// peer tracks internal status information about a peer.
type peer struct {
    ID        PeerID
    Status    PeerStatus
    Priority  PeerPriority
    Endpoints map[PeerAddress][]Endpoint // Resolved endpoints by address.
}

// PeerStatus specifies the status of a peer.
type PeerStatus string

const (
    PeerStatusNew     = "new"     // New peer which we haven't tried to contact yet.
    PeerStatusUp      = "up"      // Peer which we have an active connection to.
    PeerStatusDown    = "down"    // Peer which we're temporarily disconnected from.
    PeerStatusRemoved = "removed" // Peer which has been removed.
    PeerStatusBanned  = "banned"  // Peer which is banned for misbehavior.
)

// PeerPriority specifies peer priorities.
type PeerPriority int

const (
    PeerPriorityNormal PeerPriority = iota + 1
    PeerPriorityValidator
    PeerPriorityPersistent
)
```

Peer information is stored in a `peerStore`, which may be persisted in an underlying database. It is kept internal to avoid race conditions and tight coupling. The peer store will replace the current address book, either partially or in full.

The `peerStore` should at the very least contain basic CRUD functionality as outlined below, but will likely need additional functionality and is intentionally underspecified:

```go
// peerStore contains information about peers, possibly persisted to disk.
type peerStore struct {
    peers map[string]*peer // Entire set in memory, with PeerID.String() keys.
    db    dbm.DB           // Database for persistence, if non-nil.
}

func (p *peerStore) Delete(id PeerID) error     { return nil }
func (p *peerStore) Get(id PeerID) (peer, bool) { return peer{}, false }
func (p *peerStore) List() []peer               { return nil }
func (p *peerStore) Set(peer peer) error        { return nil }
```

### Channels

While low-level data exchange happens via transport IO streams, the high-level API is based on a bidirectional `Channel` that can send and receive Protobuf messages addressed by `PeerID`. A channel is identified by an arbitrary `ChannelID` identifier, and can exchange Protobuf messages of one specific type (since the type to unmarshal into must be known). Message delivery is asynchronous and at-most-once.

A `Channel` has this interface:

```go
// Channel is a bidirectional channel for Protobuf message exchange with peers.
type Channel struct {
    // ID contains the channel ID.
    ID ChannelID

    // messageType specifies the type of messages exchanged via the channel, and
    // is used e.g. for automatic unmarshaling.
    messageType proto.Message

    // In is a channel for receiving inbound messages. Envelope.From is always
    // set.
    In <-chan Envelope

    // Out is a channel for sending outbound messages. Envelope.To or Broadcast
    // must be set, otherwise the message is discarded.
    Out chan<- Envelope
}

// Close closes the channel, and is equivalent to close(Channel.Out). This will
// cause Channel.In to be closed when appropriate. The ID can then be reused.
func (c *Channel) Close() error { return nil }

// ChannelID is an arbitrary channel ID, and maps directly onto a stream ID.
type ChannelID StreamID

// Envelope specifies the message receiver and sender.
type Envelope struct {
    From      PeerID        // Message sender, or empty for outbound messages.
    To        PeerID        // Message receiver, or empty for inbound messages.
    Broadcast bool          // Send message to all connected peers, ignoring To.
    Message   proto.Message // Payload.
}
```

A channel can reach any connected peer, and is implemented using transport streams against each individual peers (with `StreamID` equal to `ChannelID`). The channel will automatically (un)marshal Protobuf to byte slices and use length-prefixed framing (the de facto standard for Protobuf streams) when writing them to the stream.

Message scheduling and queueing is left as an implementation detail, and can use any number of algorithms such as FIFO, round-robin, priority queues, etc. Since message delivery is not guaranteed, both inbound and outbound messages may be dropped, buffered, or blocked as appropriate.

Since a channel can only exchange messages of a single type, it is often useful to use a wrapper message type with e.g. a Protobuf `oneof` field that specifies a set of inner message types that it can contain. The channel can automatically perform this (un)wrapping if the outer message type implements the `Wrapper` interface (see Reactors for an example):

```go
// Wrapper is a Protobuf message that can contain a variety of inner messages.
// If a Channel's message type implements Wrapper, the channel will
// automatically (un)wrap passed messages using the container type, such that
// the channel can transparently support multiple message types.
type Wrapper interface {
    proto.Message

    // Wrap will take a message and wrap it in this one.
    Wrap(proto.Message) error

    // Unwrap will unwrap the inner message contained in this message.
    Unwrap() (proto.Message, error)
}
```

### Routers

The router oversees all P2P networking for a node. It is responsible for keeping track of network peers, maintaining transport connections, and routing channel messages. As such, it must do e.g. connection retries and backoff, message QoS scheduling and backpressure, peer quality assessments, and endpoint detection and advertisement. In addition, the router provides mechanisms for reactors to receive updates about peer status changes and to report peer errors (which can optionally cause the peers to be disconnected or banned).

The implementation of the router is likely to be non-trivial, and is intentionally unspecified here. A separate ADR will likely be submitted for this.

The `Router` API is as follows:

```go
// Router manages connections to peers and routes Protobuf messages between them
// and local reactors. It also provides peer status updates and error reporting.
type Router struct{
    peerStore  *peerStore
    transports map[Protocol]Transport
}

// NewRouter creates a new router, using the given peer store to track peers.
// Transports must be pre-initialized to listen on appropriate endpoints.
func NewRouter(peerStore *peerStore, transports map[Protocol]Transport) *Router { return nil }

// Channel opens a new channel with the given ID. messageType should be an empty
// Protobuf message of the type that will be passed through the channel. The
// message can implement Wrapper for automatic message (un)wrapping.
func (r *Router) Channel(id ChannelID, messageType proto.Message) (*Channel, error) { return nil, nil }

// PeerErrors returns a channel that can be used to submit peer errors,
// specifying an action to take. The sender should not close the channel.
func (r *Router) PeerErrors() chan<- PeerError { return nil }

// PeerUpdates returns a channel with peer updates. The caller must cancel the
// context to end the subscription, and keep consuming messages in a timely
// fashion until the channel is closed to avoid blocking updates.
func (r *Router) PeerUpdates(ctx context.Context) <-chan PeerUpdate { return nil }

// PeerErrors is a channel for submitting peer errors.
type PeerErrors chan<- PeerError

// PeerError is a peer error reported by a reactor, and an action to take.
type PeerError struct {
    ID     PeerID
    Err    error
    Action PeerAction
}

func (e PeerError) Error() string { return "" }

// PeerAction is an action to take for a peer error.
type PeerAction string

const (
    PeerActionNone       PeerAction = "none"
    PeerActionDisconnect PeerAction = "disconnect"
    PeerActionBan        PeerAction = "ban"
)

// PeerUpdates is a channel for receiving peer updates.
type PeerUpdates <-chan PeerUpdate

// PeerUpdate is a peer status update for reactors.
type PeerUpdate struct {
    ID     PeerID
    Status PeerStatus
}
```

### Reactor Example

While reactors are a first-class concept in the current P2P stack (i.e. there is an explicit `p2p.Reactor` interface), they will simply be a design pattern in the new stack, loosely defined as "something which listens on a channel and reacts to messages".

Below is an example of a simple echo reactor implemented as a function. The reactor will exchange the following Protobuf messages:

```protobuf
message EchoMessage {
    oneof inner {
        PingMessage ping = 1;
        PongMessage pong = 2;
    }
}

message PingMessage {
    string content = 1;
}

message PongMessage {
    string content = 1;
}
```

If we implement the `Wrapper` interface for `EchoMessage`, we can transparently pass `PingMessage` and `PongMessage` through the channel and it will automatically be (un)wrapped in an `EchoMessage`:

```go
func (m *EchoMessage) Wrap(inner proto.Message) error {
    switch inner := inner.(type) {
    case *PingMessage:
        m.Inner = &EchoMessage_PingMessage{PingMessage: inner}
    case *PongMessage:
        m.Inner = &EchoMessage_PongMessage{PongMessage: inner}
    default:
        return fmt.Errorf("unknown message %T", inner)
    }
    return nil
}

func (m *EchoMessage) Unwrap() (proto.Message, error) {
    switch inner := m.Inner.(type) {
    case *EchoMessage_PingMessage:
        return inner.PingMessage, nil
    case *EchoMessage_PongMessage:
        return inner.PongMessage, nil
    default:
        return nil, fmt.Errorf("unknown message %T", inner)
    }
}
```

The reactor itself would be implemented e.g. like this:

```go
// RunEchoReactor wires up a reactor to a router and runs it.
func RunEchoReactor(router *p2p.Router) error {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    channel, err := router.Channel(1, &EchoMessage{})
    if err != nil {
        return err
    }
    defer channel.Close()

    return EchoReactor(ctx, channel, router.PeerUpdates(ctx), router.PeerErrors())
}

// EchoReactor provides an echo service, pinging all known peers until cancelled.
func EchoReactor(
    ctx context.Context,
    channel *p2p.Channel,
    peerUpdates p2p.PeerUpdates,
    peerErrors p2p.PeerErrors,
) error {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        // Send ping message to all known peers every 5 seconds.
        case <-ticker.C:
            channel.Out <- Envelope{
                Broadcast: true,
                Message:   &PingMessage{Content: "👋"},
            }

        // When we receive a message from a peer, either respond to ping, output
        // pong, or disconnect peer on unknown message type.
        case envelope := <-channel.In:
            switch msg := envelope.Message.(type) {
            case *PingMessage:
                channel.Out <- Envelope{
                    To:      envelope.From,
                    Message: &PongMessage{Content: msg.Content},
                }

            case *PongMessage:
                fmt.Printf("%q replied with %q\n", envelope.From, msg.Content)

            default:
                peerErrors <- PeerError{
                    ID:     envelope.From,
                    Err:    fmt.Errorf("unexpected message %T", msg),
                    Action: PeerActionDisconnect,
                }
            }

        // Output info about any peer status changes.
        case peerUpdate := <-peerUpdates:
            fmt.Printf("Peer %q changed status to %q", peerUpdate.ID, peerUpdate.Status)

        // Exit when context is cancelled.
        case <-ctx.Done():
            return nil
        }
    }
}
```

### Implementation Plan

The existing P2P stack should be gradually migrated towards this design. The simplest order would likely be:

1. Port the `privval` package to no longer use `SecretConnection` (e.g. by using gRPC instead), or temporarily duplicate its functionality.

2. Migrate the current `p2p` connection and transport code to use the new `Transport` API, and migrate existing code to use it instead.

3. Implement the `Channel`, `PeerUpdates`, and `PeerErrors` APIs as shims on top of the current `Switch` and `Peer` APIs, and rewrite all reactors to use them instead.

4. Implement the new `peer` and `peerStore` APIs, and either make the current address book a shim on top of these or replace it.

5. Replace the existing `Switch` abstraction with the new `Router`.

6. Consider rewriting and/or cleaning up reactors and other P2P-related code to make better use of the new abstractions.

## Status

Proposed

## Consequences

### Positive

* Reduced coupling and simplified interfaces should lead to better understandability, increased reliability, and more testing.

* Using message passing via Go channels gives better control of backpressure and quality-of-service scheduling.

* Peer lifecycle and connection management is centralized in a single entity, making it easier to reason about.

* Detection, advertisement, and exchange of node addresses will be improved.

* Additional transports (e.g. QUIC) can be implemented and used in parallel with the existing MConn protocol.

* The P2P protocol will not be broken in the initial version, if possible.

### Negative

* Fully implementing the new design as indended is likely to require breaking changes to the P2P protocol at some point, although the initial implementation shouldn't.

* Gradually migrating the existing stack and maintaining backwards-compatibility will be more labor-intensive than simply replacing the entire stack.

* A complete overhaul of P2P internals is likely to cause temporary performance regressions and bugs as the implementation matures.

* Hiding peer management information inside the `p2p` package may prevent certain functionality or require additional deliberate interfaces for information exchange, as a tradeoff to simplify the design, reduce coupling, and avoid race conditions and lock contention.

### Neutral

* Implementation details around e.g. peer management, message scheduling, and peer and endpoint advertisement are not yet determined.

## References

* [ADR 061: P2P Refactor Scope](adr-061-p2p-refactor-scope.md)
* [#2067: P2P Refactor](https://github.com/tendermint/tendermint/issues/2067)