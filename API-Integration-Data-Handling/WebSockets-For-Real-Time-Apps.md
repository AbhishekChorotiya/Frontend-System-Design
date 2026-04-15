# WebSockets for Real-Time Apps

WebSockets provide a persistent, full-duplex communication channel between a client (typically a browser) and a server over a single TCP connection. Unlike the traditional HTTP request-response model, where the client must initiate every exchange, WebSockets allow both parties to send data independently at any time. This makes them the foundation for real-time features like chat applications, live dashboards, collaborative editing, multiplayer games, and financial tickers -- scenarios where data must flow instantly without the overhead of repeated HTTP handshakes.

For frontend system design, understanding WebSockets is essential because real-time communication fundamentally changes how you architect state management, handle network resilience, and design user experiences. A poorly managed WebSocket connection can lead to memory leaks, stale data, dropped messages, and degraded performance. A well-designed one can deliver sub-100ms updates to thousands of concurrent users while gracefully handling disconnections and server failures.

> **Think of it like a phone call.** HTTP is like sending letters back and forth -- you write a request, mail it, wait for a response, and repeat. WebSockets are like picking up the phone and keeping the line open. Either side can speak at any time, the conversation flows naturally, and there's no need to redial for every exchange. The tradeoff is that an open phone line consumes resources for as long as it's connected, so you need to manage those connections carefully.

## Core Concepts

1. **WebSocket Protocol:** WebSockets use the `ws://` (or `wss://` for encrypted) protocol, defined in RFC 6455. The connection starts as an HTTP request and is then *upgraded* to a persistent WebSocket connection. Once established, data flows as lightweight frames rather than full HTTP requests, dramatically reducing overhead.

2. **Full-Duplex Communication:** Unlike HTTP (half-duplex), WebSockets allow simultaneous bidirectional data transfer. The server can push data to the client without being asked, and the client can send messages to the server at any time -- both over the same connection.

3. **Connection Lifecycle:** Every WebSocket connection passes through a series of events:
    - `open` -- The connection is established and ready for communication.
    - `message` -- Data is received from the other side.
    - `error` -- An error occurred on the connection.
    - `close` -- The connection has been terminated (by either side or due to a network failure).

4. **Heartbeat / Ping-Pong:** WebSocket connections can silently die (e.g., a mobile user enters a tunnel). Heartbeats are periodic ping/pong frames exchanged between client and server to detect dead connections. If a pong is not received within a timeout, the connection is considered lost and reconnection logic kicks in.

5. **Reconnection Strategies:** Network interruptions are inevitable. Robust WebSocket implementations include automatic reconnection with *exponential backoff* -- waiting progressively longer between retry attempts (e.g., 1s, 2s, 4s, 8s) to avoid overwhelming the server. A maximum retry limit or a cap on the backoff interval prevents infinite loops.

6. **Message Framing:** WebSocket data is transmitted in *frames* -- small packets that can carry text (UTF-8) or binary data. A single logical message may span multiple frames. The protocol handles framing transparently, but understanding it matters when designing message serialization (JSON, Protocol Buffers, MessagePack) and managing payload sizes.

## How It Works

### The WebSocket Handshake

A WebSocket connection begins with an HTTP `GET` request that includes an `Upgrade` header. The server responds with a `101 Switching Protocols` status, and the connection is upgraded from HTTP to WebSocket.

```
Client Request:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After this handshake, the TCP connection remains open and both sides communicate via WebSocket frames.

### Connection Lifecycle Diagram

```
Client                                    Server
  |                                         |
  |  --- HTTP GET (Upgrade: websocket) ---> |
  |  <-- 101 Switching Protocols ---------- |
  |                                         |
  |  ========= Connection Open =========== |
  |                                         |
  |  --- message (JSON/binary) ----------> |
  |  <-- message (JSON/binary) ----------- |
  |  <-- message (server push) ----------- |
  |  --- message (client event) ----------> |
  |                                         |
  |  --- ping -----------------------------> |
  |  <-- pong ----------------------------- |
  |                                         |
  |  --- close (code 1000) ---------------> |
  |  <-- close (acknowledge) -------------- |
  |                                         |
  |  ========= Connection Closed ========= |
