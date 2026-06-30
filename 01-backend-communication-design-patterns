# 01 — Backend Communication Design Patterns
> Fundamentals of Backend Engineering | Hussein Nasser

---

## Why This Section Matters

Every backend application you build communicates — with clients, with databases, with other services. But most engineers learn only one pattern (request-response) and try to apply it everywhere. This section teaches you the full vocabulary of backend communication so you can pick the right tool for the right job.

These patterns show up in every senior interview. When an interviewer asks "how would you build a notification system?" or "how does Kafka work?", they're testing whether you know these patterns and their tradeoffs.

---

## 1. Request-Response — The Foundation

### What it is

The most fundamental pattern. The client sends a request, the server processes it, the server sends a response. Everything stops until the response arrives.

```
Client                    Server
  |                          |
  |------- Request --------->|
  |                          | (processing)
  |<------ Response ---------|
  |                          |
```

### What actually happens inside a request

This is where most engineers have gaps. A request is not just "data sent over the wire." Here's every step:

```
1. Client serializes data
   (JSON.stringify, protobuf encode, XML build)

2. Client writes to socket
   (TCP segments created, IP packets formed)

3. Packets travel the network
   (routed, may arrive out of order)

4. Server receives and reorders packets

5. Server parses the raw bytes
   (finds where request starts and ends — the boundary problem)

6. Server deserializes the payload
   (JSON.parse, protobuf decode)

7. Server processes the request
   (queries DB, runs business logic)

8. Server serializes the response
9. Response travels back
10. Client parses and deserializes response
```

Every single one of these steps has a cost. Most engineers only think about step 7.

### The boundary problem

TCP is a continuous stream of bytes. When the server receives data, how does it know where one request ends and the next begins?

This is solved by the **protocol**. HTTP/1.1 uses blank lines and `Content-Length` headers. HTTP/2 uses binary frames with explicit length. Protocol buffers use length-prefixed messages. If you're building a custom protocol, you must define this yourself.

This is also why parsing is expensive — for every byte that arrives, the parser is asking "am I still inside a request, or did a new one start?"

### Serialization formats and their costs

| Format | Human readable | Parse speed | Size |
|---|---|---|---|
| XML (SOAP) | Yes | Slowest | Largest |
| JSON (REST) | Yes | Moderate | Medium |
| Protocol Buffers (gRPC) | No | Fastest | Smallest |

This is why companies move from REST/JSON to gRPC for high-throughput internal services. JSON is fine for user-facing APIs where readability matters. Protocol buffers win when you're doing millions of RPC calls per second between internal microservices.

### Where request-response is used

Everywhere: HTTP, SQL queries, DNS, REST APIs, GraphQL, RPC, SOAP.

### When request-response breaks down

Three scenarios where it doesn't work well:

**Long-running requests** — What if processing takes 5 minutes? The client sits there waiting. If it disconnects, the work is lost.

**Real-time notifications** — The server has information the client doesn't have yet (someone just logged in, a new message arrived). The client can't request something it doesn't know exists.

**Multiple receivers** — One event needs to go to many services. Chaining request-responses creates tight coupling and fragile systems.

---

## 2. Synchronous vs Asynchronous Execution

This is one of the most important concepts in backend engineering. Get this wrong and your Node.js server will be slow and your users will be unhappy.

### Synchronous IO — The old way

The caller sends a request and **blocks**. It cannot do anything else until the response arrives.

```
Thread:  [do work] --- blocked waiting for IO --- [continue]
CPU:     [executing]    [idle, OS removes thread]  [executing]
```

The OS literally removes your process from the CPU when it's blocked on IO because there's nothing for it to execute. This is called **context switching** — taking your thread off the CPU and putting another thread on. Context switching costs microseconds, but at scale it adds up.

In the old VB5 / early Windows days, a single blocked IO operation could freeze the entire UI. This was synchronous everything.

### Asynchronous IO — How Node.js works

The caller sends a request and **continues executing**. It doesn't wait. When the response eventually arrives, a callback is invoked.

```
Thread:  [do work] [send IO request] [continue doing other work] [callback fires]
CPU:     [executing throughout — never idle]
```

Node.js achieves this two ways depending on the platform:

**On Linux:** Uses `epoll` — a system call that says "tell me when any of these file descriptors are ready to read." Instead of blocking on one connection, Node.js registers all connections with epoll and processes whichever one has data ready.

