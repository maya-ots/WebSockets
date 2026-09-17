# WebSocket Workshop Topic
## workshop structure :
Problem → limitations of HTTP → alternatives → WebSocket → how it works → what we build on top of it → what breaks in production → how we scale it

HTTP → why real-time is difficult → polling → long polling → WebSocket → handshake → lifecycle → frames/messages → broadcasting/rooms → security → reconnection → scaling → Redis/NGINX..

# WebSocket Workshop

## Workshop Goal

The goal of this workshop is to understand how real-time communication works on the web, why traditional HTTP can become inefficient for real-time applications, how WebSockets solve this problem, and what additional challenges appear when building a production-ready WebSocket system.

By the end of the workshop, participants should understand:

* How HTTP request/response communication works.
* Why applications sometimes need persistent, bidirectional communication.
* How the WebSocket HTTP upgrade and handshake work.
* How WebSocket connections, messages, frames, and control frames work.
* How applications implement broadcasting and rooms.
* How to design a basic WebSocket message protocol.
* How authentication and security work with WebSockets.
* How to handle failures and reconnection.
* Why WebSocket applications become more difficult to scale.
* How NGINX and Redis Pub/Sub can be used in a scaled architecture.

---

# 0. Hook — How Does a Chat Message Actually Travel?

let me ask you a question:

 Alice presses "Send". How does Bob receive the message?

At first glance:

```text
Alice
  │
  │ "Hello Bob"
  ▼
Server
  │
  │ "Hello Bob"
  ▼
Bob
```

But what is actually happening underneath?

* How does Alice connect to the server?
* Is HTTP involved?
* Does the server continuously listen to Alice?
* How does the server know which clients should receive the message?
* What happens if Alice loses her connection?
* What happens if there are 100,000 connected users?
* What happens if we have multiple backend servers?

**tkemlo l workshop tefahmo 3lech**

---

# 1. Client/Server & HTTP

## 1.1 Client–Server Architecture

A web application generally consists of clients communicating with servers across a network.

```text
Client
   │
   │ Request
   ▼
Server
   │
   │ Response
   ▼
Client
```

The client could be:

* A web browser
* A mobile application
* A desktop application
* Another backend service

The server receives requests, performs application logic, accesses databases or other services, and produces responses.

---

## 1.2 What Is HTTP?

HTTP stands for **HyperText Transfer Protocol**.

It is an application-layer protocol used for communication between clients and servers.

HTTP is not limited to HTML pages. It is also used for:

* REST APIs
* JSON
* Images
* Files
* Authentication
* Forms
* Web applications

Its fundamental communication model is:

```text
Client → HTTP Request
Server → HTTP Response
```

---

# 2. HTTP Request & Response

## 2.1 HTTP Request

An HTTP request can conceptually be divided into:

```text
HTTP REQUEST
│
├── Method
├── Request Target
├── Headers
└── Body
```

Example:

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer abc123

{
  "name": "Alice"
}
```

---

## 2.2 HTTP Methods

The method describes the intended operation.

| Method  | Common meaning                           |
| ------- | ---------------------------------------- |
| GET     | Retrieve a resource                      |
| POST    | Create/process/submit data               |
| PUT     | Replace a resource                       |
| PATCH   | Partially modify a resource              |
| DELETE  | Delete a resource                        |
| HEAD    | Retrieve headers without a response body |
| OPTIONS | Discover supported communication options |

For example:

```http
GET /users/42
```

asks for user 42.

Whereas:

```http
DELETE /users/42
```

requests that user 42 be deleted.

HTTP methods also have semantics such as **safe** and **idempotent** behavior.

---

## 2.3 Request Target

Example:

```http
GET /users/42?verbose=true HTTP/1.1
Host: api.example.com
```

The request target is:

```text
/users/42?verbose=true
```

It contains:

```text
/users/42
    ↓
path

?verbose=true
    ↓
query
```

The `Host` header identifies the host:

```http
Host: api.example.com
```

---

## 2.4 Headers

Headers contain metadata and control information.

Examples:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer abc123
Cookie: session=xyz
```

### Content-Type

Describes the media type of the body.

```http
Content-Type: application/json
```

means:

 The body is JSON.