```

### Basic WebSocket Connection

```javascript
// src/services/websocket-basic.js

const socket = new WebSocket('wss://api.example.com/ws');

socket.addEventListener('open', (event) => {
  console.log('Connected to WebSocket server');
  socket.send(JSON.stringify({ type: 'subscribe', channel: 'notifications' }));
});

socket.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
});

socket.addEventListener('error', (event) => {
  console.error('WebSocket error:', event);
});

socket.addEventListener('close', (event) => {
  console.log(`Connection closed: code=${event.code}, reason=${event.reason}`);
  if (!event.wasClean) {
    console.warn('Connection lost unexpectedly, attempting reconnect...');
  }
});
```

## WebSocket vs HTTP Polling vs SSE

Choosing the right real-time communication strategy depends on the direction of data flow, latency requirements, and infrastructure complexity. Here is how the three main approaches compare:

| Feature | **Short Polling** | **Long Polling** | **SSE** | **WebSockets** |
|---|---|---|---|---|
| **Direction** | Client → Server | Client → Server | Server → Client | Bidirectional |
| **Connection** | New request each cycle | Held open until response | Persistent HTTP | Persistent TCP |
| **Latency** | High (poll interval) | Medium (wait for data) | Low | Very low |
| **Overhead** | High (repeated headers) | Medium | Low | Very low (after handshake) |
| **Browser Support** | Universal | Universal | All modern (no IE) | All modern |
| **Protocol** | HTTP | HTTP | HTTP | ws:// / wss:// |
| **Reconnection** | Built-in (next poll) | Built-in (next poll) | Auto-reconnect built-in | Manual implementation |
| **Binary Data** | Yes (with encoding) | Yes (with encoding) | No (text only) | Yes (native) |
| **Scalability** | Poor | Moderate | Good | Good (with infra) |
| **Complexity** | Very low | Low | Low | Medium-High |
| **Best For** | Infrequent updates | Near-real-time reads | Live feeds, notifications | Chat, gaming, collaboration |

### When to Use Each

- **Short Polling:** Acceptable when updates are infrequent (every 30-60 seconds) and simplicity is paramount. Example: checking for new emails every minute.
- **Long Polling:** A step up when you need near-real-time updates but cannot use WebSockets (e.g., behind restrictive proxies). The server holds the connection open until data is available.
- **Server-Sent Events (SSE):** Ideal for unidirectional server-to-client streaming such as live news feeds, stock tickers, or notification streams. SSE auto-reconnects and supports event IDs for resumption.
- **WebSockets:** The right choice when you need true bidirectional, low-latency communication -- chat apps, collaborative editors, live gaming, or any scenario where both client and server initiate messages frequently.

```typescript
// src/services/polling-vs-websocket.ts

// ❌ Bad: Polling for real-time chat messages
const pollMessages = () => {
  setInterval(async () => {
    const response = await fetch('/api/messages?since=' + lastTimestamp);
    const messages = await response.json();
    updateUI(messages);
  }, 1000); // Wastes bandwidth even when no new messages exist
};

// ✅ Good: WebSocket for real-time chat messages
const connectChat = () => {
  const ws = new WebSocket('wss://api.example.com/chat');
  ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    updateUI([message]); // Instant delivery, zero wasted requests
  };
};
```

## Real-Time Architecture Patterns

### Pub/Sub (Publish-Subscribe)

The pub/sub pattern decouples message producers from consumers. Clients subscribe to *topics* or *channels*, and the server broadcasts messages to all subscribers of a given topic. This is the most common pattern in WebSocket architectures.

```typescript
// src/services/pubsub-client.ts

type EventHandler<T = unknown> = (data: T) => void;

interface ChatMessage {
  userId: string;
  text: string;
  timestamp: number;
}

interface PresenceUpdate {
  userId: string;
  status: 'online' | 'offline';
}

