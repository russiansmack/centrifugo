---
template: overrides/blog_base.html
title: An introduction to Centrifuge – real-time messaging in Go
description: Introducing Centrifuge real-time messaging library for Go language
og_title: An introduction to Centrifuge – real-time messaging in Go
og_image: https://i.imgur.com/W1PeoJL.jpg
og_image_width: 1280
og_image_height: 640
---

# An introduction to Centrifuge – real-time messaging in Go

![Centrifuge](https://i.imgur.com/W1PeoJL.jpg)

Today is 4th January – New Year holidays in progress. I am sitting inside a chair with a ticking clocks on the wall and a cup of coffee near on the table. The blue winter semidarkness outside the window makes room magnificent. I spent enough time coding last year – not sure I'll find a better time to write about the open-source library I was working on and made a big progress with.

Maybe you have already heard about it – it's called [Centrifuge](https://github.com/centrifugal/centrifuge). This is a real-time messaging library for Go language. OK, I think this is actually a mistake to call Centrifuge a library – `framework` suits better. As a Gopher I don't like frameworks a lot. As soon as you start using framework it dictates you pretty much of how things should be done. I suppose that's why I still avoid describing Centrifuge with the right word. But the code is written – so I can only move on with it :)

Centrifuge can do many things for you. Here I'll try to introduce Centrifuge and its possibilities.

This post is going to be pretty long (looks like I am a huge fan of long posts) – so make sure you also have a drink, and let's go! 

## How it's all started

I wrote several blog posts before about an original motivation of [Centrifugo](https://github.com/centrifugal/centrifugo) server.

!!!danger
    Centrifugo server is not the same as Centrifuge library for Go. It's a full-featured project built on top of Centrifuge library. Naming can be confusing but it's not too hard once you spend some time with ecosystem.

In short – Centrifugo was implemented to help traditional web frameworks dealing with many persistent connections (like WebSocket or SockJS HTTP transports). So frameworks like Django or Ruby on Rails, or frameworks from PHP world could be used on a backend but still provide real-time messaging features like chats, multiplayer browser games etc for users. With a little help from Centrifugo.

Now there are cases when Centrifugo server used in conjunction even with backend written in Go. While Go mostly has no problems dealing with many concurrent connections – Centrifugo provides some features beyond simple message passing between a client and a server. That makes it useful, especially since design is pretty non-obtrusive and fits well microservices world. Centrifugo is used in some well-known projects (like ManyChat, Yoola.io, Spot.im, Badoo etc).

In the end of 2018 I released Centrifugo v2 which is based on a real-time messaging library for Go language – Centrifuge – the subject of this post.

It was a pretty hard experience to decouple Centrifuge out of monolithic Centrifugo server – I was unable to make all the things right immediately, so Centrifuge library API went through several iterations where I introduced backwards incompatible changes. All those changes were being targeted to make Centrifuge a more generic tool and remove some opinionated or limiting parts.

## So what is Centrifuge?

This is ... well, a framework to build real-time messaging applications with Go language. I think the most popular applications these days are chats of different forms, but I want to emphasize that Centrifuge is not a framework to build chats – it's a generic instrument that can be used to create different sorts of real-time applications – real-time charts, multiplayer games.

The obvious choice for a real-time messaging transport to achieve fast and cross-platform bidirectional communication these days is WebSocket. Especially if you are targeting browser environment. You mostly don't need to use WebSocket HTTP polyfills in 2021 (though there are still corner cases so Centrifuge supports [SockJS](https://github.com/sockjs/sockjs-client) polyfill).

Centrifuge has its own custom protocol on top of plain WebSocket or SockJS frames. 

The reason why Centrifuge has its own protocol on top of underlying transport is that it provides several useful primitives to build real-time applications. The protocol [described as strict Protobuf schema](https://github.com/centrifugal/protocol/blob/master/definitions/client.proto). It's possible to pass JSON or binary Protobuf-encoded data over the wire with Centrifuge.

!!!note
    GRPC is very handy these days too (and can be used in a browser with a help of additional proxies), some developers prefer using it for real-time messaging apps – especially when one-way communication is needed. It can be a bit better from integration perspective but more resource-consuming on server side and a bit trickier to deploy.

!!!note
    Take a look at [WebTransport](https://w3c.github.io/webtransport/) – a brand-new spec for web browsers to allow fast communication between a client and a server on top of QUIC – it may be a good alternative to WebSocket in future. This in a draft status at the moment, but it's [already possible to play with in Chrome](https://centrifugal.github.io/centrifugo/blog/quic_web_transport/).

Own protocol is one of the things that prove framework status of Centrifuge. This dictates certain limits (you can't simply use alternative message encoding for example) and makes developer use custom client connectors on a front-end side to communicate with a Centrifuge-based server (see more about connectors in ecosystem part).

But protocol solves a lot of practical tasks – and here we will look at real-time features it provides for a developer.

## Run Centrifuge Node

To start working with Centrifuge you need to start Centrifuge server Node. Node is a core of Centrifuge – it has many useful methods – set event handlers, publish messages to channels etc. We will look at some events and channels concept very soon.

Also Node abstracts away scalability aspects so you don't need to think about how to scale WebSocket connections over different server instances and still have a way to deliver published messages to interested clients.

For now let's start a single instance of Node that will serve connections for us:

```go
node, err := centrifuge.New(centrifuge.DefaultConfig)
if err != nil {
    log.Fatal(err)
}

if err := node.Run(); err != nil {
    log.Fatal(err)
}
```

It's also required to serve a WebSocket handler – this is possible just by registering `centrifuge.WebsocketHandler` in HTTP mux, the process should be pretty familiar to you:

```go
wsHandler := centrifuge.NewWebsocketHandler(node, centrifuge.WebsocketConfig{})
http.Handle("/connection/websocket", wsHandler)
```

Now it's possible to connect to a server (using Centrifuge connector for a browser called `centrifuge-js`):

```javascript
const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket');
centrifuge.connect();
```

But connection will be rejected since we also need to provide authentication details.

## Authentication

Let's look at how Centrifuge gets a knowledge about connected user identity.

There are two main ways to authenticate client connection in Centrifuge.

The first one is over native middleware mechanism. It's possible to wrap `centrifuge.WebsocketHandler` or `centrifuge.SockjsHandler` with middleware that checks user authentication and tells Centrifuge current user ID over `context.Context`:

```go
func auth(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		cred := &centrifuge.Credentials{
			UserID: "42",
		}
		newCtx := centrifuge.SetCredentials(r.Context(), cred)
		r = r.WithContext(newCtx)
		h.ServeHTTP(w, r)
	})
}
```

So WebsocketHandler can be registered this way (note that a handler now wrapped by auth middleware):

```go
wsHandler := centrifuge.NewWebsocketHandler(node, centrifuge.WebsocketConfig{})
http.Handle("/connection/websocket", auth(wsHandler))
```

Another authentication way is a bit more generic – developers can authenticate connection based on custom token sent from a client inside first WebSocket/SockJS frame. This is called `connect` frame in terms of Centrifuge protocol. Any string token can be set – this opens a way to use JWT, Paceto, and any other kind of authentication tokens. For example [see an authenticaton with JWT](https://github.com/centrifugal/centrifuge/tree/master/_examples/jwt_token).

!!!note
    BTW it's also possible to pass any information from client side with a first connect message from client to server and return custom information about server state to a client. But this is out of post scope.

Nothing prevents you to [integrate Centrifuge with OAuth2](https://github.com/centrifugal/centrifuge/tree/master/_examples/chat_oauth2) or another framework session mechanism – [like Gin for example](https://github.com/centrifugal/centrifuge/tree/master/_examples/chat_oauth2).

## Channel subscriptions

As soon as client connected and successfully authenticated it can subscribe to channels. Channel (room or topic in other systems) is a lighweight and ephemeral entity in Centrifuge. Channel can have different features – but we will look at some channel features below. Channels created automatically as soon as first subscriber joins and destroyed as soon as last subscriber left.

The application can have many real-time features – even on one app screen. So sometimes client subscribes to several different channels – each related to a specific real-time feature (for example one channel for chat updates, one channel likes notification stream etc).

Channel is just an ascii string. Developer is responsible to find the best channel naming convention suitable for an application. Channel naming convention is an important aspect since in many cases developers want to authorize subscription to a channel on server side – so only authorized users could listen to specific channel updates.

Let's look at basic subscription example on client side:

```javascript
centrifuge.subscribe('example', function(msgCtx) {
    console.log(msgCtx)
})
```

And on server side you need to define subscribe event handler. If subscribe event handler not set then connection won't be able to subscribe to channels at all. Subscribe handler is where developer may want to check permissions of current connection to read channel updates. Here is a basic example of subscribe handler that just allows subscriptions to channel `example` for all authenticated connections and reject subscriptions to all other channels:

```go
node.OnConnect(func(client *centrifuge.Client) {
    client.OnSubscribe(func(e centrifuge.SubscribeEvent, cb centrifuge.SubscribeCallback) {
        log.Printf("client subscribes on channel %s", e.Channel)
        if e.Channel != "example" {
            cb(centrifuge.SubscribeReply{}, centrifuge.ErrorPermissionDenied)
            return
        }
        cb(centrifuge.SubscribeReply{}, nil)
    })
})
```

You may already noticed callback style of reacting to connection related things. While not being very idiomatic for Go it's very practical actually. The reason why we use callback style inside client event handlers is that it gives a developer possibility to control operation concurrency (i.e. process sth in separate goroutines or goroutine pool) and still control order of events. See [an example](https://github.com/centrifugal/centrifuge/tree/master/_examples/concurrency) that demonstrates concurrency control in action.

Now if some event published to a channel:

```go
// Here is how we can publish data to a channel.
node.Publish("example", []byte(`{"input": "hello"}`))
```

– data will be delivered to a subscribed client, and message will be printed to Javascript console. PUB/SUB in its usual form.

!!!note
    Though Centrifuge protocol based on Protobuf schema in example above we published a JSON message into a channel. By default, we can only send JSON to connections since default protocol format is JSON. But we can switch to Protobuf-based binary protocol by connecting to `ws://localhost:8000/connection/websocket?format=protobuf` endpoint – then it's possible to send binary data to clients.

## Asynchronous message passing

While Centrifuge mostly shines when you need channel semantics it's also possible to send any data to connection directly – to achieve bidirectional communication, just what a native WebSocket provides.

To send a message to a server one can use `send` method on a client side:

```javascript
centrifuge.send({"input": "hello"});
```

On server side data will be available inside message handler:

```go
client.OnMessage(func(e centrifuge.MessageEvent) {
    log.Printf("message from client: %s", e.Data)
})
```

And vice-versa, to send data to a client use `Send` method of `centrifuge.Client`:

```go
client.Send([]byte(`{"input": "hello"}`))
```

And listen to it on client-side:

```javascript
centrifuge.on('message', function(data) {
    console.log(data);
});
```

## RPC

RPC is a primitive that allows sending a message from a client to a server and wait for a response (in this case all communication still happens via asynchronous message passing internally, but Centrifuge takes care of matching response data to request previously sent).

On client side it's as simple as:

```javascript
const resp = await centrifuge.namedRPC('my_method', {});
```

On server side RPC event handler should be set to make calls available:

```go
client.OnRPC(func(e centrifuge.RPCEvent, cb centrifuge.RPCCallback) {
    if e.Method == "my_method" {
        cb(centrifuge.RPCReply{Data: []byte(`{"result": "42"}`)}, nil)
        return
    }
    cb(centrifuge.RPCReply{}, centrifuge.ErrorMethodNotFound)
})
```

Note, that it's possible to pass name of RPC and depending on it and custom request params return different results to a client – just like a regular HTTP requests but over asynchronous WebSocket (or SockJS) connection.

## Server-side subscriptions

In most cases client is a source of knowledge which channels it wants to subscribe to on a specific application screen. But sometimes you want to control subscriptions to channels on a server side. This is also possible in Centrifuge.

It's possible to provide a slice of channels to subscribe connection to at the moment of connection establishement phase:

```go
node.OnConnecting(func(ctx context.Context, e centrifuge.ConnectEvent) (centrifuge.ConnectReply, error) {
    return centrifuge.ConnectReply{
        Subscriptions: map[string]centrifuge.SubscribeOptions{
            "example": {},
        },
    }, nil
})
```

Note, that `OnConnecting` does not follow callback-style – this is because it can only happen once on start of each connection – so there is no need to control operation concurrency.

In this case on client side you will have an access to messages published to channels by listening to `on('publish')` event:

```javascript
centrifuge.on('publish', function(msgCtx) {
    console.log(msgCtx);
});
```

Also, `centrifuge.Client` has `Subscribe` and `Unsubscribe` methods so it's possible to subscribe/unsubscribe client to/from channel somewhere in the middle of its long WebSocket session.

## Windowed history in channel

Every time message published to a channel it's possible to provide custom history options. For example:

```go
node.Publish(
    "example",
    []byte(`{"input": "hello"}`),
    centrifuge.WithHistory(300, time.Minute),
)
```

In this case Centrifuge will maintain a windowed Publication cache for a channel - or in other words maintain publication stream. This stream will have a time retention (one minute in example above) and maximum size will be limited to value provided during Publish (300 in example above).

Every message inside a history stream has incremental `offset` field. Also stream has a field called `epoch` – this is a unique identifier of stream generation - thus client will have a possibility to distinguish situations where stream completely removed and there is no guarantee that no messages have been lost in between even if offset looks fine.

Client protocol provides a possibility to paginate over a stream from a certain position with limit (though this is only available with Javascript client at the moment and only available for subscription object - i.e. client must be subscribed to a channel to iterate over its history):

```javascript
const streamPosition = {'offset': 0, epoch: 'xyz'} 
resp = await sub.history({since: streamPosition, limit: 10});
```

Also Centrifuge has an automatic message recovery feature. Automatic recovery very useful in scenarios when tons of persistent connections start reconnecting at once. I already described why this is useful in one of my previous posts about Websocket scalability. In short – since WebSocket connections are stateful then at the moment of mass reconnect they can create a very big spike in load on your main application database. Such mass reconnects are a usual thing on practice - for example when you reload your load balancers or re-deploying Websocket server (new code version).

Of course recovery can also be useful for regular short network disconnects - when user travels in subway for example. But you always need a way to load an actual state from main application database in case of unsuccessful recovery.

To enable automatic recovery you can provide `Recover` flag in subscribe options:

```go
client.OnSubscribe(func(e centrifuge.SubscribeEvent, cb centrifuge.SubscribeCallback) {
    cb(centrifuge.SubscribeReply{
        Options: centrifuge.SubscribeOptions{
            Recover:   true,
        },
    }, nil)
})
```

Obviously, recovery will work only for channels where history stream maintained. The limitation in recovery is that all missed publications sent to client in one protocol frame – pagination is not supported during recovery process. This means that recovery is mostly effective for not too long offline time without tons of missed messages.

## Presence and presence stats

Another cool thing Centrifuge exposes to developers is presence information for channels. Presence information contains a list of active channel subscribers. This is useful to show online status of players in a game for example.

Also it's possible to turn on Join/Leave message feature inside channels: so each time connection subscribes to a channel all channel subscribers receive Join message with client information (client ID, user ID). As soon as client unsubscribes Leave message sent to remaining channel subscribers with information who left a channel.

Here is how to enable both presence an join/leave features for a subscription to channel:

```go
client.OnSubscribe(func(e centrifuge.SubscribeEvent, cb centrifuge.SubscribeCallback) {
    cb(centrifuge.SubscribeReply{
        Options: centrifuge.SubscribeOptions{
            Presence:   true,
            JoinLeave:  true,
        },
    }, nil)
})
```

The important thing to be aware of when using Join/Leave messages is that this feature can dramatically increase a CPU utilization and overall traffic in channels with big number of active subscribers – since on every client connect/disconnect event such Join or Leave message must be sent to all subscribers. The advice here – avoid using Join/Leave messages or be ready to scale (Join/Leave messages scale well when adding more Centrifuge Nodes – more about scalability below).

One more thing to remember is that presence information can also be pretty expensive to request in channels with many active subscribers – since it returns information about all connections – thus payload in response can be large. To help a bit with this situation Centrifuge has a presence stats client API method. Presence stats only contain two counters: amount of active connections in channel and amount of unique users in channel.

If you still need to somehow process presence in rooms with massive number of active subscribers – then I think you better do it in near real-time - for example with fast OLAP like ClickHouse.

## Scalability aspects

To be fair it's pretty simple to implement most of features above inside one in-memory process. Yes, it takes time – but code is mostly straighforward. When it comes to scalability things tend to be a bit harder.

Centrifuge designed with an idea in mind that one machine is not enough to handle all application WebSocket connections. Connections should scale over application backend instances and it should be simple to add more application nodes when the amount of users (connections) grows.

Centrifuge abstracts scalability over `Node` instance and two interfaces: `Broker` interface and `PresenceManager` interface.

Broker is responsible for PUB/SUB and streaming semantics:

```go
type Broker interface {
	Run(BrokerEventHandler) error
	Subscribe(ch string) error
	Unsubscribe(ch string) error
	Publish(ch string, data []byte, opts PublishOptions) (StreamPosition, error)
	PublishJoin(ch string, info *ClientInfo) error
	PublishLeave(ch string, info *ClientInfo) error
	PublishControl(data []byte, nodeID string) error
	History(ch string, filter HistoryFilter) ([]*Publication, StreamPosition, error)
	RemoveHistory(ch string) error
}
```

See [full version with comments](https://github.com/centrifugal/centrifuge/blob/v0.14.2/engine.go#L98) in source code.

It's and important thing to combine PUB/SUB with history to achieve atomicity of saving message into history stream and publishing it to PUB/SUB with generated offset.

PresenceManager is responsible for presence information management:

```go
type PresenceManager interface {
	Presence(ch string) (map[string]*ClientInfo, error)
	PresenceStats(ch string) (PresenceStats, error)
	AddPresence(ch string, clientID string, info *ClientInfo, expire time.Duration) error
	RemovePresence(ch string, clientID string) error
}
```

[Full code with comments](https://github.com/centrifugal/centrifuge/blob/v0.14.2/engine.go#L150).

`Broker` and `PresenceManager` combined together form an `Engine` interface:

```go
type Engine interface {
	Broker
	PresenceManager
}
```

By default Centrifuge uses `MemoryEngine` that does not use any external services but limits developer to using only one Centrifuge Node (i.e. one server instance). Memory Engine is very fast and can be fine for some scenarios - even in production (with configured backup instance) – but as soon as amount of connections grow you need to load balance connections to different server instances. Here comes Redis Engine.

Redis Engine utilizes Redis for Broker and PresenceManager parts.

History cache can be saved to Redis STREAM or Redis LIST data structures. For presence Centrifuge uses combination of HASH and ZSET structures.

Centrifuge tries to fully utilize connection between Node and Redis using pipelining where possible and smart batching technique. All operations can be done in single RTT with the help of small Lua scripts loaded automatically to Redis on engine start.

Redis is pretty fast and will allow your app to scale to some limits. When Redis starts being bottleneck it's possible to shard data over different Redis instances. Client-side consistent sharding is built-in in Cenrifuge and allows to scale further.

It's also possible to achieve Redis high availability with built-in Sentinel support. Also Redis Cluster supported. So Redis Engine covers many options to communicate with Redis deployed in different ways.

At Avito we served about 800k active connections in messenger app with ease using a slightly adapted Centrifuge Redis Engine – so an approach proved to be working for rather big applications. We will look at some more concrete numbers below in performance section.

Both `Broker` and `PresenceManager` are pluggable – so it's possible to replace them with alternative implementations. Examples show [how to use Nats server](https://github.com/centrifugal/centrifuge/tree/master/_examples/custom_broker_nats) for at most once only PUB/SUB together with Centrifuge. Also we have [an example of full-featured Engine for Tarantool database](https://github.com/centrifugal/centrifuge/tree/master/_examples/custom_engine_tarantool) – Tarantool Engine shows even better thoughput for history and presence operations than Redis-based Engine (up to 10x for some ops).

## Ecosystem

Here I want to be fair with my readers – Centrifuge is not ideal. This is a project maintained mostly by one person at the moment with all consequences. This hits an ecosystem a lot, can make some design choices opinionated or non-optimal.

I mentioned in first post that Centrifuge is built on top of its own protocol. The protocol is based on strict Protobuf schema, works with JSON and binary data transfer, supports many features. But – this means that to connect to Centrifuge-based server developers have to use custom connectors that can speak with Centrifuge over its custom protocol.

The difficulty here is that protocol is asynchronous in its nature. Asynchronous protocols are harder to implement than synchronous ones. Multiplexing frames allows to achieve a good performance and fully utilize a single connection – but it hurts simplicity.

At this moment Centrifuge has client connectors for:

* [centrifuge-js](https://github.com/centrifugal/centrifuge-js) - Javascript client for a browser, NodeJS and React Native
* [centrifuge-go](https://github.com/centrifugal/centrifuge-go) - for Go language
* [centrifuge-mobile](https://github.com/centrifugal/centrifuge-mobile) - for mobile development based on centrifuge-go and [gomobile](https://github.com/golang/mobile) project
* [centrifuge-swift](https://github.com/centrifugal/centrifuge-swift) - for iOS native development
* [centrifuge-java](https://github.com/centrifugal/centrifuge-java) - for Android native development and general Java
* [centrifuge-dart](https://github.com/centrifugal/centrifuge-dart) - for Dart and Flutter

Not all clients support all protocol features. Another drawback is that all clients do not have a persistent maintainers – I mostly maintain everything myself. Connectors can have non-idiomatic and pretty dumb code since I had no previous experience with mobile development, they lack proper tests and documentation. This is unfortunate.

But the good thing is that all connectors feel very similar, I am constantly making new releases – especially when someone sends a pull request with improvements or bug fixes. So all connectors are alive.

I maintain feature matrix in connectors to let users understand what's supported. Actually feature support is pretty nice throughout all these connectors - there are only several things missing and not so much work required to make all connectors full-featured. But I really need help here.

It will be a big mistake to not mention Centrifugo as a big plus for Centrifuge library ecosystem. Centrifugo is a server that deployed in many projects throughout the world. Many features of Centrifuge library and its connectors already tested by Centrifugo users.

One more thing to mention is that Centrifuge does not have v1 release. It still evolves – I believe that the most dramatic changes have already been made and backwards compatibility issues will be minimal in next releases – but can't say for sure.

## Performance

I made a test stand in Kubernetes with one million of connections.

I can't call this a proper benchmark – since in benchmark your main goal is to destroy a system, in my test I just achieved some reasonable numbers on a limited hardware. These numbers should give a good insight on a possible throughput, latency and estimate hardware requirements (at least approximately).

Connections landed on different server pods, 5 Redis instances have been used to scale connections between pods.

The detailed test stand description [can be found in Centrifugo documentation](https://centrifugal.github.io/centrifugo/misc/benchmark/).

![Benchmark](../images/benchmark.gif)

Some quick conclusions are:

* One connection costs about 30kb of RAM
* Redis broker CPU utilization increases linearly with more messages travelling around
* 1 million connections with 500k **delivered** messages per second with 200ms delivery latency in 99 percentile can be served with hardware amount equal to one modern physical server machine. Possible amount of messages can vary a lot depending on number of channel subscribers though.

## Limitations

Centrifuge does not allow subscribing on the same channel twice inside a single connection. It's not simple to add due to design decisions made – though there were no single user report about this in seven years of Centrifugo/Centrifuge history.

Centrifuge does not support wildcard subscriptions. Not only because I never needed this myself but also due to some design choices made – so be aware of this.

SockJS fallback does not support binary data - only JSON. If you want to use binary in your application then you can only use WebSocket with Centrifuge - there is no built-in fallback transport in this case.

SockJS also requires sticky session support from your load balancer to achieve stateful (a-la bidirectional) connection with its HTTP fallback transports. Ideally Centrifuge will go away from SockJS at some point, maybe in time when WebTransport become mature.

Websocket permessage-deflate compression supported (thanks to Gorilla WebSocket) but it can be pretty expensive in terms of CPU utiliztion and memory usage.

I am not very happy with current error and disconnect handling throughout connector ecosystem – this can be improved though.

## Examples

I am adding examples to [_examples](https://github.com/centrifugal/centrifuge/tree/master/_examples) folder of Centrifuge repo. These examples completely cover Centrifuge API - including things not mentioned here.

## Conclusion

It's hard to tell where and when you may want to use Centrifuge. I think it's a nice alternative to socket.io - with a better performance and main server implementation in Go language. But since Centrifuge is not backed by lots of contributors you may find Centrifuge ecosystem worse - especially in area of client connectors.

As time shows – Centrifuge fits pretty well proprietary application development where time matters and deadlines are close. For Centrifugo server users it provides a way to write a more flexible server code adapted for business requirements but still use the same real-time core and have the same protocol features.