### Accept

Describes what response formats the client prefers.

```http
Accept: application/json
```

means:

 I would prefer JSON in the response.

### Authorization

Carries authentication credentials.

```http
Authorization: Bearer <token>
```

### Cookie

Sends cookies previously stored by the browser.

```http
Cookie: sessionId=abc123
```

---

## 2.5 Request Body

The body contains the payload being sent.

Example:

```http
POST /users HTTP/1.1
Content-Type: application/json

{
  "name": "Alice",
  "age": 24
}
```

The body can contain:

* JSON
* Form data
* Multipart file uploads
* Plain text
* Binary data

A `GET` request commonly has no body.

---

# 3. HTTP Response

The response can conceptually be divided into:

```text
HTTP RESPONSE
│
├── Status Code
├── Headers
└── Body
```

Example:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /users/42

{
  "id": 42,
  "name": "Alice"
}
```

---

## 3.1 HTTP Status Codes

Status codes describe the result of the request.

```text
2xx → Success
3xx → Redirection
4xx → Client/request error
5xx → Server-side failure
```

Important examples:

```text
200 OK
201 Created
204 No Content

301 Moved Permanently
302 Found
304 Not Modified
307 Temporary Redirect
308 Permanent Redirect

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
429 Too Many Requests

500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

A useful distinction:

```text
401 → Authentication problem
403 → Authenticated but not allowed
```

---

## 3.2 Response Headers

Example:

```http
Content-Type: application/json
Content-Length: 57
Cache-Control: max-age=3600
Set-Cookie: session=abc123
```

### Content-Type

Describes the response body.

```http
Content-Type: application/json
```

### Content-Length

Describes the body size in bytes.

### Set-Cookie

Instructs the browser to store a cookie.

```http
Set-Cookie: sessionId=abc123
```

The browser may later send:

```http
Cookie: sessionId=abc123
```

---

## 3.3 Response Body

The body contains the returned data.

For an API:

```json
{
  "id": 42,
  "name": "Alice"
}
```

For a webpage:

```html
<html>
  ...
</html>
```

For an image, it can contain binary data.

---

# 4. HTTP Statelessness

HTTP is fundamentally stateless at the protocol level.

This means that HTTP does not inherently require the server to remember previous requests.

For example:

```text
Request 1
GET /profile

Request 2
GET /messages
```

HTTP does not automatically define these as part of the same conversation.

Applications can introduce state using mechanisms such as:

* Cookies
* Sessions
* Tokens
* Databases
* External state stores

Example:

```text
Login
  ↓
Server creates session
  ↓
Set-Cookie
  ↓
Browser stores cookie
  ↓
Future request
  ↓
Cookie: sessionId=abc123
```

Therefore:

 Stateless HTTP does not mean applications cannot have state.

It means the protocol itself does not automatically maintain conversational state between requests.

---

# 5. Persistent HTTP Connections

HTTP/1.1 introduced persistent connections as the normal behavior.

Instead of:

```text
TCP connection
    ↓
Request
    ↓
Response
    ↓
Close
```

a connection can remain available:

```text
TCP connection
│
├── Request → Response
├── Request → Response
├── Request → Response
└── ...
```

In HTTP/1.x, this is associated with:

```http
Connection: keep-alive
```

Persistent connections reduce the overhead of repeatedly establishing TCP connections.

However:

 A persistent HTTP connection is NOT the same thing as a WebSocket connection.

HTTP still fundamentally follows:

```text
Client → Request
Server → Response
```

---

# 6. HTTP Caching

HTTP has a caching system that allows browsers, proxies, and CDNs to reuse responses.

For example:

```http
Cache-Control: max-age=3600
```

means the response can generally be considered fresh for 3600 seconds.

Another mechanism is ETag:

```http
ETag: "abc123"
```

The client can later send:

```http
If-None-Match: "abc123"
```

If nothing changed, the server can respond:

```http
304 Not Modified
```

The client then reuses its cached copy.

Caching can happen at multiple layers:

```text
Browser
   ↓
CDN
   ↓
Reverse Proxy
   ↓
Application
```




---

# 8. Why HTTP Isn't Enough for Some Real-Time Applications