type EventMap = {
  'chat:message': ChatMessage;
  'presence:update': PresenceUpdate;
  'typing:start': { userId: string };
  'typing:stop': { userId: string };
};

class WebSocketPubSub {
  private ws: WebSocket | null = null;
  private listeners: Map<string, Set<EventHandler>> = new Map();

  connect(url: string): void {
    this.ws = new WebSocket(url);

    this.ws.onmessage = (event) => {
      const { type, payload } = JSON.parse(event.data);
      const handlers = this.listeners.get(type);
      if (handlers) {
        handlers.forEach((handler) => handler(payload));
      }
    };
  }

  subscribe<K extends keyof EventMap>(
    event: K,
    handler: EventHandler<EventMap[K]>
  ): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler as EventHandler);

    // Send subscription intent to server
    this.ws?.send(JSON.stringify({ action: 'subscribe', channel: event }));

    // Return unsubscribe function
    return () => {
      this.listeners.get(event)?.delete(handler as EventHandler);
      this.ws?.send(JSON.stringify({ action: 'unsubscribe', channel: event }));
    };
  }

  publish<K extends keyof EventMap>(event: K, data: EventMap[K]): void {
    this.ws?.send(JSON.stringify({ type: event, payload: data }));
  }
}

// Usage
const pubsub = new WebSocketPubSub();
pubsub.connect('wss://api.example.com/ws');

const unsubscribe = pubsub.subscribe('chat:message', (message) => {
  console.log(`${message.userId}: ${message.text}`);
});

pubsub.publish('chat:message', {
  userId: 'user-123',
  text: 'Hello, world!',
  timestamp: Date.now(),
});
```

### Rooms and Channels

Rooms allow segmentation of connections. Instead of broadcasting to all connected clients, messages are scoped to a specific room (e.g., a chat room, a document, or a game lobby). Clients join and leave rooms dynamically.

```typescript
// src/services/room-manager.ts

class RoomManager {
  private currentRooms: Set<string> = new Set();

  constructor(private ws: WebSocket) {}

  join(roomId: string): void {
    this.ws.send(JSON.stringify({ action: 'join', room: roomId }));
    this.currentRooms.add(roomId);
  }

  leave(roomId: string): void {
    this.ws.send(JSON.stringify({ action: 'leave', room: roomId }));
    this.currentRooms.delete(roomId);
  }

  sendToRoom(roomId: string, message: unknown): void {
    if (!this.currentRooms.has(roomId)) {
      console.warn(`Not a member of room: ${roomId}`);
      return;
    }
    this.ws.send(JSON.stringify({ action: 'message', room: roomId, payload: message }));
  }

  leaveAll(): void {
    this.currentRooms.forEach((room) => this.leave(room));
  }
}
```

### Event-Driven Architecture

In event-driven designs, every action is modeled as an event with a type and payload. The server acts as an event bus, routing events to the appropriate handlers or subscribers. This pairs naturally with frontend state management libraries (Redux, Zustand) where incoming events dispatch actions.

## Connection Management

Connection management is the most operationally critical part of a WebSocket implementation. A connection that silently dies without reconnection leads to a user staring at a frozen UI. A connection that reconnects too aggressively during an outage can DDoS your own server.

### React Hook with Reconnection Logic

```typescript
// src/hooks/useWebSocket.ts

import { useEffect, useRef, useState, useCallback } from 'react';

type ConnectionStatus = 'connecting' | 'connected' | 'disconnected' | 'reconnecting';

interface UseWebSocketOptions {
  url: string;
  onMessage?: (data: unknown) => void;
  onOpen?: () => void;
  onClose?: () => void;
  maxRetries?: number;
  maxBackoff?: number;
  protocols?: string[];
}

interface UseWebSocketReturn {
  sendMessage: (data: unknown) => void;
  status: ConnectionStatus;
  lastMessage: unknown | null;
  disconnect: () => void;
  reconnect: () => void;
}

