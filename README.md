# Bun WebSocket Router

Tired of wrestling WebSocket connections like a tangled mess of holiday lights? `bun-ws-router` is here to bring order to the chaos! It's a type-safe WebSocket router for Bun, powered by Zod, making your real-time message handling smoother than a fresh jar of peanut butter. Route WebSocket messages to handlers based on message type with full TypeScript support.

### Key Features

- 🔒 **Type-safe messaging**: Built-in validation using Zod schemas (catch errors before they bite!)
- ✨ **Simple API**: Intuitive routing system for WebSocket messages (so easy, your cat could _almost_ use it)
- 🚀 **Performance**: Leverages Bun's native WebSocket implementation (it's fast, like, _really_ fast)
- 🧩 **Flexible**: Works with any Bun server setup, including Hono
- 🪶 **Lightweight**: Minimal dependencies for fast startup

Perfect for real-time applications like chat systems, live dashboards, multiplayer games, and notification services.

## Installation

```bash
bun add bun-ws-router zod
bun add @types/bun -D
```

## Getting Started

The following example demonstrates how to set up a Bun server with both (RESTful) HTTP and WebSocket routers.

```ts
import { Hono } from "hono";
import { WebSocketRouter } from "bun-ws-router";
import { exampleRouter } from "./example";

// HTTP router
const app = new Hono();
app.get("/", (c) => c.text("Welcome to Hono!"));

// WebSocket router
const ws = new WebSocketRouter();
ws.addRoutes(exampleRouter); // Add routes from another file

Bun.serve({
  port: 3000,

  fetch(req, server) {
    const url = new URL(req.url);

    // Handle WebSocket upgrade requests
    if (url.pathname === "/ws") {
      return ws.upgrade(req, {
        server,
      });
    }

    // Handle regular HTTP requests
    return app.fetch(req, { server });
  },

  // Handle WebSocket connections
  websocket: ws.websocket,
});

console.log(`WebSocket server listening on ws://localhost:3000/ws`);
```

## How to handle authentication

You can handle authentication by checking the `Authorization` header for a JWT token or any other authentication method you prefer. The following example demonstrates how to verify a JWT token and pass the user information to the WebSocket router.

```ts
import { WebSocketRouter } from "bun-ws-router";
import { DecodedIdToken } from "firebase-admin/auth";
import { verifyIdToken } from "./auth"; // Your authentication logic

type Meta = {
  user?: DecodedIdToken | null;
};

// WebSocket router
const ws = new WebSocketRouter<Meta>();

Bun.serve({
  port: 3000,

  async fetch(req, server) {
    const url = new URL(req.url);

    // Check if the user is authenticated
    const user = await verifyToken(req);

    // Handle WebSocket upgrade requests
    if (url.pathname === "/ws") {
      return ws.upgrade(req, {
        server,
        data: { user },
      });
    }

    // Handle regular HTTP requests
    return await app.fetch(req, { server, user });
  },

  // Handle WebSocket connections
  websocket: ws.websocket,
});
```

The `verifyIdToken` function is a placeholder for your authentication logic which could use user ID token verification from `firebase-admin` or any other authentication library.

By verifying the user _before_ the WebSocket connection is fully established and passing the `user` data, you ensure that only authenticated users can connect, and you have their info ready to use in your `onOpen`, `onMessage`, and `onClose` handlers without needing to ask again. Secure _and_ convenient!

## How to define message types

You can define message types using the `messageSchema` function from `bun-ws-router`. This function takes a message type name such as `JOIN_ROOM`, `SEND_MESSAGE` etc. and a Zod schema for the message payload. The following example demonstrates how to define message types for a chat application.

```ts
import { messageSchema } from "bun-ws-router";
import { z } from "zod";

export const JoinRoom = messageSchema("JOIN_ROOM", {
  roomId: z.string(),
});

export const UserJoined = messageSchema("USER_JOINED", {
  roomId: z.string(),
  userId: z.string(),
});

export const UserLeft = messageSchema("USER_LEFT", {
  userId: z.string(),
});

export const SendMessage = messageSchema("SEND_MESSAGE", {
  roomId: z.string(),
  message: z.string(),
});
```

## How to define routes

You can define routes using the `WebSocketRouter` instance methods: `onOpen`, `onMessage`, and `onClose`.

```ts
import { WebSocketRouter } from "bun-ws-router";
import { Meta, JoinRoom, UserJoined, SendMessage, UserLeft } from "./schema";

const ws = new WebSocketRouter<Meta>();

// Handle new connections
ws.onOpen((c) => {
  console.log(
    `Client connected: ${c.ws.data.clientId}, User ID: ${c.ws.data.userId}`,
  );
  // You could send a welcome message here
});