HTTP works extremely well for normal web communication.

But real-time applications introduce a different requirement.

Consider a chat application.

Bob sends a message:

```text
Bob
  ↓
Server
```

The server needs to notify Alice:

```text
Server
  ↓
Alice
```

But Alice did not necessarily ask:

```text
"Do I have a new message?"
```

The problem becomes:

> How can the server send information to the client when the client did not initiate a new request?

This is where different real-time communication techniques appear.

---

# 9. Polling

Polling repeatedly asks the server whether something changed.

```text
Client → "Anything new?"
Server → "No"

Client → "Anything new?"
Server → "No"

Client → "Anything new?"
Server → "Yes"
```

### Advantages

* Simple
* Uses standard HTTP
* Easy to implement
* Works with traditional infrastructure

### Problems

* Unnecessary requests
* Increased server load
* Latency depends on polling interval
* Wastes resources when nothing changes

---

# 10. Long Polling

Long polling keeps an HTTP request open until the server has something to return or a timeout occurs.

```text
Client
  │
  │ Request
  ▼
Server
  │
  │ waits...
  │
  │ event occurs
  ▼
Client ← Response
```

The client then creates another request.

### Advantages

* Less wasteful than normal polling
* Lower latency
* Works with HTTP infrastructure

### Problems

* Requests still need to be recreated
* More connection management
* More complexity
* Not truly bidirectional

---

# 11. WebSockets

## 11.1 What Are WebSockets?

WebSocket is a protocol that provides a persistent, bidirectional communication channel between a client and server.

Instead of:

```text
Client → Request
Server → Response
```

the connection becomes:

```text
Client ←────────→ Server
       messages
```

Either side can send data whenever necessary.

---

# 12. Why WebSockets?

WebSockets are useful when applications need frequent, low-latency, bidirectional communication.

Examples:

* Chat applications
* Live notifications
* Multiplayer games
* Collaborative editing
* Live dashboards
* Presence systems
* Typing indicators
* Real-time trading interfaces

The key characteristics are:

```text
Persistent
Bidirectional
Full-duplex
Low overhead after connection
```

---

# 13. WebSocket Architecture

```text
Client A ─────┐
Client B ─────┼──→ WebSocket Server
Client C ─────┘
```

The server maintains connections to clients.

After the connection is established, clients and the server can exchange messages independently.

---

# 14. Client and Server Roles

The client normally initiates the WebSocket connection.

```text
Client
   │
   │ Connection request
   ▼
Server
```

The server can:

* Accept the connection
* Reject the connection
* Authenticate the client
* Send messages
* Receive messages
* Close the connection

After the connection is established, both sides can send data.

---

# 15. WebSocket URLs

WebSockets use:

```text
ws://
wss://
```

`ws://` is the unencrypted WebSocket scheme.

`wss://` is WebSocket over TLS.

For production applications, `wss://` should generally be used.

Conceptually:

```text
ws://
WebSocket
   ↓
TCP
```

and:

```text
wss://
WebSocket
   ↓
TLS
   ↓
TCP
```

---

# 16. Why Does WebSocket Start With HTTP?

A WebSocket connection begins with an HTTP request.

This allows WebSocket to use the existing HTTP-based web infrastructure and establish the connection through an HTTP upgrade.

The client asks:

> Can we upgrade this connection from HTTP to WebSocket?

---

# 17. The WebSocket Handshake

The client sends an HTTP request:

```http
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13
```

The important headers are:

```http
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ...
Sec-WebSocket-Version: 13
```

---

# 18. HTTP Upgrade

The:

```http
Upgrade: websocket
```

header requests switching to the WebSocket protocol.

The:

```http
Connection: Upgrade
```

header indicates that the connection is being upgraded.

The server can accept the upgrade.

---

# 19. `101 Switching Protocols`

If the server accepts:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: ...
```

The `101` status means:

> The server agrees to switch protocols.

The communication then transitions from HTTP to WebSocket.

```text
HTTP
 │
 │ HTTP Request
 ▼
Server
 │
 │ 101 Switching Protocols
 ▼
WebSocket
 │
 ├──── message ────→
 │←──── message ────
 ├──── message ────→
 │←──── message ────