**For file system operations:** `epoll` doesn't work with regular files. Node.js spawns a thread pool (default 4 threads via libuv). The main thread assigns the file read to a worker thread, which blocks while the main thread continues. When the worker finishes, it calls back the main thread.

This is why Node.js can handle thousands of concurrent connections on a single thread — it's never blocked, always processing whatever is ready.

### The three syntaxes in Node.js — same mechanism, different style

```javascript
// Style 1: Callbacks (original)
fs.readFile('data.txt', (err, data) => {
    console.log(data.toString()); // called when ready
});
console.log('This runs immediately'); // doesn't wait

// Style 2: Promises
fs.promises.readFile('data.txt')
    .then(data => console.log(data.toString()));
console.log('This also runs immediately');

// Style 3: Async/await (syntactic sugar over promises)
async function readData() {
    const data = await fs.promises.readFile('data.txt');
    // looks synchronous but the thread is NOT blocked
    console.log(data.toString());
}
```

`async/await` looks like blocking code but it isn't. When you `await`, you yield control back to the event loop. The event loop can process other requests while you're waiting. When your IO completes, the event loop resumes your function.

### Asynchronous backend processing

This is a pattern you'll use constantly at senior level. The client sends a request, but instead of processing it immediately, the server queues it and returns a job ID.

```
Client                    Server                    Queue
  |                          |                         |
  |------- POST /job ------->|                         |
  |                          |---- push job ---------->|
  |<------ { jobId: 123 } ---|                         |
  |                          |         (worker picks up job)
  |--- GET /job/123/status -->|                         |
  |<-- { status: "pending" }--|                         |
  |--- GET /job/123/status -->|                         |
  |<-- { status: "done" } ---|                         |
```

The client can disconnect, save the job ID, and come back later. This is how YouTube video uploads work — you get an upload ID immediately, processing happens in the background.

### Postgres async commits

Even your database uses this pattern. By default, Postgres waits for the WAL to flush to disk before returning commit success — this is synchronous commit. With `synchronous_commit = off`, Postgres returns success immediately and flushes asynchronously. Faster, but risk of losing the last few transactions on a crash.

Connects to what you know: this is exactly what you learned in Section 8 of the database course.

---

## 3. Push

### What it is

The server sends data to the client **without** the client requesting it. The only requirement is an established connection.

```
Client                    Server
  |<------- data ------------|  (client didn't ask)
  |<------- data ------------|
  |<------- data ------------|
```

### How it works technically

TCP connections are bidirectional. The server literally writes to the client's socket. The client's OS buffers this data and your application reads it. WebSockets expose this cleanly — once the WebSocket handshake completes, either side can write to the connection at any time.

### When to use push

Real-time scenarios: chat applications, live notifications, live scores, collaborative editing (like Google Docs). Anything where the server has information the client needs immediately.

### The client capacity problem — why Kafka doesn't push

If the server pushes faster than the client can process, the client's buffer fills up and it crashes or drops messages. The server has no knowledge of the client's processing speed.

This is why Kafka chose **long polling** instead of push. Message consumers are often doing heavy processing (database writes, external API calls). Kafka can't know when they're ready for the next batch. By making consumers pull, each consumer controls its own intake rate.

RabbitMQ chose push — it pushes messages to consumers as fast as they arrive. This works when consumers are fast. Kafka's pull model works when consumers are slow or unpredictable.

### The 100M subscriber problem

Even if you wanted to push a notification to 100M YouTube subscribers when a video is uploaded, you can't maintain 100M persistent connections. YouTube pushes to Apple's APNs and Google's FCM (Firebase Cloud Messaging) instead — they manage the connections to devices.

### WebSocket demo concept

```javascript
// Server
const connections = [];

wss.on('connection', (socket) => {
    connections.push(socket);

    socket.on('message', (msg) => {
        // Push to ALL connected clients
        connections.forEach(conn => {
            if (conn.readyState === WebSocket.OPEN) {
                conn.send(msg); // server pushes, client didn't ask
            }
        });
    });
});
```

---

## 4. Short Polling

### What it is

The client repeatedly asks "is it done yet?" on a short interval. Each poll is a normal request-response.