export function useWebSocket({
  url,
  onMessage,
  onOpen,
  onClose,
  maxRetries = 10,
  maxBackoff = 30000,
  protocols,
}: UseWebSocketOptions): UseWebSocketReturn {
  const wsRef = useRef<WebSocket | null>(null);
  const retryCountRef = useRef(0);
  const retryTimeoutRef = useRef<ReturnType<typeof setTimeout>>();
  const intentionalCloseRef = useRef(false);

  const [status, setStatus] = useState<ConnectionStatus>('disconnected');
  const [lastMessage, setLastMessage] = useState<unknown | null>(null);

  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    setStatus('connecting');
    const ws = new WebSocket(url, protocols);

    ws.onopen = () => {
      setStatus('connected');
      retryCountRef.current = 0;
      onOpen?.();
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setLastMessage(data);
      onMessage?.(data);
    };

    ws.onerror = () => {
      // Error always fires before close, so let onclose handle reconnection
    };

    ws.onclose = () => {
      setStatus('disconnected');
      onClose?.();

      if (!intentionalCloseRef.current && retryCountRef.current < maxRetries) {
        const backoff = Math.min(
          1000 * Math.pow(2, retryCountRef.current) + Math.random() * 1000,
          maxBackoff
        );
        setStatus('reconnecting');
        retryTimeoutRef.current = setTimeout(() => {
          retryCountRef.current += 1;
          connect();
        }, backoff);
      }
    };

    wsRef.current = ws;
  }, [url, protocols, maxRetries, maxBackoff, onMessage, onOpen, onClose]);

  const disconnect = useCallback(() => {
    intentionalCloseRef.current = true;
    clearTimeout(retryTimeoutRef.current);
    wsRef.current?.close(1000, 'Client disconnect');
    setStatus('disconnected');
  }, []);

  const reconnect = useCallback(() => {
    intentionalCloseRef.current = false;
    retryCountRef.current = 0;
    disconnect();
    setTimeout(connect, 100);
  }, [connect, disconnect]);

  const sendMessage = useCallback((data: unknown) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    } else {
      console.warn('WebSocket is not connected. Message not sent.');
    }
  }, []);

  useEffect(() => {
    intentionalCloseRef.current = false;
    connect();
    return () => {
      intentionalCloseRef.current = true;
      clearTimeout(retryTimeoutRef.current);
      wsRef.current?.close(1000, 'Component unmounted');
    };
  }, [connect]);

  return { sendMessage, status, lastMessage, disconnect, reconnect };
}
```

### Usage in a React Component

```tsx
// src/components/ChatRoom.tsx

import React, { useState } from 'react';
import { useWebSocket } from '../hooks/useWebSocket';

interface ChatMessage {
  id: string;
  userId: string;
  text: string;
  timestamp: number;
}

const ChatRoom: React.FC<{ roomId: string }> = ({ roomId }) => {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState('');

  const { sendMessage, status } = useWebSocket({
    url: `wss://api.example.com/chat?room=${roomId}`,
    onMessage: (data) => {
      const message = data as ChatMessage;
      setMessages((prev) => [...prev, message]);
    },
  });

  const handleSend = () => {
    if (!input.trim()) return;
    sendMessage({ type: 'chat:message', text: input, room: roomId });
    setInput('');
  };

  return (
    <div>
      <div className="status-bar">
        Status: <strong>{status}</strong>
        {status === 'reconnecting' && <span> (attempting to reconnect...)</span>}
      </div>
      <div className="messages">
        {messages.map((msg) => (
          <div key={msg.id}>
            <strong>{msg.userId}</strong>: {msg.text}
          </div>
        ))}
      </div>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSend()}
        disabled={status !== 'connected'}
        placeholder={status === 'connected' ? 'Type a message...' : 'Reconnecting...'}
      />
      <button onClick={handleSend} disabled={status !== 'connected'}>
        Send
      </button>
    </div>
  );
};

export default ChatRoom;
```

### Message Queue / Buffering During Reconnection

When a WebSocket disconnects, messages sent by the client would be lost. A message queue buffers outgoing messages and flushes them once the connection is restored.

```typescript
// src/services/buffered-websocket.ts