```

---

# 20. Handshake Headers

## `Sec-WebSocket-Key`

The client generates a random value.

The server uses it when generating `Sec-WebSocket-Accept`.

This helps the client verify that the server understood the WebSocket handshake.

---

## `Sec-WebSocket-Accept`

The server generates this value from the client's key using the WebSocket protocol's defined transformation.

The client verifies it.

This is not a password or authentication token.

---

## `Sec-WebSocket-Version`

Specifies the WebSocket protocol version.

The commonly used version is:

```text
13
```

---

# 21. What Happens If the Handshake Fails?

The server can reject the handshake.

For example:

```text
401 Unauthorized
403 Forbidden
400 Bad Request
```

Possible reasons include:

* Invalid authentication
* Invalid WebSocket headers
* Unsupported version
* Origin rejection
* Server policy
* Rate limiting

If the upgrade does not succeed, the WebSocket connection is not established.

---

# 22. WebSocket Over TCP

The WebSocket protocol runs over TCP.

Conceptually:

```text
Application
     ↓
WebSocket
     ↓
TCP
     ↓
IP
```

TCP provides:

* Reliable delivery
* Ordered delivery
* Connection-oriented communication
* Retransmission of lost TCP segments

WebSocket uses these properties rather than implementing its own transport reliability.

---

# 23. WebSocket Over TLS

With:

```text
wss://
```

TLS protects the WebSocket communication.

```text
Application
     ↓
WebSocket
     ↓
TLS
     ↓
TCP
     ↓
IP
```

TLS provides encryption and server authentication.

---

# 24. Connection Lifecycle

A WebSocket connection has a lifecycle.

```text
CONNECTING
     ↓
OPEN
     ↓
CLOSING
     ↓
CLOSED
```

---

## 24.1 Connecting

The client starts the connection.

```text
Client
   ↓
HTTP Upgrade Request
```

---

## 24.2 Open

The server responds with:

```text
101 Switching Protocols
```

The WebSocket connection is now established.

---

## 24.3 Communication

Both sides can send messages.

```text
Client ─────→ Server
Client ←───── Server
Client ─────→ Server
Client ←───── Server
```

---

## 24.4 Closing

Either side can initiate a close.

A graceful WebSocket shutdown uses a close control frame.

---

## 24.5 Unexpected Disconnect

Connections can disappear unexpectedly because of:

* Network failures
* Server crashes
* Device changes network
* Browser closes
* Proxy timeouts
* Infrastructure failures

Applications need to handle these cases.

---

# 25. WebSocket Messages and Frames

This distinction is important.

A **message** is logical application data.

A **frame** is a protocol-level unit used to transport that data.

Conceptually:

```text
Message
   ↓
one or more
   ↓
Frames
```

---

# 26. WebSocket Frame Structure

A WebSocket frame contains fields such as:

```text
FIN
RSV
Opcode
MASK
Payload Length
Masking Key
Payload
```

The workshop should explain the purpose of these fields without requiring participants to memorize every bit.

---

# 27. FIN

The FIN bit indicates whether the current frame is the final frame of the message.

This becomes relevant when a message is fragmented across multiple frames.

---

# 28. Opcodes

The opcode tells the receiver what kind of frame it is.

Common categories include:

```text
Text
Binary
Close
Ping
Pong
```

---

# 29. Text Frames

Text frames carry UTF-8 encoded text.

For example:

```json
{
  "type": "message",
  "content": "Hello"
}
```

JSON is commonly used for application-level WebSocket messages.

---

# 30. Binary Frames

Binary frames allow arbitrary binary data.

Useful for:

* Images
* Audio
* Files
* Binary protocols
* High-performance applications

---

# 31. Control Frames

WebSocket defines control frames for connection management.

The important ones are:

```text
Ping
Pong
Close
```

---

# 32. Ping / Pong

Ping and Pong are WebSocket control frames used for connection liveness.

```text
Server → Ping
Client → Pong
```

Many WebSocket libraries handle protocol-level Ping/Pong automatically.

Do not confuse protocol-level Ping/Pong with application-level heartbeat messages.

For example, an application might send:

```json
{
  "type": "heartbeat"
}
```

That is application logic, not a WebSocket Ping frame.

---

# 33. Close Frames

A WebSocket connection can close gracefully using a Close frame.

A close frame can contain:

* A status code
* A reason

The close process allows both sides to understand that the connection is shutting down.

---

# 34. Masking

WebSocket clients mask frames they send to servers.

The server generally does not mask frames sent to the client.

Masking helps prevent certain intermediary/proxy attacks where an intermediary could accidentally interpret WebSocket traffic as another protocol.

The important workshop takeaway:

> Client-to-server WebSocket frames are masked; server-to-client frames generally are not.

---

# 35. Fragmentation

A WebSocket message can be split across multiple frames.

For example:

```text
Message
  ↓