// Handle specific message types
ws.onMessage(JoinRoom, (c) => {
  const { roomId } = c.payload;
  const userId = c.ws.data.userId || c.ws.data.clientId; // Use userId if available, else clientId
  c.ws.data.roomId = roomId; // Store room in connection data
  console.log(`User ${userId} joining room: ${roomId}`);

  // Example: Send confirmation back or broadcast to room
  // This requires implementing broadcast/room logic separately
  // c.send(UserJoined, { roomId, userId });
});

ws.onMessage(SendMessage, (c) => {
  const { message } = c.payload;
  const userId = c.ws.data.userId || c.ws.data.clientId;
  const roomId = c.ws.data.roomId;
  console.log(`Message in room ${roomId} from ${userId}: ${message}`);
  // Add logic to broadcast message to others in the room
});

// Handle disconnections
ws.onClose((c) => {
  const userId = c.ws.data.userId || c.ws.data.clientId;
  console.log(`Client disconnected: ${userId}, code: ${c.code}`);
  // Example: Notify others in the room the user left
  // This requires implementing broadcast/room logic separately
  // broadcast(c.ws.data.roomId, UserLeft, { userId });
});
```

**Important Note:** The `c.send(...)` function sends a message back _only_ to the client that sent the original message. For broadcasting to rooms or multiple clients, you'll need to implement your own logic (we handle the routing, you handle the party!).

## How to broadcast messages

Broadcasting messages to multiple clients is a common requirement for real-time applications. `bun-ws-router` complements Bun's built-in PubSub functionality with schema validation support.

### Option 1: Using Bun's native WebSocket PubSub

Bun's WebSocket implementation includes built-in support for the PubSub pattern through `subscribe`, `publish`, and `unsubscribe` methods:

```ts
ws.onMessage(JoinRoom, (c) => {
  const { roomId } = c.payload;
  const userId = c.ws.data.userId || c.ws.data.clientId;

  // Store room ID in connection data
  c.ws.data.roomId = roomId;

  // Subscribe the client to the room's topic
  c.ws.subscribe(roomId);

  console.log(`User ${userId} joined room: ${roomId}`);

  // Send confirmation back to the user who joined
  c.send(UserJoined, { roomId, userId });

  // Broadcast to all other subscribers that a new user joined
  const message = JSON.stringify({
    type: "USER_JOINED",
    meta: { timestamp: Date.now() },
    payload: { roomId, userId },
  });
  c.ws.publish(roomId, message);
});

ws.onClose((c) => {
  const userId = c.ws.data.userId || c.ws.data.clientId;
  const roomId = c.ws.data.roomId;

  if (roomId) {
    // Unsubscribe from room
    c.ws.unsubscribe(roomId);

    // Notify others the user has left
    const message = JSON.stringify({
      type: "USER_LEFT",
      meta: { timestamp: Date.now() },
      payload: { userId },
    });
    c.ws.publish(roomId, message);
  }
});
```

### Option 2: Using the publish helper function

The library provides a helper function that combines schema validation with publishing:

```ts
import { WebSocketRouter, publish } from "bun-ws-router";
import { JoinRoom, UserJoined, SendMessage, UserLeft } from "./schema";

ws.onMessage(JoinRoom, (c) => {
  const { roomId } = c.payload;
  const userId = c.ws.data.userId || c.ws.data.clientId;

  // Store room ID and subscribe to topic
  c.ws.data.roomId = roomId;
  c.ws.subscribe(roomId);

  // Send confirmation back to the user who joined
  c.send(UserJoined, { roomId, userId });

  // Broadcast to other subscribers with schema validation
  publish(c.ws, roomId, UserJoined, {
    roomId,
    userId,
  });
});

ws.onMessage(SendMessage, (c) => {
  const { roomId, message } = c.payload;
  const userId = c.ws.data.userId || c.ws.data.clientId;

  console.log(`Message in room ${roomId} from ${userId}: ${message}`);

  // Broadcast the message to all subscribers in the room
  publish(c.ws, roomId, NewMessage, {
    roomId,
    userId,
    message,
    timestamp: Date.now(),
  });
});

ws.onClose((c) => {
  const userId = c.ws.data.userId || c.ws.data.clientId;
  const roomId = c.ws.data.roomId;

  if (roomId) {
    c.ws.unsubscribe(roomId);

    // Notify others using the publish helper
    publish(c.ws, roomId, UserLeft, { userId });
  }
});
```

The `publish()` function ensures that all broadcast messages are validated against their schemas before being sent, providing the same type safety for broadcasts that you get with direct messaging.

## Error handling and sending error messages

Effective error handling is crucial for maintaining robust WebSocket connections. `bun-ws-router` provides built-in tools for standardized error messages that align with the library's schema validation pattern.

### Using ErrorCode and ErrorMessage schema

The library includes a standardized error schema and predefined error codes:

```ts
import { ErrorMessage, ErrorCode } from "bun-ws-router";