class BufferedWebSocket {
  private ws: WebSocket | null = null;
  private messageQueue: string[] = [];
  private isConnected = false;

  connect(url: string): void {
    this.ws = new WebSocket(url);

    this.ws.onopen = () => {
      this.isConnected = true;
      this.flushQueue();
    };

    this.ws.onclose = () => {
      this.isConnected = false;
    };
  }

  send(data: unknown): void {
    const message = JSON.stringify(data);

    if (this.isConnected && this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(message);
    } else {
      // Buffer messages while disconnected
      this.messageQueue.push(message);
      console.log(`Message queued. Queue size: ${this.messageQueue.length}`);
    }
  }

  private flushQueue(): void {
    console.log(`Flushing ${this.messageQueue.length} queued messages`);
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;
      this.ws?.send(message);
    }
  }
}

// ❌ Bad: Silently dropping messages when disconnected
const sendUnsafe = (ws: WebSocket, data: unknown) => {
  ws.send(JSON.stringify(data)); // Throws if ws is not open
};

// ✅ Good: Buffering messages and flushing on reconnect
const bufferedWs = new BufferedWebSocket();
bufferedWs.send({ type: 'cursor:move', x: 100, y: 200 }); // Queued if disconnected
```

### Heartbeat / Ping-Pong Implementation

```typescript
// src/services/heartbeat.ts

class HeartbeatManager {
  private pingInterval: ReturnType<typeof setInterval> | null = null;
  private pongTimeout: ReturnType<typeof setTimeout> | null = null;
  private readonly PING_INTERVAL = 30000; // 30 seconds
  private readonly PONG_TIMEOUT = 10000;  // 10 seconds to respond

  start(ws: WebSocket, onDead: () => void): void {
    this.stop();

    this.pingInterval = setInterval(() => {
      if (ws.readyState !== WebSocket.OPEN) return;

      ws.send(JSON.stringify({ type: 'ping', timestamp: Date.now() }));

      this.pongTimeout = setTimeout(() => {
        console.warn('Pong not received. Connection presumed dead.');
        onDead();
      }, this.PONG_TIMEOUT);
    }, this.PING_INTERVAL);
  }

  handlePong(): void {
    if (this.pongTimeout) {
      clearTimeout(this.pongTimeout);
      this.pongTimeout = null;
    }
  }

  stop(): void {
    if (this.pingInterval) clearInterval(this.pingInterval);
    if (this.pongTimeout) clearTimeout(this.pongTimeout);
    this.pingInterval = null;
    this.pongTimeout = null;
  }
}

// Usage with a WebSocket connection
const heartbeat = new HeartbeatManager();
const ws = new WebSocket('wss://api.example.com/ws');

ws.onopen = () => {
  heartbeat.start(ws, () => {
    ws.close();
    // Trigger reconnection logic
  });
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'pong') {
    heartbeat.handlePong();
  }
};

ws.onclose = () => {
  heartbeat.stop();
};
```

### Handling Online/Offline Events

The browser provides `online` and `offline` events that can be used to proactively manage WebSocket connections rather than waiting for a timeout.

```typescript
// src/services/network-aware-ws.ts

class NetworkAwareWebSocket {
  private ws: WebSocket | null = null;
  private url: string;