Frame 1
Frame 2
Frame 3
```

The FIN bit indicates whether a frame completes the message.

The receiver reassembles the fragments into the logical message.

This is mostly a protocol-level concept and does not need a deep implementation demo.

---

# 36. Communication Patterns

WebSocket gives us a communication channel.

Applications then build communication patterns on top of it.

---

# 37. One-to-One Communication

```text
Client A
   │
   │ message
   ▼
Server
   │
   │ message
   ▼
Client B
```

For example:

Alice sends a private message to Bob.

---

# 38. Broadcasting

Broadcasting means sending an event to multiple connected clients.

```text
             Server
            /  |  \
           ↓   ↓   ↓
          A    B    C
```

Important:

> Broadcasting is an application-level mechanism, not a special WebSocket protocol feature.

The server decides which clients should receive the message.

---

# 39. Rooms

Rooms group clients into logical groups.

```text
Room A
├── Alice
├── Bob
└── Charlie

Room B
├── David
└── Eve
```

A message sent to Room A should only reach:

```text
Alice
Bob
Charlie
```

Rooms are an application-level abstraction often provided by WebSocket frameworks.

---

# 40. Topics / Subscriptions

Another pattern is topic-based communication.

For example:

```text
subscribe: stock:AAPL
```

Clients subscribed to:

```text
stock:AAPL
```

receive AAPL-related events.

This pattern is common in:

* Notifications
* Financial dashboards
* IoT
* Event-driven applications

---

# 41. Presence

WebSocket applications can implement presence information such as:

```text
Online
Offline
Last seen
Typing
Away
```

For example:

```text
Alice is typing...
```

can be represented as an application-level event:

```json
{
  "type": "typing.started",
  "userId": "alice"
}
```

---

# 42. Message Protocol Design

WebSocket itself does not define what the application's messages mean.

The application needs its own message protocol.

---

# 43. Why Define a Message Protocol?

Without a defined structure, messages quickly become inconsistent.

Instead of:

```text
"hello"
```

we can define:

```json
{
  "type": "chat.message",
  "payload": {
    "text": "Hello"
  }
}
```

Now the server knows what kind of event it received.

---

# 44. Message Types

Examples:

```text
chat.message
user.joined
user.left
typing.started
typing.stopped
room.joined
room.left
error
```

---

# 45. Payload Design

A consistent message structure makes systems easier to maintain.

Example:

```json
{
  "type": "chat.message",
  "id": "msg_123",
  "timestamp": "2026-08-25T18:00:00Z",
  "payload": {
    "text": "Hello"
  }
}
```

Useful fields can include:

* Message ID
* Type
* Timestamp
* Payload
* User ID
* Room ID

---

# 46. JSON vs Binary

JSON:

* Human-readable
* Easy to debug
* Easy to implement
* Common for business applications

Binary:

* More compact
* Useful for high-throughput systems
* Useful for binary data

For a normal chat application, JSON is usually sufficient.

---

# 47. Message Validation

Never trust incoming WebSocket messages.

A connection being authenticated does not make its messages trustworthy.

Use:

```text
Message
   ↓
Parse
   ↓
Validate
   ↓
Authorize
   ↓
