# hicli Documentation

The hicli (High-Level Interface Client Library) package provides a comprehensive framework for
building Matrix clients. It abstracts away the complexity of Matrix protocol implementation,
encryption, and state management, allowing you to focus on building user interfaces and application logic.

## What is hicli?

hicli is the "business logic layer" for Matrix clients. It sits between your UI and the Matrix
protocol, handling:

- **Protocol complexity**: Matrix sync, event processing, state resolution
- **Encryption**: Automatic E2EE with Olm/Megolm, device verification, key management
- **Data persistence**: SQLite-based storage for offline access and state caching
- **Event processing**: Decryption, timeline assembly, push rule evaluation
- **Command interface**: JSON-based API for all Matrix operations

## Documentation Structure

- **[API.md](./API.md)** - Complete API reference documentation
- **[Integration.md](./Integration.md)** - Integration guide for TUI clients
- **[Examples.md](./Examples.md)** - Code examples and usage patterns

## Quick Links

### Getting Started

- [Installation](#installation)
- [Basic Usage](#basic-usage)
- [Architecture Overview](#architecture-overview)

### Core Concepts

- [Command System](./API.md#command-system)
- [Event Handling](./API.md#event-handling)
- [Matrix Integration](./API.md#matrix-integration)
- [Encryption & Verification](./API.md#encryption--verification)

### Advanced Topics

- [Database Layer](./API.md#database-layer)
- [Performance Optimization](./API.md#best-practices)
- [Security Considerations](./API.md#security-considerations)

## Installation

```bash
go get github.com/gomuks/gomuks/pkg/hicli
```

## Basic Usage

```go
package main

import (
    "context"
    "log"

    "github.com/gomuks/gomuks/pkg/hicli"
)

func main() {
    // Create configuration
    cfg := &hicli.Config{
        DataDir:      "./data",
        CacheDir:     "./cache",
        DeviceName:   "My Matrix Client",
        PickleKey:    []byte("secure-pickle-key"),
        EventHandler: handleEvent,
    }

    // Create and start client
    client, err := hicli.NewHiClient(cfg)
    if err != nil {
        log.Fatal(err)
    }

    if err := client.Start(context.Background()); err != nil {
        log.Fatal(err)
    }

    // Login
    _, err = client.Login(context.Background(), &hicli.LoginRequest{
        Homeserver: "https://matrix.org",
        Username:   "alice",
        Password:   "secret",
    })
    if err != nil {
        log.Fatal(err)
    }
}

func handleEvent(evt any) {
    // Process Matrix events
}
```

## Architecture Philosophy

hicli follows a **command-response pattern** where:

- Your application sends **commands** (login, send message, join room) to HiClient
- HiClient processes commands and sends **events** back to your application
- All Matrix complexity is hidden behind a clean, asynchronous interface

### Data Flow

```plain
Your Application
    ↓ Commands (SendMessage, JoinRoom, etc.)
    ↓
HiClient
    ↓ Matrix Protocol Operations
    ↓
Matrix Server
    ↓ Sync responses, events
    ↓
HiClient (processes & stores)
    ↓ Events (SyncComplete, EventsDecrypted, etc.)
    ↓
Your Application (updates UI)
```

### Core Responsibilities

**HiClient**: The central coordinator that manages Matrix protocol, encryption, database, and event processing.

**Database Layer**: SQLite-based persistence for rooms, events, state, media, and encryption keys.

**Crypto Engine**: Handles all E2EE operations - Olm sessions, Megolm rooms, device verification,
key backup.

**Event System**: Processes incoming Matrix events, decrypts them, and assembles them into timelines.

**Command System**: Provides a uniform JSON-based API for all Matrix operations.

### Key Features

**Matrix Protocol Support**

- Full Matrix v1.11 compatibility
- Spaces and hierarchies
- Threading and reactions
- Rich media handling

**End-to-End Encryption**

- Olm/Megolm protocols
- Device verification
- Cross-signing support
- Key backup and recovery

**State Management**

- Comprehensive room state tracking
- Timeline management with pagination
- Unread counting and notifications
- Offline-first design

**Performance & Reliability**

- SQLite-based persistence
- Efficient sync processing
- Automatic retry logic
- Memory-conscious design

## Key Architectural Decisions

**Command-Response Pattern**: All operations are explicit commands with clear responses, making the
API predictable and debuggable.

**Event-Driven Updates**: Real-time updates come through event callbacks, keeping your UI reactive
to Matrix state changes.

**Offline-First Design**: All data is persisted locally first, then synchronized, ensuring your app
works without constant connectivity.

**Encryption Transparency**: E2EE is handled automatically - encrypted rooms just work without
special handling in your application code.

**Thread Safety**: All operations are thread-safe, allowing UI updates from any goroutine.

## Understanding the Component Interaction

### HiClient Structure

The `HiClient` struct contains all the major subsystems:

- **Database**: Main SQLite database for Matrix state
- **CryptoDB**: Separate SQLite database for encryption keys
- **Client**: Low-level Matrix HTTP client (mautrix)
- **Crypto**: Olm/Megolm encryption engine
- **Account**: Current user session and settings

### Event Processing Pipeline

1. **Matrix Sync**: Receives events from Matrix server
2. **Database Storage**: Events stored to SQLite for persistence
3. **Decryption Queue**: Encrypted events queued for background processing
4. **Event Assembly**: Decrypted events assembled into room timelines
5. **Callback Dispatch**: Processed events sent to your application

### Database Schema Design

The database is designed around Matrix's event-sourced architecture:

- **Events table**: All Matrix events (messages, state changes, etc.)
- **Timeline table**: Ordering of events within rooms
- **Current state**: Latest state events for efficient queries
- **Rooms table**: Computed room metadata and unread counts

## When to Use hicli

**Perfect for:**

- Full-featured Matrix clients (chat apps, collaboration tools)
- Applications requiring Matrix integration with complex UI
- Bots and automation tools that need reliable Matrix connectivity
- Bridging applications that translate between Matrix and other protocols

**Consider alternatives for:**

- Simple webhook-style bots (use matrix-bot-sdk instead)
- Read-only Matrix monitoring (use simpler HTTP client)
- Applications that don't need encryption (use basic mautrix client)

## Core Concepts

### Commands vs Events

**Commands** are requests you send to HiClient (SendMessage, JoinRoom, MarkRead)
**Events** are notifications HiClient sends to you (SyncComplete, EventsDecrypted, SendComplete)

This separation makes the API predictable - you know exactly when you're requesting an action vs.
responding to a change.

### State Management

HiClient maintains three types of state:

1. **Matrix State**: Room membership, power levels, room names (from Matrix state events)
2. **Timeline State**: Message history, read receipts, typing indicators (from Matrix timeline events)
3. **Client State**: Unread counts, sync status, local UI state (computed by HiClient)

### Encryption Transparency

Encryption "just works" - your application doesn't need to know whether a room is encrypted:

- Send messages the same way for encrypted and unencrypted rooms
- Receive decrypted events through the same callback system
- Device verification and key management happen in the background

## Understanding Matrix Through hicli

### Rooms and Events

Everything in Matrix is an **event** in a **room**:

- Messages are `m.room.message` events
- Someone joining is a `m.room.member` event
- Room name changes are `m.room.name` events
- Even encryption keys are events (`m.room.encrypted`)

hicli handles all event types and presents them through a unified interface.

### Timeline vs State

Matrix events are either **timeline events** (messages, reactions) or **state events** (room name, membership):

- **Timeline events**: Ordered sequence, can be paginated, represent "things that happened"
- **State events**: Key-value store, represent "current room configuration"

hicli automatically separates these and stores them appropriately.

### Sync and Reactivity

Matrix uses a "sync" model where:

1. Client requests latest changes since last sync
2. Server responds with new events and state changes
3. Client processes events and updates local state
4. Client immediately syncs again for next batch

hicli manages this sync loop and translates sync responses into meaningful application events.

## Integration Philosophy

hicli is designed to integrate with any UI framework through its event-driven architecture:

### With TUI Frameworks (mauview, tview, etc.)

- HiClient events trigger UI updates via `RedrawSoon()`
- UI input creates HiClient commands
- Focus and navigation handled by UI framework
- HiClient manages Matrix state in background

### With Web Frameworks (via WebSocket/HTTP)

- HiClient events serialized and sent to web frontend
- Web frontend sends commands back via HTTP/WebSocket
- HiClient acts as Matrix backend service

### With Native GUI Frameworks

- HiClient events trigger widget updates
- GUI input handlers create HiClient commands
- Native threading models work with HiClient's thread safety

### Integration Best Practices

1. **Separate concerns**: UI handles presentation, HiClient handles Matrix
2. **Event-driven**: React to HiClient events rather than polling state
3. **Command-based**: Use HiClient commands for all Matrix operations
4. **Thread-safe**: HiClient can be called from any goroutine

## Rich Text Processing

Built-in support for rich text formatting:

```go
// Markdown processing
content := "**Bold text** and *italic text*"
htmlContent, plaintext := ProcessMarkdown(content, MarkdownOptions{
    EnableSpoilers: true,
    EnableMath:     true,
    EnableTables:   true,
})

// Send formatted message
req := &hicli.SendMessageRequest{
    RoomID:      roomID,
    Content:     content,
    ContentType: "text/markdown",
}
```

## Media Handling

Comprehensive media support:

```go
// Upload media
uploadResp, err := client.UploadMedia(ctx, imageData, "image/png", "screenshot.png")

// Send media message
mediaReq := &hicli.SendEventRequest{
    RoomID:    roomID,
    EventType: "m.room.message",
    Content: map[string]any{
        "msgtype": "m.image",
        "body":    "screenshot.png",
        "url":     uploadResp.ContentURI,
        "info": map[string]any{
            "mimetype": "image/png",
            "size":     len(imageData),
        },
    },
}
```

## Space Management

Full support for Matrix Spaces:

```go
// Get space hierarchy
hierarchy, err := client.GetSpaceHierarchy(spaceID)

// Add room to space
err = client.AddRoomToSpace(ctx, spaceID, roomID, "order", true)

// Get rooms in space
rooms, err := client.GetRoomsInSpace(spaceID)
```

## Error Handling

Robust error handling with Matrix-specific error types:

```go
if err != nil {
    if httpErr, ok := err.(*mautrix.HTTPError); ok {
        switch httpErr.ResponseCode {
        case 403:
            handlePermissionDenied()
        case 429:
            handleRateLimited()
        case 500:
            handleServerError()
        }
    }
}
```

## Performance and Reliability

### Database Design

hicli uses SQLite with careful schema design:

- **Indexed queries**: Common queries (recent messages, room lists) are highly optimized
- **Transactional sync**: Each Matrix sync is processed in a single transaction
- **Incremental updates**: Only changed data is processed and stored
- **Background cleanup**: Old events and media are cleaned up automatically

### Memory Management

- **Event streaming**: Large sync responses are processed incrementally
- **Lazy loading**: Room members and historical events loaded on demand
- **Cache limits**: In-memory caches have size limits to prevent unbounded growth
- **Garbage collection**: Unused Olm sessions and old events are cleaned up

### Network Resilience

- **Automatic retry**: Failed requests are retried with exponential backoff
- **Offline support**: App continues working with cached data when network is unavailable
- **Incremental sync**: Only downloads new events, not entire room state
- **Connection pooling**: HTTP connections are reused for efficiency

## Security Best Practices

1. **Key Management**: Store pickle keys securely
2. **Device Verification**: Verify all new devices
3. **Data Protection**: Encrypt local databases
4. **Network Security**: Use TLS for all connections
5. **Input Validation**: Sanitize all user inputs

## Common Use Cases

### Chat Client

- Real-time messaging
- Media sharing
- User presence
- Read receipts

### Collaboration Tool

- File sharing
- Thread discussions
- Space organization
- Member management

### Bot Framework

- Automated responses
- Command processing
- Event monitoring
- Integration services

### Bridging

- Protocol translation
- Message forwarding
- User mapping
- State synchronization

## Migration and Compatibility

hicli maintains backward compatibility with existing Matrix clients:

- Standard Matrix protocol compliance
- Compatible with Element and other clients
- Supports legacy room versions
- Automatic database migrations

## Debugging and Monitoring

Built-in debugging features:

```go
// Enable debug logging
client.Log = zerolog.New(os.Stdout).Level(zerolog.DebugLevel)

// Monitor sync status
go func() {
    for {
        status := client.GetSyncStatus()
        log.Printf("Sync state: %s, connected: %t", status.State, status.Connected)
        time.Sleep(30 * time.Second)
    }
}()
```

## Contributing

hicli is part of the gomuks project. See the main gomuks repository for contribution guidelines.

## License

hicli is licensed under the same license as gomuks. See the main repository for details.

For complete API documentation, see [API.md](./API.md).