  constructor(url: string) {
    this.url = url;

    window.addEventListener('online', () => {
      console.log('Network restored. Reconnecting WebSocket...');
      this.connect();
    });

    window.addEventListener('offline', () => {
      console.log('Network lost. WebSocket will reconnect when online.');
      this.ws?.close(1000, 'Network offline');
    });

    // Also handle page visibility to conserve resources
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'visible' && !this.isConnected()) {
        this.connect();
      }
    });
  }

  connect(): void {
    if (this.isConnected()) return;
    this.ws = new WebSocket(this.url);
    // ... attach event handlers
  }

  private isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }
}
```

## Benefits

* **Low Latency:** Once the connection is established, messages travel with minimal overhead -- no HTTP headers, no handshake per message. Round-trip times are typically under 10ms on a good network.
* **Bidirectional Communication:** Both client and server can initiate messages at any time, enabling true interactive experiences that HTTP alone cannot deliver.
* **Reduced Server Load:** A single persistent connection replaces potentially hundreds of polling requests per minute per client, reducing CPU and bandwidth consumption on the server.
* **Efficient Binary Support:** WebSockets natively support binary frames, making them suitable for streaming audio, video chunks, or serialized binary protocols like Protocol Buffers.
* **Real-Time User Experience:** Users see updates instantly -- new messages appear, cursors move, scores update -- without needing to refresh or wait for the next poll cycle.
* **Event-Driven Architecture Fit:** WebSockets align naturally with event-driven frontend architectures (Redux, state machines, reactive streams), where incoming server events trigger state transitions.

## Drawbacks and Challenges

* **Connection State Complexity:** Managing open, closing, closed, and reconnecting states adds significant complexity compared to stateless HTTP requests. Every component consuming the connection must handle all states.
* **Scaling Difficulty:** Each WebSocket connection consumes a persistent socket on the server. Scaling to millions of connections requires sticky sessions, connection-aware load balancers, and often a pub/sub layer (Redis, NATS) behind the WebSocket servers.
* **No Built-In Reconnection:** The browser `WebSocket` API does not auto-reconnect. Developers must implement reconnection logic, exponential backoff, message queuing, and state reconciliation manually (or use a library).
* **Proxy and Firewall Issues:** Some corporate proxies, firewalls, and older network infrastructure do not support the WebSocket protocol or may terminate long-lived connections. Fallback mechanisms (like Socket.IO's HTTP long-polling fallback) are sometimes necessary.
* **Debugging Complexity:** WebSocket traffic is harder to inspect than HTTP in browser dev tools. Messages are opaque binary or JSON streams without the structured request/response pattern of REST.
* **Memory Leaks:** Forgetting to remove event listeners, clear intervals, or close connections on component unmount is a common source of memory leaks in single-page applications.
* **Stale State After Reconnection:** When a connection drops and reconnects, the client may have missed messages. Designing a strategy to reconcile state (e.g., requesting a snapshot or replaying missed events) adds complexity.

## Use Cases

* **Chat Applications:** Real-time messaging between users. WebSockets deliver messages instantly and support typing indicators, read receipts, and presence updates.
* **Live Dashboards:** Monitoring dashboards for DevOps, analytics, or business metrics that update in real time as new data arrives from backend systems.
* **Collaborative Editing:** Applications like Google Docs or Figma where multiple users edit the same document simultaneously. WebSockets carry operational transforms or CRDTs between clients.
* **Multiplayer Gaming:** Browser-based games where player positions, actions, and game state must be synchronized across all participants with minimal delay.
* **Notifications:** Push notifications for new content, mentions, alerts, or status changes without requiring the user to refresh the page.
* **Financial Tickers:** Stock prices, cryptocurrency rates, and trading data that update multiple times per second require the low-latency delivery that only WebSockets (or SSE) can provide.
* **IoT Device Communication:** Receiving sensor readings, device status updates, or sending commands to connected devices through a WebSocket bridge.

## Socket.IO Integration

Socket.IO is the most popular WebSocket library for JavaScript. It provides automatic reconnection, room support, fallback to HTTP long polling, and a higher-level event API. While not a pure WebSocket implementation (it uses its own protocol on top), it solves many of the challenges described above out of the box.

```typescript
// src/services/socket-io-client.ts

import { io, Socket } from 'socket.io-client';

interface ServerToClientEvents {
  'chat:message': (message: { userId: string; text: string; timestamp: number }) => void;
  'user:joined': (data: { userId: string; username: string }) => void;
  'user:left': (data: { userId: string }) => void;
}

interface ClientToServerEvents {
  'chat:send': (data: { text: string; roomId: string }) => void;
  'room:join': (roomId: string) => void;
  'room:leave': (roomId: string) => void;
}

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io(
  'wss://api.example.com',
  {
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 30000,
    transports: ['websocket', 'polling'], // Try WebSocket first, fall back to polling
    auth: {
      token: localStorage.getItem('authToken'),
    },
  }
);