Process
```

For example, validate:

* Required fields
* Types
* Message size
* Allowed operations
* User permissions

---

# 48. Schema Validation

A schema can define what a valid message looks like.

Example concept:

```json
{
  "type": "chat.message",
  "payload": {
    "text": "string"
  }
}
```

Libraries such as Pydantic can enforce these rules in a FastAPI backend.

---

# 49. Acknowledgements

Applications may need confirmation that an operation was processed.

Example:

```json
{
  "type": "message.sent",
  "messageId": "123"
}
```

This becomes useful when reliability matters.

---

# 50. Correlation IDs

A correlation/request ID allows the client to associate a response with a previous operation.

Example:

```json
{
  "type": "get.user",
  "requestId": "abc123"
}
```

Response:

```json
{
  "type": "user.result",
  "requestId": "abc123",
  "payload": {
    "id": 42
  }
}
```

---

# 51. Error Messages

Applications should define consistent error messages.

Example:

```json
{
  "type": "error",
  "code": "INVALID_MESSAGE",
  "message": "Invalid payload"
}
```

This is better than returning arbitrary strings.

---

# 52. Authentication & Authorization

Authentication and authorization are different.

### Authentication

> Who are you?

### Authorization

> What are you allowed to do?

---

# 53. Authentication During the Handshake

A server can authenticate a client during or around the WebSocket connection establishment process.

If authentication fails, the server can reject the connection.

```text
Client
  ↓
WebSocket handshake
  ↓
Authenticate
  ↓
Valid?
 ┌──────┴──────┐
No             Yes
↓               ↓
Reject          Accept
```

---

# 54. Cookies and Sessions

Browsers can send applicable cookies during the WebSocket handshake.

For example:

```http
Cookie: sessionId=abc123
```

The server can use the session ID to identify the user.

---

# 55. Tokens / JWT

Another architecture is token-based authentication.

Conceptually:

```text
Client
   ↓
Authentication token
   ↓
WebSocket server
   ↓
Validate token
   ↓
Identify user
```

The exact way a token is transported should be chosen carefully based on the application's architecture and security requirements.

---

# 56. Authorization After Connection

Authentication does not mean the user can perform every operation.

For example:

```text
Authenticated user
       ↓
Can join public room
       ↓
Cannot join admin room
```

The server must enforce authorization for every sensitive operation.

---

# 57. Room Permissions

The server may determine whether a user can:

* Join a room
* Send messages
* Receive messages
* Remove users
* Moderate a room

The client should never be trusted to enforce these rules by itself.

---

# 58. WebSocket Security

## 58.1 `ws://` vs `wss://`

Use:

```text
wss://
```

for encrypted production communication.

---

## 58.2 TLS

TLS protects the communication against network-level interception.

---

## 58.3 Origin Validation

The server can validate the browser's Origin to help prevent unauthorized websites from establishing connections using a user's credentials.

---

## 58.4 Cross-Site WebSocket Hijacking

If authentication relies on cookies, a malicious website could potentially attempt to establish a WebSocket connection using the victim's existing credentials.

Defenses can include:

* Origin validation
* Proper authentication
* Authorization
* Secure cookie configuration

---

## 58.5 Input Validation

Never trust client input.

Validate every message.

---

## 58.6 Message Size Limits

Large messages can consume excessive memory or processing resources.

Servers should enforce reasonable limits.

---

## 58.7 Rate Limiting

A malicious client could send huge numbers of messages.

Rate limiting can protect the application.

---

## 58.8 Connection Limits

Servers should consider limits on:

* Connections per user
* Connections per IP
* Total connections
* Resources consumed per connection

---

## 58.9 Denial of Service

WebSocket connections are long-lived.

A server handling thousands of persistent connections can consume significant resources.

Attackers may exploit:

* Connection exhaustion
* Message flooding
* Large payloads
* Reconnection storms

---

# 59. Reliability & Failure Handling

WebSocket connections are persistent, but networks are not reliable.

Connections can fail.

---

# 60. Why Connections Fail

Examples:

```text
Wi-Fi disconnect
Mobile network change
Server restart
Proxy timeout
Browser suspension
Network outage
```

---

# 61. Detecting Failure

Applications can detect failures through:

* Close events
* Error events
* Ping/Pong
* Application heartbeats
* Timeouts

---

# 62. Reconnection

A real-world WebSocket client should often attempt to reconnect after an unexpected disconnection.