```
Client                    Server
  |--- POST /submit -------->|
  |<-- { jobId: "abc" } -----|

  |--- GET /status/abc ------>|
  |<-- { status: "pending" }--|  (immediately)

  |--- GET /status/abc ------>|  (2 seconds later)
  |<-- { status: "pending" }--|

  |--- GET /status/abc ------>|  (2 more seconds)
  |<-- { status: "done" } ---|
```

### When to use it

Long-running background jobs where the client wants to show progress. File uploads, video processing, report generation, anything that takes more than a few seconds.

### The key advantage: client resilience

The client can disconnect after receiving the job ID. It persists the ID to disk. When it reconnects, it reads the pending job IDs and resumes polling. The client and server are decoupled in time.

### The cost: chattiness

If you have 10,000 clients each polling every 2 seconds, that's 5,000 requests per second hitting your backend — most of which return "pending." This wastes:
- Network bandwidth
- Server CPU (each poll costs a DB lookup)
- Money (cloud egress costs money per GB)

You pay for all the "no" answers.

---

## 5. Long Polling

### What it is

The client asks "is it done yet?" but the server **doesn't respond until it is done**. The connection hangs open.

```
Client                    Server
  |--- GET /status/abc ------>|
  |                           | (server holds connection open)
  |                           | (job processing...)
  |                           | (job done!)
  |<-- { status: "done" } ---|  (responds now)

  |--- GET /status/abc ------>|  (next long poll immediately)
  |                           | (waiting again...)
```

### This is exactly how Kafka consumers work

A Kafka consumer sends "give me messages from partition X." If there are no messages, Kafka holds the connection. The moment a message is produced to that partition, Kafka responds to all waiting consumers. No wasted "are you done yet?" polls. The consumer controls when it's ready for the next batch.

### Node.js implementation

```javascript
// Server: don't respond until job is complete
app.get('/status/:jobId', async (req, res) => {
    const { jobId } = req.params;

    // Keep checking until done (or timeout)
    while (true) {
        const job = jobs[jobId];
        if (job.progress === 100) {
            return res.json({ status: 'done', result: job.result });
        }
        // Wait 1 second before checking again
        await new Promise(r => setTimeout(r, 1000));
        // This await yields to event loop — other requests can be handled
    }
});
```

The `await` inside the loop is critical. Without it, this loop would block the Node.js event loop, freezing the entire server. The `await` yields control, allowing other requests to be processed while this one waits.

### Short polling vs long polling

| | Short Polling | Long Polling |
|---|---|---|
| Server responds | Immediately (yes or no) | Only when ready |
| Network usage | High (many empty responses) | Low (fewer responses) |
| Latency | Depends on poll interval | Near real-time |
| Client complexity | Simple | Moderate |
| Server complexity | Simple | Moderate |
| Used by | Basic status checks | Kafka, Comet, chat |

---

## 6. Server-Sent Events (SSE)

### The elegant trick

SSE is a hack that turns HTTP request-response into a one-way stream. The client makes ONE request. The server never finishes writing the response. It just keeps writing events, forever.

```
Client                    Server
  |--- GET /stream ---------->|
  |<-- headers + event 1 -----|  (response starts but doesn't end)
  |<-- event 2 ---------------|
  |<-- event 3 ---------------|
  |<-- event 4 ---------------|
  |           ...             |  (never closes)
```

### How the boundary works

The server sends events in this format:
```
data: {"user": "arjun", "action": "login"}\n\n
data: {"user": "sarah", "action": "comment"}\n\n
```

The `\n\n` (double newline) is the event boundary. The browser's `EventSource` object parses these chunks automatically and fires an `onmessage` event for each one.

### Content-Type is the signal

When the server responds with `Content-Type: text/event-stream`, the browser knows this is an SSE stream and switches to streaming mode. Without this header, it would wait for the full response before parsing.

### HTTP/1.1 limitation with SSE

Chrome allows maximum 6 TCP connections per domain with HTTP/1.1. If 6 browser tabs all open SSE connections to the same domain, all 6 connections are occupied. A 7th request (even a simple CSS fetch) is blocked until one SSE connection closes.

HTTP/2 solves this — one TCP connection with unlimited streams (default ~200). All SSE streams go through the same connection, and regular requests use the same connection without competition.

This is why HTTP/2 matters in practice, not just theory.

### SSE vs WebSockets