// Typed event handlers
socket.on('chat:message', (message) => {
  console.log(`${message.userId}: ${message.text}`);
});

socket.on('connect', () => {
  console.log('Connected with ID:', socket.id);
  socket.emit('room:join', 'general');
});

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
  if (reason === 'io server disconnect') {
    // Server intentionally disconnected, reconnect manually
    socket.connect();
  }
  // Otherwise, Socket.IO will auto-reconnect
});

// Sending a message
socket.emit('chat:send', { text: 'Hello everyone!', roomId: 'general' });
```

### React Hook for Socket.IO

```tsx
// src/hooks/useSocketIO.ts

import { useEffect, useRef, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export function useSocketIO(url: string, options?: Parameters<typeof io>[1]) {
  const socketRef = useRef<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const socket = io(url, options);
    socketRef.current = socket;

    socket.on('connect', () => setIsConnected(true));
    socket.on('disconnect', () => setIsConnected(false));

    return () => {
      socket.disconnect();
      socketRef.current = null;
    };
  }, [url]);

  return { socket: socketRef.current, isConnected };
}
```

## Best Practices

1. **Always implement reconnection with exponential backoff.** The native `WebSocket` API does not reconnect automatically. Use exponential backoff with jitter (random delay) to prevent thundering herd problems when many clients reconnect simultaneously after a server restart.

2. **Use a heartbeat mechanism.** Send periodic ping messages and expect pong responses. This detects silently dead connections (e.g., mobile networks, NAT timeouts) far faster than waiting for a TCP timeout, which can take minutes.

3. **Buffer messages during disconnection.** Queue outgoing messages while the connection is down and flush them upon reconnection. This prevents data loss during brief network interruptions.

4. **Clean up on unmount.** In React (or any SPA framework), close WebSocket connections and remove event listeners when components unmount. Failing to do so causes memory leaks and phantom event handlers.

    ```typescript
    // ✅ Good: Cleanup in useEffect return
    useEffect(() => {
      const ws = new WebSocket(url);
      ws.onmessage = handleMessage;
      return () => {
        ws.close(1000, 'Component unmounted');
      };
    }, [url]);

    // ❌ Bad: No cleanup — connection leaks on every re-render
    useEffect(() => {
      const ws = new WebSocket(url);
      ws.onmessage = handleMessage;
      // Missing return cleanup!
    }, [url]);
    ```

5. **Use typed message contracts.** Define TypeScript interfaces for all message types exchanged over the connection. This catches serialization bugs at compile time and makes the protocol self-documenting.

6. **Implement state reconciliation after reconnection.** When a client reconnects, it may have missed messages. Request a state snapshot or use a `lastEventId` to replay missed events from the server.

7. **Handle tab visibility.** Use the `visibilitychange` event to pause heartbeats and non-critical updates when the tab is in the background. This conserves battery on mobile devices and reduces unnecessary server load.

8. **Use `wss://` (TLS) in production.** Always encrypt WebSocket connections in production. Unencrypted `ws://` connections are vulnerable to man-in-the-middle attacks and may be blocked by corporate proxies.

9. **Set message size limits.** Both client and server should enforce maximum message sizes to prevent abuse. On the client, validate payload size before sending.

10. **Consider using a library for production apps.** Libraries like Socket.IO, `reconnecting-websocket`, or `Sockette` handle reconnection, heartbeats, and fallbacks so you do not have to reimplement these patterns for every project.

## Common Beginner Doubts or Questions

### When should I use WebSockets vs SSE vs polling?

Use **WebSockets** when you need bidirectional communication -- both the client and server send messages to each other (chat, collaborative editing, gaming). Use **Server-Sent Events (SSE)** when only the server pushes data to the client (live feeds, notifications, stock prices). SSE is simpler to set up, auto-reconnects, and works over standard HTTP. Use **polling** (short or long) as a last resort when WebSocket or SSE support is unavailable, or when updates are very infrequent and the simplicity of polling outweighs its inefficiency.