```text
Disconnected
     ↓
Wait
     ↓
Reconnect
     ↓
Success?
 ┌───┴───┐
Yes      No
↓         ↓
Open     Wait again
```

---

# 63. Exponential Backoff

Instead of reconnecting continuously:

```text
connect
connect
connect
connect
connect
```

use increasing delays:

```text
1s
2s
4s
8s
16s
```

This reduces pressure on the server.

---

# 64. Message Loss

Consider:

```text
Client → Server
```

The network fails immediately afterward.

Did the server process the message?

The client may not know.

Solutions can include:

* Acknowledgements
* Message IDs
* Persistence
* Replay
* Sequence numbers

---

# 65. Duplicate Messages

Retries can produce duplicate messages.

For example:

```text
Client sends message
        ↓
Server processes it
        ↓
Network fails
        ↓
Client doesn't receive acknowledgement
        ↓
Client retries
        ↓
Server receives duplicate
```

Applications may use message IDs and idempotency to detect duplicates.

---

# 66. Scaling WebSockets

A single WebSocket server is relatively straightforward.

```text
Clients
   ↓
WebSocket Server
```

But production systems often need multiple servers.

```text
             Load Balancer
             /           \
            ↓             ↓
       Server A       Server B
```

This introduces new problems.

---

# 67. The Cross-Server Broadcasting Problem

Imagine:

```text
Alice → Server A

Bob → Server B
```

Alice sends:

```text
Hello Bob
```

Server A knows Alice's connection.

But Bob's connection exists on Server B.

How does Server A tell Server B about the message?

This is the core distributed WebSocket problem.

---

# 68. Sticky Sessions

A load balancer can use sticky sessions to keep a client connected to the same backend server.

```text
Alice
  ↓
Load Balancer
  ↓
Server A
```

Future connections from Alice are routed back to Server A.

Sticky sessions can help with connection affinity.

However:

> Sticky sessions do not solve cross-server communication.

Server A still needs a way to communicate with Server B.

---

# 69. Shared State

A WebSocket server may keep connection information in local memory.

For example:

```text
Server A
 ├── Alice
 └── Bob

Server B
 ├── Charlie
 └── David
```

This state isn't automatically shared.

Distributed systems therefore often need an external coordination mechanism.

---

# 70. Redis Pub/Sub

Redis can act as a message broker between WebSocket servers.

```text
             Redis
            /     \
           ↓       ↓
      Server A   Server B
```

Server A publishes:

```text
PUBLISH room:123 "Hello"
```

Server B subscribes:

```text
SUBSCRIBE room:123
```

The flow becomes:

```text
Alice
  ↓
Server A
  ↓
Redis
  ↓
Server B
  ↓
Bob
```

Now clients connected to different servers can participate in the same logical room.

---

# 71. Distributed Broadcasting

With Redis Pub/Sub:

```text
Client
  ↓
WebSocket Server A
  ↓
Redis
  ↓
WebSocket Server B
  ↓
Other Clients
```

This allows WebSocket instances to exchange events.

Important distinction:

> Redis Pub/Sub coordinates messages between servers; it does not replace the WebSocket connection itself.

---

# 72. NGINX & Reverse Proxy

A reverse proxy sits between clients and backend servers.

```text
Client
   ↓
NGINX
   ↓
Application
```

NGINX can handle:

* Routing
* TLS termination
* Load balancing
* Connection management

---

# 73. NGINX + WebSockets

NGINX must correctly forward the WebSocket upgrade.

Conceptually:

```text
Client
  │
  │ Upgrade: websocket
  ▼
NGINX
  │
  │ Upgrade: websocket
  ▼
WebSocket Server
```

---

# 74. NGINX Load Balancing

```text
                NGINX
              /       \
             ↓         ↓
        Server A    Server B
```

NGINX can distribute incoming connections between WebSocket servers.

When combined with Redis:

```text
                    NGINX
                   /     \
                  ↓       ↓
             WS Server A  WS Server B
                  │       │
                  └───┬───┘
                      ↓
                 Redis Pub/Sub
```

This forms a basic horizontally scalable architecture.

---

# 75. Performance Considerations

WebSocket performance depends on:

* Number of concurrent connections
* Message rate
* Message size
* CPU
* Memory
* Network bandwidth
* Broadcasting fan-out
* Redis throughput

