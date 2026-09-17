# WebSocket Challenge : Real-Time Chat Room

## Objective

Build a small real-time chat application using WebSockets.

The goal is to practice the concepts covered during the workshop:

* WebSocket connection
* Connection lifecycle
* Sending and receiving messages
* Broadcasting
* Rooms
* Message protocols
* Connection status
* Disconnection handling

---

## Requirements

### 1. WebSocket Connection

Create a WebSocket endpoint and allow clients to connect to the server.

The client should display its connection state:

```text
🟢 Connected
🟡 Connecting...
🔴 Disconnected
```

---

### 2. User Identity

When connecting, each client provides a username.

Example:

```text
Alice connected
Bob connected
```

The server should associate the WebSocket connection with the user.

---

### 3. Message Protocol

Messages must follow a defined structure.

For example:

```json
{
  "type": "chat.message",
  "data": {
    "message": "Hello everyone!"
  }
}
```

At minimum, support:

```text
chat.message
user.joined
user.left
error
```

---

### 4. Broadcasting

When a user sends a message, broadcast it to the other users in the same room.

Example:

```text
Alice → "Hello!"

              Server
             /      \
            ↓        ↓
          Bob      Charlie
```

The sender should also see their own message.

---

### 5. Rooms

Users must be able to join a room.

Example:

```text
Room: developers

Alice
Bob
Charlie

Room: gaming

David
Emma
```

A message sent in `developers` must **not** appear in `gaming`.

---

### 6. Join / Leave Events

When someone joins a room:

```text
System:
Alice joined the room.
```

When they disconnect:

```text
System:
Alice left the room.
```

---

### 7. Connection Handling

The application must correctly handle:

* New connections
* Normal disconnections
* Unexpected disconnections
* Invalid messages
* Empty messages

The server should not crash when a client disconnects unexpectedly.

---

### 8. Reconnection

If the connection is lost, the client should attempt to reconnect.

Use a simple delay strategy:

```text
1 second
   ↓
2 seconds
   ↓
4 seconds
   ↓
8 seconds
```

Stop retrying after a reasonable limit or allow the user to retry manually.

---

# Bonus Challenges

These are optional for participants who want to go further.

## Bonus 1 : Typing Indicator

Implement:

```text
Alice is typing...
```

Use WebSocket events instead of HTTP requests.

Example:

```json
{
  "type": "typing.start"
}
```

and:

```json
{
  "type": "typing.stop"
}
```

---

## Bonus 2 : Authentication

Require users to authenticate before joining a room.

The server must reject unauthorized clients.

---

## Bonus 3 : Multiple Server Instances

Run two WebSocket server instances.

```text
Client A
   ↓
Server A

Client B
   ↓
Server B
```

Use Redis Pub/Sub so that messages can travel between users connected to different servers.

---

# Questions to Think About

Before considering the challenge complete, be able to answer:

1. Why are WebSockets useful for this application?
2. What happens during the WebSocket handshake?
3. What happens when a client disconnects unexpectedly?
4. How does the server know which clients belong to a room?
5. Why isn't a room a native WebSocket protocol feature?
6. What happens if Alice is connected to Server A and Bob is connected to Server B?
7. Why would Redis Pub/Sub help?
8. Why should the server validate messages instead of trusting the client?
9. What happens if the client reconnects?
10. What could happen if thousands of clients reconnect simultaneously?

---

# Expected Result

By completing the challenge, you should have a working real-time application demonstrating:

```text
              WebSocket
                  │
                  ▼
             Connection
                  │
                  ▼
             User joins
                  │
                  ▼
                Room
                  │
                  ▼
             Send message
                  │
                  ▼
             Server validates
                  │
                  ▼
             Broadcasting
             /     |      \
            /      |       \
         User A  User B   User C
                  │
                  ▼
             Disconnect
                  │
                  ▼
              Reconnect
```

* ### the implementation language/framework is not strictly required. 
* ###  use technologies such as FastAPI, Node.js, or another WebSocket-compatible backend. 
* ### check the workshop resources and materials before working on this.
## best of luck !!