A practical heuristic: if the client only *reads* real-time data, SSE is likely sufficient. If the client also *writes* frequently (beyond simple REST calls), WebSockets are the better fit. Many applications combine WebSockets for real-time features with REST for standard CRUD operations.

### How do I handle reconnection gracefully?

Implement exponential backoff with jitter: after the first disconnect, wait 1 second before retrying, then 2 seconds, 4 seconds, 8 seconds, and so on, up to a maximum cap (e.g., 30 seconds). Add a small random delay (jitter) to each retry to prevent all clients from reconnecting at the same instant after a server restart.

During the disconnection period, buffer any outgoing messages in a queue and flush them once the connection is re-established. On the UI side, show a clear connection status indicator so users know their messages are not being delivered. When the connection is restored, request a state snapshot or provide a `lastEventId` so the server can replay any missed events.

### Are WebSockets secure?

WebSockets can be made secure by using `wss://` (WebSocket Secure), which runs over TLS -- the same encryption layer that `https://` uses. In production, you should always use `wss://`. For authentication, pass tokens during the initial handshake via query parameters or the first message after connection. You can also validate the `Origin` header on the server to prevent cross-origin WebSocket hijacking.

However, WebSockets are not protected by the same-origin policy in the same way as HTTP requests, and they are not subject to CORS. This means the server must explicitly validate that incoming connections are from trusted origins. Additionally, since WebSocket connections are long-lived, token expiration handling requires extra care -- you may need to periodically re-authenticate or refresh tokens over the open connection.

### Do WebSockets work alongside REST APIs?

Absolutely. In fact, most production applications use WebSockets alongside REST APIs rather than replacing REST entirely. REST handles standard request-response operations like authentication, CRUD operations, file uploads, and data queries. WebSockets handle the real-time layer -- pushing live updates, synchronizing state, and delivering notifications.

A common pattern is for the REST API to handle mutations (creating a message, updating a document) and for the WebSocket to broadcast the resulting event to all connected clients. This separates concerns cleanly: REST for commands, WebSockets for events.

### How do I scale WebSocket connections?

Scaling WebSockets is harder than scaling stateless HTTP APIs because each connection is persistent and stateful. Here are the key strategies:

**Horizontal scaling** requires a *pub/sub backend* (Redis Pub/Sub, NATS, Kafka) so that a message published to one WebSocket server is forwarded to clients connected to other servers. Without this, clients on different servers would not receive each other's messages.

**Sticky sessions** ensure that a client always reconnects to the same server (via cookie or IP hashing), which simplifies state management but limits flexibility. Alternatively, externalize all state so any server can handle any client.

**Connection limits** vary by server and OS. A single Node.js process can typically handle 10,000-100,000 concurrent WebSocket connections depending on message volume and processing complexity. Beyond that, you need multiple processes or servers behind a WebSocket-aware load balancer (NGINX, HAProxy, or a cloud service like AWS ALB with WebSocket support).

### What happens if the client sends a message while disconnected?

With the native `WebSocket` API, calling `send()` on a closed connection throws an error. This is why message buffering is critical. A well-designed WebSocket wrapper queues messages in memory (or `localStorage` for persistence across page reloads) and flushes them when the connection is restored. If messages are time-sensitive (e.g., cursor position updates), you may choose to discard stale queued messages instead of sending them all. The buffering strategy depends on your application's semantics -- chat messages should be queued, but real-time cursor positions should be dropped.

### How do I debug WebSocket connections?

Browser DevTools provide a WebSocket inspector. In Chrome, open the Network tab, filter by "WS", and click on a WebSocket connection to see individual frames (messages) in both directions with timestamps. Firefox offers similar tooling. For more advanced debugging, tools like `wscat` (a command-line WebSocket client) or Postman's WebSocket support let you connect to a WebSocket server and send/receive messages manually. On the code side, wrap your WebSocket with logging middleware that records all sent and received messages with timestamps for post-mortem analysis.