| | SSE | WebSockets |
|---|---|---|
| Direction | Server → Client only | Bidirectional |
| Protocol | Plain HTTP | WebSocket (upgrade from HTTP) |
| Works with HTTP/1.1 | Yes (with 6-connection limit) | Yes |
| Browser support | Native `EventSource` | Native `WebSocket` |
| Use case | News feeds, notifications | Chat, gaming, collaborative editing |

Use SSE when you only need server → client. Use WebSockets when you need both directions.

---

## 7. Publish-Subscribe (Pub/Sub)

### The problem it solves

Microservices need to talk to each other. Without pub/sub, you get this:

```
Upload Service ──→ Compression Service
                ──→ Format Service
                ──→ Notification Service
                ──→ Copyright Service
```

Every service is directly coupled to every other. Adding a new service means modifying Upload Service. If any downstream service is down, the entire chain fails.

### The pub/sub solution

```
Upload Service ──→ [Broker: "raw-video" topic]
                        ↓
             ┌──────────┼──────────┐
      Compression   Format     Notification
        Service    Service      Service
```

Upload Service publishes to a topic and moves on. It doesn't know or care who consumes it. Each consumer subscribes independently. New consumers can be added without touching Upload Service.

### Key vocabulary

**Publisher** — produces messages to a topic
**Subscriber/Consumer** — reads messages from a topic
**Topic** — a named channel of messages
**Broker** — the server that holds topics (Kafka, RabbitMQ)

### Push vs Pull in pub/sub

**RabbitMQ pushes** — the broker pushes messages to connected consumers immediately. Fast, but consumers can be overwhelmed.

**Kafka pulls (long polls)** — consumers ask for messages when they're ready. Consumer controls its intake rate. Better for slow consumers.

### Message acknowledgment — the hard problem

When a consumer receives a message, it must acknowledge it. If the consumer crashes before acknowledging, the broker redelivers the message to another consumer.

This creates **at-least-once delivery** — every message is processed at least once, but might be processed twice. Making your consumers idempotent (safe to run twice) is a key engineering challenge with pub/sub.

### Pub/sub vs request-response