Long-lived connections also consume resources even when they are idle.

---

# 76. Backpressure

Backpressure occurs when data is produced faster than the receiver can process it.

For example:

```text
Producer
  ↓
1000 messages/sec
  ↓
Slow Client
  ↓
Can't process fast enough
```

Without proper handling, buffers can grow and consume memory.

Applications may need:

* Queue limits
* Dropping strategies
* Rate limits
* Flow control
* Disconnecting unhealthy clients

---

# 77. Monitoring WebSocket Applications

Important metrics include:

```text
Active connections
Connection failures
Connection duration
Messages/sec
Message latency
Error rate
CPU
Memory
Redis health
```

Logging should help answer:

> What happened to this connection?

Distributed tracing can become useful when a message travels through:

```text
Client
 ↓
NGINX
 ↓
Server A
 ↓
Redis
 ↓
Server B
 ↓
Client
```

---

# 78. Common WebSocket Mistakes

## 78.1 Treating WebSockets Like HTTP Requests

A WebSocket is a long-lived connection, not a collection of independent request/response operations.

---

## 78.2 Forgetting Disconnect Handling

Connections can disappear at any time.

---

## 78.3 No Authentication

Do not assume that establishing a connection means the user is trusted.

---

## 78.4 No Authorization

A connected user should not automatically have access to every room or operation.

---

## 78.5 Trusting Client Input

Every message should be validated.

---

## 78.6 No Reconnection Strategy

Network failures are normal.

---

## 78.7 No Rate Limiting

Clients can abuse persistent connections.

---

## 78.8 Assuming One Server

Local-memory connection state becomes a problem once multiple instances exist.

---

## 78.9 Ignoring Dead Connections

Dead connections can consume resources indefinitely.

---

## 78.10 Ignoring Backpressure

Fast producers and slow consumers can exhaust memory.

---

## 78.11 Using WebSockets Everywhere

WebSockets are not automatically the best solution.

If the application only needs server-to-client events, SSE may be simpler.

If updates can tolerate delay, polling may be sufficient.

---





# 80. Workshop Summary

The complete mental model:

```text
HTTP
 │
 ├── Request / Response
 │
 ├── Polling
 │
 └── Long Polling
 │
 ▼
WebSocket
 │
 ├── HTTP Upgrade
 │
 ├── Handshake
 │
 ├── Persistent Connection
 │
 ├── Bidirectional Communication
 │
 ├── Messages
 │
 ├── Frames
 │
 ├── Ping / Pong
 │
 ├── Close
 │
 └── Application Layer
       │
       ├── Broadcasting
       ├── Rooms
       ├── Presence
       ├── Message Protocol
       ├── Authentication
       ├── Authorization
       └── Reliability
              │
              ├── Reconnection
              ├── Backoff
              └── Message Recovery
                     │
                     ▼
                   Scaling
                     │
                     ├── Load Balancer
                     ├── NGINX
                     ├── Multiple Servers
                     └── Redis Pub/Sub
```

The core idea is:

> **WebSockets provide the communication channel. The application defines what happens over that channel. Infrastructure determines how that system survives and scales.**



# Final Takeaway

WebSockets are not simply "faster HTTP."

They solve a different communication problem.

HTTP normally gives us:

```text
Client → Request
Server → Response
```

WebSockets establish a persistent communication channel:

```text
Client ←────────→ Server
```

The WebSocket protocol defines how that channel works.

The application then builds features such as:

```text
Messages
Broadcasting
Rooms
Presence
Authentication
```

And production infrastructure adds:

```text
Reconnection
Load Balancing
NGINX
Redis Pub/Sub
Horizontal Scaling
Monitoring
```

Understanding WebSockets therefore means understanding **three layers**:

```text
┌───────────────────────────────┐
│ Application                   │
│ Rooms, messages, presence     │
├───────────────────────────────┤
│ WebSocket Protocol            │
│ Handshake, frames, ping/pong  │
├───────────────────────────────┤
│ Infrastructure / Transport    │
│ TCP, TLS, NGINX, Redis        │
└───────────────────────────────┘
```

### la fin. (3yit)