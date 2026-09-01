# WhatsApp High Level Design (1-to-1 Messaging)

## Overview

This diagram shows the end-to-end flow of a **1-to-1 message** in a WhatsApp-like system.

The important idea is that:

- Every connected user has **one WebSocket connection**.
- A **WebSocket Handler** manages thousands of such connections.
- The handler **does not contain business logic**. It only receives and sends data.
- Business logic (validation, persistence, routing) is handled by the **Message Service**.

---

## Components

### 1. Client

The mobile application.

After login it establishes a **WebSocket connection** with a Chat Server.

```
Client
   │
Persistent WebSocket
   │
WS Handler
```

The connection remains open until the user disconnects.

---

### 2. WebSocket Handler

Think of this as a receptionist.

It manages many WebSocket connections simultaneously.

Example:

```
WS Handler

Max   -> Socket 1
Emily -> Socket 2
John  -> Socket 3
Alex  -> Socket 4
```

Responsibilities:

- Accept WebSocket connections
- Maintain active sockets
- Receive incoming messages
- Push outgoing messages
- Detect disconnects
- Heartbeats (Ping/Pong)

It **does not** decide where to store messages or how to route them.

---

### 3. Message Service

This is the brain of the chat application.

Responsibilities:

- Validate message
- Generate Message ID
- Store message in database
- Find recipient
- Trigger delivery
- Handle acknowledgements

---

### 4. Messages Database

Stores chat history.

Typical schema:

| Field | Description |
|------|-------------|
| MessageId | Unique message id |
| From | Sender |
| To | Receiver |
| Message | Text |
| Timestamp | Time |
| Status | SENT / DELIVERED / READ |

Messages are persisted before delivery to avoid data loss.

---

### 5. WS Connection Manager

Keeps track of which handler currently owns each user.

Example:

```
Max
   ↓
Handler 1

Emily
   ↓
Handler 2
```

---

### 6. WS Connection Cache (Redis)

Distributed cache used by all chat servers.

Stores

```
UserId

↓

Handler / Chat Server
```

Example

```
Max -> Handler 1

Emily -> Handler 2
```

Without this cache every server would need to know about every other server.

---

# Complete Message Flow

The numbers correspond to the diagram.

## Step 1

Max types

```
Hi Emily
```

The message travels through the existing WebSocket connection to **WS Handler 1**.

No HTTP request is created.

---

## Step 2

WS Handler forwards the message to the **Message Service**.

The handler is only a communication layer.

---

## Step 3

Message Service stores the message in the database.

Example

```
MessageId : M1

From : Max

To : Emily

Message : Hi Emily

Status : SENT
```

Persist first, deliver later.

---

## Step 4

The handler (or routing layer) asks the **WS Connection Manager**

```
Where is Emily connected?
```

---

## Step 5

Connection Manager checks Redis.

Example

```
Emily

↓

Handler 2
```

Now the system knows which handler owns Emily's socket.

---

## Step 6

The message is forwarded internally to **WS Handler 2**.

Notice:

Handler 1 never talks directly to Emily's phone.

It communicates with Handler 2 through backend routing.

---

## Step 7

Handler 2 writes the message to Emily's existing WebSocket.

```
Handler 2

↓

Emily Socket

↓

Emily Phone
```

The message appears instantly.

---

# What if Emily is Offline?

Instead of Step 7:

```
Store Message

↓

Mark Pending

↓

Emily reconnects

↓

Deliver Pending Messages
```

---

# Why separate Handler and Message Service?

If handlers also contained business logic:

```
Handler

↓

Validate

↓

Store

↓

Route

↓

Notify
```

every handler would become huge and hard to scale.

Instead:

```
Handler

↓

Message Service

↓

Database

↓

Routing
```

Handlers remain lightweight.

---

# Interview Points

Mention these points:

- WebSocket is a long-lived bidirectional TCP connection.
- One handler manages thousands of sockets.
- One socket belongs to one connected user.
- Redis maps User → Handler.
- Message Service contains business logic.
- Database persistence happens before delivery.
- Offline users receive messages after reconnecting.
- Delivery/read acknowledgements travel back over WebSocket.

---

# Mental Model

Imagine a telephone exchange.

- Phone line = WebSocket connection
- Telephone operator = WebSocket Handler
- Operator directory = Redis Connection Cache
- Messaging department = Message Service
- Filing cabinet = Messages Database

The operator receives the call, looks up the recipient's line, and connects the message to the correct phone.