| | Request-Response | Pub/Sub |
|---|---|---|
| Coupling | Tight (caller knows callee) | Loose (publisher doesn't know consumers) |
| Multiple receivers | Hard (must call each explicitly) | Easy (all subscribers get it) |
| Client availability | Both must be online | Consumer can be offline |
| Failure isolation | One failure breaks chain | Consumers fail independently |
| Use case | Direct queries, CRUD | Events, notifications, pipelines |

---

## 8. Multiplexing and Demultiplexing

### The concept

**Multiplexing** — combining multiple signals/streams into one channel.
**Demultiplexing** — splitting one channel back into multiple signals/streams.

This shows up everywhere in backend engineering.

### HTTP/1.1 — No multiplexing, connection pool instead

In HTTP/1.1, one TCP connection can carry one request at a time. The browser opens up to 6 parallel connections per domain to compensate.

```
Browser:  Connection 1 ──→ request A
          Connection 2 ──→ request B
          Connection 3 ──→ request C
          (3 more available)
```

### HTTP/2 — True multiplexing

HTTP/2 multiplexes multiple **streams** over one TCP connection. Stream 1 carries request A, Stream 2 carries request B, Stream 3 carries request C — all interleaved in the same connection.

```
Browser:  One TCP connection ──→ [Stream 1: request A]
                                  [Stream 2: request B]
                                  [Stream 3: request C]
```

Benefits: fewer TCP handshakes, no 6-connection limit, no head-of-line blocking between requests, SSE streams don't block other requests.

Cost: HTTP/2 parsing is CPU-intensive on the server. One connection serving hundreds of streams puts load on the server to demultiplex them.

### Connection pooling — multiplexing to the database

This is the same concept applied to database connections. Your Node.js app opens a pool of N database connections and multiplexes all incoming queries across them.

```javascript
// pg pool — you already know this from Node/Postgres work
const pool = new Pool({ max: 10 }); // 10 DB connections

// Multiple concurrent requests share these 10 connections
app.get('/users', async (req, res) => {
    const result = await pool.query('SELECT * FROM users');
    // pool.query() picks an available connection automatically
    res.json(result.rows);
});
```

If all 10 connections are busy, the 11th query waits. This is the queue in front of the pool.

### The 6-connection limit demo Hussein showed

When you open 6 long-polling connections in a browser to the same domain (HTTP/1.1), the 7th request is blocked. You can literally see it in Chrome DevTools — the request shows "Stalled" state, meaning it's waiting for a connection slot to open up. This is the 6-connection pool of the browser exhausted.

---

## 9. Stateful vs Stateless

### The definition that actually matters

**Stateful** = the server stores information about a client in memory and **relies on it being there**. If the server restarts, the client's workflow breaks.

**Stateless** = the client sends everything the server needs with every request. The server can be restarted or replaced at any time without breaking client workflows.

### The practical test

Can you restart your backend during idle time and have clients continue working as if nothing happened? If yes, your backend is stateless. If no, you have state stored somewhere that clients depend on.

### Classic stateful failure — session in memory

```javascript
// STATEFUL — session stored in server memory
app.post('/login', (req, res) => {
    // verify password...
    sessions[sessionId] = { userId: user.id }; // stored in this server's memory
    res.cookie('sessionId', sessionId);
});

app.get('/profile', (req, res) => {
    const session = sessions[req.cookies.sessionId]; // only exists on THIS server
    if (!session) return res.redirect('/login'); // fails if load balancer sends to different server
});
```

With 3 servers behind a load balancer, this breaks randomly. Request 1 (login) hits Server A and creates the session. Request 2 (profile) hits Server B — session doesn't exist there — user is logged out.

### Fix — move state to a shared store

```javascript
// STATELESS backend — session in Redis
app.post('/login', async (req, res) => {
    // verify password...
    await redis.set(`session:${sessionId}`, JSON.stringify({ userId: user.id }), 'EX', 3600);
    res.cookie('sessionId', sessionId);
});

app.get('/profile', async (req, res) => {
    const session = JSON.parse(await redis.get(`session:${req.cookies.sessionId}`));
    if (!session) return res.redirect('/login');
    // works on any server — they all share Redis
});
```

The backend is now stateless (restart any server, clients keep working). The system is still stateful (state lives in Redis). This distinction matters for horizontal scaling.

### Sticky sessions — the band-aid

Some load balancers support "sticky sessions" — routing all requests from the same client to the same server. This makes stateful backends work without fixing the root problem. It breaks when that server restarts or is removed. Only use this as a temporary measure.

### JWT — stateless authentication

JSON Web Tokens encode the entire session into a signed token. The server doesn't store anything.

```
Token: base64(header).base64(payload).signature
Payload: { userId: 123, role: "admin", exp: 1719825600 }
```

Any server can verify the token by checking the signature (using the secret key) without hitting a database. Pure stateless.

**The tradeoff:** You can't invalidate a JWT before it expires. If someone's token is stolen and you want to log them out immediately, you can't — unless you maintain a blacklist (which makes you stateful again). This is why JWT access tokens should have short expiry (15 minutes) with longer-lived refresh tokens.

### Stateful protocols

**TCP** is stateful — both sides maintain sequence numbers, window sizes, connection state machine. If either side loses this state, the connection is dead.

**UDP** is stateless — each datagram is independent. DNS uses UDP and adds a query ID to correlate responses, but the protocol itself maintains no state.

**HTTP** is stateless built on stateful TCP — each HTTP request is independent (that's why you need cookies/JWT to remember who you are). HTTP uses TCP as a transport layer without caring if the TCP connection dies and restarts.

---

## 10. Sidecar Pattern

### The problem

Every protocol requires a library. gRPC requires a gRPC library. HTTP/2 requires an HTTP/2 library. TLS requires OpenSSL. These libraries:
- Must be the same language as your application
- Are hard to upgrade (breaking changes, retesting)
- Lock you into a technology stack
- Are complex to configure correctly

If Twitter wants all services to use the same networking library (Finagle), every service must be written in Scala.

### The sidecar solution

Instead of putting the complex protocol library inside your application, put it in a separate proxy that runs alongside your application (the "sidecar"). Your app communicates with the sidecar using simple HTTP/1.1 (which never changes). The sidecar handles all the complex protocols.

```
Your App (any language)
     ↕ HTTP/1.1 (simple, stable)
Sidecar Proxy (Envoy, Linkerd)
     ↕ HTTP/2 + TLS 1.3 + distributed tracing + retry logic
Other Services
```

### What the sidecar handles

- Protocol upgrades (HTTP/1.1 → HTTP/2 → HTTP/3) without changing your app
- TLS termination and certificate management
- Distributed tracing (adds trace ID to every request automatically)
- Service discovery (finds where other services are)
- Retry logic and circuit breaking
- Load balancing between instances
- Observability and metrics

### This is service mesh

When every service has a sidecar proxy and all proxies talk to a central control plane, that's a **service mesh** (Istio, Linkerd, Consul Connect). The control plane manages routing rules, security policies, and certificates across all services from one place.

### The cost

Latency. Every request now makes two extra hops — into the sidecar and out of the sidecar on each end. The hops are local (loopback, not network) but they still cost CPU time for protocol parsing, TLS operations, and trace injection.

At high throughput, this adds up. Companies with extremely tight latency budgets sometimes bypass service mesh for certain paths.

---

## Connecting Everything — When to Use What

| Scenario | Pattern | Why |
|---|---|---|
| Fetch user profile | Request-Response | Simple, direct, instant |
| Upload 2GB file | Request-Response + chunking | Resumable, progress tracking |
| Check if background job is done | Short Polling | Simple, client can disconnect |
| Kafka consumer | Long Polling | Client controls rate, less chatty |
| Live notification feed | SSE | Server → client only, HTTP compatible |
| Chat application | WebSockets (Push) | Bidirectional, real-time |
| Video upload pipeline | Pub/Sub | Multiple consumers, loose coupling |
| Multiple DB queries | Connection Pool | Multiplexing, efficient connection use |
| Login sessions (horizontally scaled) | Stateless + Redis | Any server can handle any request |
| Protocol upgrade without code changes | Sidecar | Decouple protocol from app |

---

## Glossary

| Term | Definition |
|---|---|
| Request boundary | How a parser knows where one request ends and the next begins. Defined by the protocol. |
| Serialization | Converting an in-memory object (JS object, Python dict) to bytes for transmission. |
| Deserialization | Converting received bytes back to an in-memory object. |
| epoll | Linux system call for efficient async IO monitoring. Node.js uses this on Linux. |
| Context switching | OS removing a thread from CPU and putting another in its place. Costs microseconds. |
| libuv | C library Node.js uses for async IO. Provides thread pool for file operations. |
| Event loop | Node.js's main loop that checks for ready callbacks, timers, and IO events. |
| Long polling | Client hangs connection open; server responds only when it has data. Used by Kafka. |
| SSE | Server-Sent Events. Single HTTP request with never-ending response. Server pushes chunks. |
| WebSocket | Full-duplex protocol. Both client and server can send at any time. |
| Pub/Sub | Publisher writes to topic. All subscribers receive messages. Loose coupling. |
| Broker | The server that holds pub/sub topics. Kafka, RabbitMQ. |
| At-least-once delivery | Every message processed at least once. May be duplicated. Requires idempotent consumers. |
| Multiplexing | Combining multiple streams into one channel. HTTP/2 over one TCP connection. |
| Head-of-line blocking | Later requests blocked behind a slow earlier request. HTTP/1.1 problem. |
| Connection pool | Pre-opened set of connections reused across requests. Pg Pool in Node/Postgres. |
| Sticky session | Load balancer routing all requests from one client to the same server. Workaround for stateful apps. |
| JWT | JSON Web Token. Self-contained signed token. Stateless authentication. |
| Sidecar | A proxy running alongside your app handling complex protocols on its behalf. |
| Service mesh | Network of sidecars controlled by a central plane. Istio, Linkerd. |
| ALPN | Application Layer Protocol Negotiation. TLS extension that negotiates HTTP version. |
| ABI | Application Binary Interface. How different compiled libraries can interoperate. |

---

## Quick Revision

- Request-response = client asks, server answers. Both must be online. Parsing has a cost.
- Synchronous IO = thread blocks, OS removes it from CPU. Asynchronous IO = thread continues, callback fires when ready.
- Node.js uses epoll (Linux) for network async and a thread pool (libuv) for file async.
- async/await is syntactic sugar. The thread is NOT blocked — the event loop continues.
- Push = server writes to client without being asked. Requires persistent connection. Client must keep up.
- Short polling = client repeatedly asks "done yet?" — simple but chatty and expensive.
- Long polling = client asks, server holds connection until it has something. Kafka uses this.
- SSE = one HTTP request, never-ending response. Server pushes chunks. HTTP/1.1 limited to 6 SSE connections per domain.
- HTTP/2 multiplexes many streams over one TCP connection — solves the 6-connection limit.
- Pub/Sub = publisher doesn't know consumers. Consumers are independent. Used in microservices pipelines.
- Stateless backend = can restart without breaking clients. Session state lives in Redis/DB, not server memory.
- JWT = stateless auth. Can't revoke before expiry. Use short access tokens + longer refresh tokens.
- Sidecar = delegate protocol complexity to a proxy. Your app stays simple. Service mesh is many sidecars + control plane.

---

## Interview Q&A

**Q: What happens when you type a URL and press Enter?**
DNS lookup → TCP connection → TLS handshake → HTTP request (serialized, broken into TCP segments) → routed to server → server parses request boundary → deserializes payload → processes → serializes response → client receives, reorders segments → parses response → renders. Each step has cost. Parsing and serialization are often overlooked bottlenecks.

**Q: What's the difference between synchronous and asynchronous IO?**
Synchronous IO blocks the calling thread — the OS removes it from the CPU because it's not executing instructions. Asynchronous IO lets the thread continue — it registers interest in an event (via epoll on Linux) and gets a callback when the IO completes. Node.js is async by design — the event loop never blocks, it just processes whichever connections have data ready.

**Q: How would you build a real-time notification system?**
Start with the requirement — is it truly real-time (sub-second) or near-real-time (a few seconds is fine)?
For truly real-time: WebSockets or SSE. SSE if it's server → client only (simpler). WebSockets if bidirectional.
For near-real-time: long polling. Less connection overhead, works with HTTP/1.1, client controls rate.
At YouTube scale: push to Apple APNs / Google FCM — they manage device connections.
I'd avoid short polling for notifications — too chatty, most responses are empty.

**Q: Why did Kafka choose long polling instead of push?**
Kafka's consumers often do heavy processing — writing to databases, calling external APIs. The server can't know when a consumer is ready for the next batch. If Kafka pushed, fast producers would overwhelm slow consumers. Long polling lets each consumer pull at its own pace. The consumer sends a request, Kafka holds it until messages are available, the consumer processes them, then requests the next batch. Consumer controls the flow.

**Q: What is the difference between SSE and WebSockets?**
SSE is unidirectional (server → client only) and works over plain HTTP — one request with a never-ending chunked response. It has the 6-connection-per-domain limit with HTTP/1.1. WebSockets are bidirectional, require a protocol upgrade handshake, and use a separate connection. Use SSE for feeds and notifications. Use WebSockets for chat, gaming, collaborative editing.

**Q: How does connection pooling work and why do we need it?**
Establishing a TCP connection is expensive (SYN, SYN-ACK, ACK handshake + TLS if encrypted). Connection pooling pre-establishes N connections and reuses them. Incoming requests grab a free connection from the pool, execute the query, and return the connection. This is multiplexing — many application requests share a small number of actual connections. In Node.js with pg, `new Pool({ max: 10 })` opens 10 Postgres connections. pool.query() automatically manages acquisition and release.

**Q: What is the sidecar pattern and why use it?**
The sidecar pattern delegates network protocol complexity to a proxy running alongside your application. Instead of embedding an HTTP/2 library, a TLS library, and retry logic into every service, you put all of that in a sidecar proxy (Envoy, Linkerd). Your service talks to the sidecar over simple HTTP/1.1 on localhost. The sidecar handles protocol upgrades, TLS, distributed tracing, retries, and service discovery. This makes your services language-agnostic and lets you upgrade network behavior across all services by updating one proxy config, not hundreds of services.

**Q: What's wrong with stateful backends in horizontally scaled systems?**
If session state is stored in one server's memory, a load balancer sending subsequent requests to a different server causes that server to lose the session — the user gets logged out randomly. Solutions: move state to a shared store (Redis), use JWT (stateless tokens), or use sticky sessions (load balancer always routes a client to the same server — but breaks when that server restarts). The clean solution is stateless backends with shared state in Redis.

**Q: Explain pub/sub and when you'd use it over request-response.**
Pub/sub has publishers that write to topics and consumers that subscribe independently. The publisher doesn't know who consumes. Use pub/sub when: you have multiple services that all need the same event (video upload → compression + format + notification + copyright), you want loose coupling between services (adding a new consumer doesn't require changing the publisher), or consumers should be able to process at their own pace. Request-response is better for direct queries where you need the result immediately and only one service is involved.