ws.onMessage(JoinRoom, (c) => {
  const { roomId } = c.payload;

  // Check if room exists
  const roomExists = await checkRoomExists(roomId);
  if (!roomExists) {
    // Send error with standardized code
    c.send(ErrorMessage, {
      code: ErrorCode.RESOURCE_NOT_FOUND,
      message: `Room ${roomId} does not exist`,
      context: { roomId }, // Optional context for debugging
    });
    return;
  }

  // Continue with normal flow...
  c.ws.data.roomId = roomId;
  c.ws.subscribe(roomId);
  // ...
});
```

### Custom error handling

You can add your own error handling middleware by using the `onMessage` handler:

```ts
// Catch-all handler for message processing errors
ws.onOpen((c) => {
  try {
    // Your connection setup logic
  } catch (error) {
    console.error("Error in connection setup:", error);
    c.send(ErrorMessage, {
      code: ErrorCode.INTERNAL_SERVER_ERROR,
      message: "Failed to set up connection",
    });
  }
});

// Global error handler for authentication
ws.onMessage(AuthenticateUser, (c) => {
  try {
    // Validate token
    const { token } = c.payload;
    const user = validateToken(token);

    if (!user) {
      c.send(ErrorMessage, {
        code: ErrorCode.AUTHENTICATION_FAILED,
        message: "Invalid authentication token",
      });
      return;
    }

    // Store user data for future requests
    c.ws.data.userId = user.id;
    c.ws.data.userRole = user.role;

    // Send success response
    c.send(AuthSuccess, { userId: user.id });
  } catch (error) {
    c.send(ErrorMessage, {
      code: ErrorCode.INTERNAL_SERVER_ERROR,
      message: "Authentication process failed",
    });
  }
});
```

### Available Error Codes

The built-in `ErrorCode` enum provides consistent error types:

| Error Code                 | Description                                           |
| -------------------------- | ----------------------------------------------------- |
| `INVALID_MESSAGE_FORMAT`   | Message isn't valid JSON or lacks required structure  |
| `VALIDATION_FAILED`        | Message failed schema validation                      |
| `UNSUPPORTED_MESSAGE_TYPE` | No handler registered for this message type           |
| `AUTHENTICATION_FAILED`    | Client isn't authenticated or has invalid credentials |
| `AUTHORIZATION_FAILED`     | Client lacks permission for the requested action      |
| `RESOURCE_NOT_FOUND`       | Requested resource (user, room, etc.) doesn't exist   |
| `RATE_LIMIT_EXCEEDED`      | Client is sending messages too frequently             |
| `INTERNAL_SERVER_ERROR`    | Unexpected server error occurred                      |

You can also broadcast error messages to multiple clients using the `publish` function:

```ts
// Notify all users in a room that it's being deleted
publish(c.ws, roomId, ErrorMessage, {
  code: ErrorCode.RESOURCE_NOT_FOUND,
  message: "This room is being deleted and will no longer be available",
  context: { roomId },
});
```

## How to compose routes

You can compose routes from different files into a single router. This is useful for organizing your code and keeping related routes together.

```ts
import { WebSocketRouter } from "bun-ws-router";
import { Meta } from "./schemas";
import { chatRoutes } from "./chat";
import { notificationRoutes } from "./notification";

const ws = new WebSocketRouter<Meta>();
ws.addRoutes(chatRoutes);
ws.addRoutes(notificationRoutes);
```

Where `chatRoutes` and `notificationRoutes` are other router instances defined in separate files.

## Support

Got questions? Hit a snag? Or just want to share your awesome WebSocket creation? Find us on [Discord](https://discord.com/invite/bSsv7XM). We promise we don't bite (usually 😉).

## Backers

<a href="https://reactstarter.com/b/1"><img src="https://reactstarter.com/b/1.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/2"><img src="https://reactstarter.com/b/2.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/3"><img src="https://reactstarter.com/b/3.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/4"><img src="https://reactstarter.com/b/4.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/5"><img src="https://reactstarter.com/b/5.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/6"><img src="https://reactstarter.com/b/6.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/7"><img src="https://reactstarter.com/b/7.png" height="60" /></a>&nbsp;&nbsp;<a href="https://reactstarter.com/b/8"><img src="https://reactstarter.com/b/8.png" height="60" /></a>

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
