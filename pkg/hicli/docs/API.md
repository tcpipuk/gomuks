# hicli API Documentation

This document explains the core architecture, data flow, and component responsibilities in hicli.
Rather than exhaustive code examples, this focuses on understanding how the Matrix client library
works internally and how to integrate with it effectively.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [HiClient Structure](#hiclient-structure)
3. [Command System](#command-system)
4. [Event Processing Pipeline](#event-processing-pipeline)
5. [Database Architecture](#database-architecture)
6. [Encryption System](#encryption-system)
7. [Matrix Protocol Integration](#matrix-protocol-integration)
8. [Integration Patterns](#integration-patterns)

## Architecture Overview

hicli is designed as a **Matrix abstraction layer** that handles all the complexity of the Matrix
protocol, leaving you free to focus on building user interfaces and application logic.

### Core Design Principles

**Command-Response Pattern**: Your application sends typed commands to HiClient, which processes
them asynchronously and sends back typed event responses.

**Event-Driven Updates**: All Matrix state changes are delivered as events to your application,
making it easy to keep your UI in sync.

**Offline-First**: All data is persisted locally first, then synchronized, ensuring your application
works reliably even with poor connectivity.

**Encryption Transparency**: End-to-end encryption is handled automatically - your application code
doesn't need to know whether a room is encrypted.

### System Components

```plain
┌─────────────────┐    Commands     ┌─────────────────┐
│ Your Application│ ──────────────► │    HiClient     │
│                 │                 │                 │
│                 │ ◄────────────── │                 │
└─────────────────┘     Events      └─────────────────┘
                                            │
                        ┌───────────────────┼───────────────────┐
                        ▼                   ▼                   ▼
                ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
                │   Database    │   │ Crypto Engine │   │ Matrix Client │
                │   (SQLite)    │   │ (Olm/Megolm)  │   │  (mautrix)    │
                └───────────────┘   └───────────────┘   └───────────────┘
```

## HiClient Structure

The `HiClient` struct is the central coordinator that manages all Matrix operations. Understanding
its major components helps you understand how the system works.

### Core Components

**Database (`database.Database`)**: The main SQLite database that stores all Matrix state - rooms,
events, timeline, current state, media cache, etc. This is your application's persistent memory.

**CryptoDB (`dbutil.Database`)**: A separate SQLite database dedicated to encryption keys and crypto
state. Separated for security and performance reasons.

**Account (`database.Account`)**: Current user session information - user ID, access token, device ID,
sync state, push rules, etc.

**Client (`mautrix.Client`)**: Low-level HTTP client that handles Matrix protocol operations -
sending requests, receiving responses, managing the connection.

**Crypto (`crypto.OlmMachine`)**: End-to-end encryption engine that handles Olm/Megolm protocols,
device verification, key management, and cross-signing.

### Key Responsibilities

**State Coordination**: HiClient ensures all components stay in sync - when a new message arrives,
it's stored in the database, potentially decrypted by the crypto engine, and then delivered to your application.

**Background Processing**: Many operations happen asynchronously - encryption/decryption, database
writes, network requests - while keeping your application responsive.

**Error Handling**: Network failures, encryption errors, and protocol issues are handled gracefully
with automatic retry logic.

**Resource Management**: Database connections are pooled, old events are cleaned up periodically,
and encryption sessions have lifecycle management.

## Command System

All Matrix operations in hicli use a uniform command pattern. This makes the API predictable and
makes it easy to add new features.

### Command Structure

Every command follows the same pattern:

1. **Request struct**: Contains all parameters for the operation
2. **Context**: For cancellation and timeout handling
3. **Response struct**: Contains results and any returned data
4. **Error handling**: Typed errors for different failure modes

### Command Categories

#### Authentication Commands

**Purpose**: Manage user sessions and authentication state

**Key Commands**: `Login`, `Logout`

**What happens**: Establishes or tears down Matrix session, manages access tokens, initializes/cleans
up local state

#### Messaging Commands

**Purpose**: Send and manage messages and events

**Key Commands**: `SendMessage`, `SendEvent`, `EditMessage`

**What happens**: Composes Matrix events, handles encryption if needed, sends to server, tracks
delivery status

#### Room Commands

**Purpose**: Manage room membership and settings

**Key Commands**: `JoinRoom`, `LeaveRoom`, `CreateRoom`, `InviteUser`

**What happens**: Sends appropriate Matrix events, updates local room state, handles permission checks

#### State Commands

**Purpose**: Read and modify room state

**Key Commands**: `SetState`, `GetRoomState`, `GetMembers`

**What happens**: Manages room configuration, power levels, member lists, room settings

#### Sync Commands

**Purpose**: Manage timeline and read state

**Key Commands**: `MarkRead`, `SetTyping`, `Paginate`

**What happens**: Updates read receipts, sends typing indicators, loads historical messages

### Command Processing Flow

1. **Validation**: Request parameters are validated for correctness
2. **State Check**: Current Matrix state is checked (e.g., are we in this room?)
3. **Database Transaction**: Changes are prepared in a database transaction
4. **Network Operation**: Matrix HTTP request is sent (if needed)
5. **State Update**: Local state is updated with results
6. **Event Dispatch**: Relevant events are sent to your application

## Event Processing Pipeline

Understanding how Matrix events flow through hicli helps you understand when and why you receive
certain callbacks.

### Matrix Sync Pipeline

1. **Sync Request**: HiClient requests latest changes from Matrix server
2. **Response Processing**: Server response contains rooms, events, account data, device lists
3. **Database Storage**: All events are stored to SQLite in a transaction
4. **Decryption Queue**: Encrypted events are queued for background processing
5. **Event Assembly**: Processed events are assembled into meaningful updates
6. **Callback Dispatch**: Your application receives organized event updates

### Event Types You Receive

#### SyncComplete

**When**: After each successful Matrix sync
**Contains**: Updated room list with computed metadata (unread counts, last messages, etc.)
**Purpose**: Update your room list UI and overall application state

#### EventsDecrypted

**When**: After encrypted events are successfully decrypted
**Contains**: Batch of decrypted events, organized by room
**Purpose**: Display new messages and update timelines

#### SyncStatus

**When**: Connection state changes or sync encounters errors
**Contains**: Current sync state, connection status, error information
**Purpose**: Update connection indicators and handle offline scenarios

#### SendComplete

**When**: A message you sent is confirmed by the server
**Contains**: Correlation between your local message and the server's event ID
**Purpose**: Update UI to show message was delivered

#### Typing

**When**: Other users start/stop typing in rooms you're in
**Contains**: Room ID and list of currently typing users
**Purpose**: Show typing indicators in your UI

#### ClientState

**When**: Authentication or verification status changes
**Contains**: Login state, user ID, device verification status
**Purpose**: Update UI for login flows and security status

### Event Processing Best Practices

**Non-Blocking Handlers**: Event callbacks should return quickly - do UI updates and defer heavy processing

**Batch Updates**: Multiple events often arrive together - batch UI updates to avoid flickering

**Error Handling**: Event processing can fail - handle errors gracefully and don't crash on malformed
events

**State Consistency**: Events represent state changes - make sure your UI stays consistent with
delivered events

## Database Architecture

hicli uses SQLite with a schema designed around Matrix's event-sourced architecture. Understanding
this helps you understand performance characteristics and data flow.

### Schema Design Principles

**Event Storage**: All Matrix events are stored exactly as received, creating an immutable event log

**Computed Views**: Derived data (room lists, unread counts) is computed from events and cached in
separate tables

**Timeline Ordering**: Events within rooms are ordered in a timeline table for efficient pagination

**State Resolution**: Current room state is computed from state events and cached for fast queries

### Key Tables

#### Events Table

**Purpose**: Stores all Matrix events exactly as received
**Key insight**: This is the source of truth - everything else is derived from here
**Performance**: Indexed by room ID and timestamp for timeline queries

#### Timeline Table

**Purpose**: Maintains the ordering of events within each room
**Key insight**: Enables efficient pagination and "jump to message" functionality
**Performance**: Ordered by timeline row ID for fast chronological access

#### Current State Table

**Purpose**: Caches the latest state events for each (room, event_type, state_key) combination
**Key insight**: Avoids needing to scan all events to find current room name, membership, etc.
**Performance**: Indexed for instant lookups of current room state

#### Rooms Table

**Purpose**: Stores computed room metadata and UI state
**Key insight**: Unread counts, last message info, etc. are computed and cached here
**Performance**: Optimized for room list queries

### Database Operations

**Sync Processing**: Each Matrix sync is processed in a single SQLite transaction for consistency

**Event Insertion**: New events are inserted and trigger updates to computed tables

**State Resolution**: State events update the current_state table when they arrive

**Timeline Updates**: New timeline events update room metadata and unread counts

**Cleanup Operations**: Old events and media are periodically cleaned up to manage storage

## Encryption System

hicli's encryption system handles all Olm/Megolm complexity automatically, but understanding how it
works helps you integrate securely.

### Encryption Architecture

**Olm Sessions**: End-to-end encrypted channels between specific devices for sending room keys and
to-device messages

**Megolm Sessions**: Encrypted group chat sessions where all participants share the same room key
for a period of time

**Device Verification**: Cryptographic verification that other devices belong to who they claim to be

**Cross-Signing**: A web of trust where users sign their own devices and can sign other users' devices

### Encryption Flow

#### Sending Encrypted Messages

1. **Room Check**: Is this room encrypted? If not, send plaintext
2. **Session Check**: Do we have a current Megolm session for this room?
3. **Key Sharing**: If needed, create new session and share keys with room members
4. **Encryption**: Encrypt message content with current room key
5. **Sending**: Send encrypted event to Matrix server

#### Receiving Encrypted Messages

1. **Event Storage**: Encrypted events are stored as-is in the database
2. **Decryption Queue**: Events are queued for background decryption
3. **Session Lookup**: Find the appropriate Megolm session for decryption
4. **Decryption**: Decrypt event content using room key
5. **Event Dispatch**: Decrypted event is sent to your application

### Key Management

**Automatic Key Rotation**: Room keys are rotated when members join/leave encrypted rooms

**Key Backup**: Important keys can be backed up to the server for account recovery

**Device Verification**: Interactive verification ensures you're talking to the right devices

**Cross-Signing Bootstrap**: Sets up the web of trust for simplified device verification

### Security Considerations

**Pickle Key**: Encrypt local crypto database with a strong key derived from user password

**Key Storage**: Crypto database should have restricted file permissions

**Device Verification**: Always verify new devices to prevent man-in-the-middle attacks

**Recovery Setup**: Set up key backup for account recovery scenarios

## Matrix Protocol Integration

hicli abstracts Matrix protocol complexity, but understanding the underlying protocol helps you
design better applications.

### Matrix Fundamentals

**Rooms**: All Matrix communication happens in rooms - they're like chat channels but can contain
any type of event

**Events**: Everything in Matrix is an event - messages, state changes, receipts, typing
notifications, etc.

**Event Types**: Different event types serve different purposes (m.room.message, m.room.member,
m.room.encrypted, etc.)

**State vs Timeline**: State events represent room configuration; timeline events represent things
that happened

### Sync Model

**Long Polling**: Matrix uses long-polling sync requests to deliver events efficiently

**Incremental Updates**: Each sync only contains changes since the last sync token

**Event Ordering**: Events are delivered in causal order, respecting Matrix's event graph

**Pagination**: Historical events can be loaded by paginating backwards through the timeline

### Room State Management

**State Resolution**: Matrix has complex rules for resolving conflicting state events

**Power Levels**: Hierarchical permissions system controls who can perform different actions

**Membership States**: Users can be in various states relative to a room (joined, invited, left, banned)

**Canonical Aliases**: Rooms can have human-readable aliases like #general:matrix.org

### Federation Considerations

**Homeserver Differences**: Different Matrix servers may have different capabilities and performance
characteristics

**Network Partitions**: Federation can split, causing temporary inconsistencies

**Event Signing**: All events are cryptographically signed to prevent tampering

**Server-to-Server Protocol**: Events are replicated between homeservers using the federation protocol

## Integration Patterns

Understanding common integration patterns helps you build robust applications on top of hicli.

### UI Framework Integration

#### Event-Driven Updates

**Pattern**: React to hicli events by updating your UI state/components
**Best Practice**: Batch multiple related events into single UI updates
**Example**: SyncComplete event triggers room list refresh

#### Command-Based Actions

**Pattern**: User interactions trigger hicli commands
**Best Practice**: Provide immediate UI feedback, then handle success/failure in events
**Example**: Send button click → SendMessage command → SendComplete event

#### Thread Safety

**Pattern**: hicli is thread-safe, but UI frameworks often aren't
**Best Practice**: Use your UI framework's thread-safe update mechanisms
**Example**: Post UI updates to main thread/event loop

### State Management

#### Single Source of Truth

**Pattern**: hicli database is the authoritative source for Matrix state
**Best Practice**: Don't duplicate Matrix state in your application - query hicli when needed
**Example**: Room names come from hicli.GetRoom(), not cached in UI

#### Computed State

**Pattern**: Derive UI state from Matrix state rather than tracking it separately
**Best Practice**: Use hicli events to recompute UI state when underlying data changes
**Example**: "Unread" indicator computed from room.UnreadCount in SyncComplete events

#### Optimistic Updates

**Pattern**: Update UI immediately for user actions, then reconcile with server responses
**Best Practice**: Use temporary IDs for tracking optimistic updates
**Example**: Show message immediately, update with real event ID in SendComplete

### Error Handling

#### Network Resilience

**Pattern**: Handle network failures gracefully with automatic retry
**Implementation**: hicli handles most network errors automatically
**UI Pattern**: Show connection status and allow manual retry

#### Encryption Errors

**Pattern**: Handle cases where messages can't be decrypted
**Implementation**: Display "Unable to decrypt" placeholders
**Recovery**: Prompt for device verification or key backup restore

#### Protocol Errors

**Pattern**: Handle Matrix protocol errors (rate limiting, permissions, etc.)
**Implementation**: Display meaningful error messages to users
**Best Practice**: Don't expose raw Matrix error codes to end users

### Performance Optimization

#### Lazy Loading

**Pattern**: Load data only when needed to reduce memory usage and startup time
**Implementation**: Use hicli's pagination commands to load historical data on demand
**UI Pattern**: Virtual scrolling for large message histories

#### Background Processing

**Pattern**: Handle expensive operations without blocking the UI
**Implementation**: hicli handles encryption/decryption in background threads
**UI Pattern**: Show loading indicators for slow operations

#### Resource Management

**Pattern**: Clean up resources and limit memory usage
**Implementation**: hicli automatically cleans up old events and sessions
**UI Pattern**: Unload off-screen components and images

## Complete API Reference

### HiClient Methods

#### Authentication

**Login(ctx, req *LoginRequest) (*LoginResponse, error)**

- Authenticates user with homeserver
- Initializes crypto if password provided
- Starts background sync
- Returns user ID and device ID

**Logout(ctx) error**

- Invalidates access token
- Stops sync
- Clears local session data
- Does not clear database

#### Room Operations

**JoinRoom(ctx, req *JoinRoomRequest) (*JoinRoomResponse, error)**

- Joins room by ID or alias
- Downloads room state and recent history
- Sets up encryption if room is encrypted
- Returns canonical room ID

**LeaveRoom(ctx, req *LeaveRoomRequest) error**

- Leaves specified room
- Stops receiving events for room
- Clears typing indicators
- Room data remains in local database

**CreateRoom(ctx, req *CreateRoomRequest) (*CreateRoomResponse, error)**

- Creates new room with specified settings
- Automatically joins creator
- Sends invites if specified
- Returns room ID and alias (if set)

**InviteUser(ctx, req *InviteUserRequest) error**

- Invites user to room
- Requires invite permission in room
- User receives invitation event
- Does not guarantee acceptance

#### Messaging

**SendMessage(ctx, req *SendMessageRequest) (*SendMessageResponse, error)**

- Sends text message to room
- Handles markdown processing if specified
- Encrypts automatically for encrypted rooms
- Returns event ID and transaction ID

**SendEvent(ctx, req *SendEventRequest) (*SendEventResponse, error)**

- Sends arbitrary Matrix event
- Used for custom event types
- Handles state events and timeline events
- Returns event ID

**EditMessage(ctx, req *EditMessageRequest) error**

- Edits previously sent message
- Only works for your own messages
- Sends m.replace event
- History is preserved

#### State Management

**SetState(ctx, req *SetStateRequest) error**

- Sets room state event
- Requires appropriate power level
- Examples: room name, topic, power levels
- Changes are immediately visible

**GetRoomState(ctx, roomID) (map[string]map[string]*Event, error)**

- Returns current room state
- Organized by event type and state key
- Cached locally for performance
- Updates automatically via sync

**GetMembers(ctx, roomID) ([]*Member, error)**

- Returns room member list
- Includes display names and avatars
- Filtered by membership state
- Supports lazy loading

#### Timeline Operations

**MarkRead(ctx, req *MarkReadRequest) error**

- Marks messages as read up to event ID
- Updates local unread count
- Sends read receipt to server
- Affects room list ordering

**SetTyping(ctx, req *SetTypingRequest) error**

- Sends typing indicator
- Auto-expires after timeout
- Only visible to other room members
- Rate limited by server

**Paginate(ctx, req *PaginateRequest) (*PaginateResponse, error)**

- Loads historical messages
- Uses timeline row ID for position
- Returns requested number of events
- Handles end-of-history gracefully

#### Media Handling

**UploadMedia(ctx, data []byte, contentType, filename string) (*UploadResponse, error)**

- Uploads file to Matrix media server
- Returns MXC URI for referencing
- Supports any file type
- Has size limits (check server config)

**DownloadMedia(ctx, mxcURI string) ([]byte, string, error)**

- Downloads media by MXC URI
- Returns data and content type
- Caches locally automatically
- Handles thumbnails separately

#### Encryption Operations

**VerifyDevice(ctx, req *VerifyDeviceRequest) error**

- Marks device as verified
- Required for secure messaging
- Uses SAS or QR code verification
- Affects message trust indicators

**CrossSign(ctx, req *CrossSignRequest) error**

- Signs another user's master key
- Establishes web of trust
- Requires user confirmation
- Simplifies future device verification

#### Space Management

**GetSpaceHierarchy(ctx, spaceID) (*SpaceHierarchy, error)**

- Returns space structure
- Includes child rooms and subspaces
- Respects permissions
- Cached for performance

**AddRoomToSpace(ctx, spaceID, roomID, order string, suggested bool) error**

- Adds room to space
- Sets ordering hint
- Marks as suggested or not
- Requires space admin permission

### Data Structures

#### Room

```go
type Room struct {
    ID              id.RoomID
    Name            string
    Topic           string
    Avatar          string
    Canonical       string        // Canonical alias
    UnreadCount     int
    HighlightCount  int          // Mentions and keywords
    LastMessage     *Event
    LastMessageID   string
    Encrypted       bool
    MemberCount     int
    JoinRule        string       // public, invite, knock, restricted
    GuestAccess     string       // can_join, forbidden
    HistoryVisibility string     // invited, joined, shared, world_readable
    Tags            map[string]interface{}
    IsDirect        bool
    CreationTime    time.Time
    LastActivity    time.Time
}
```

#### Event

```go
type Event struct {
    ID              string
    Type            event.Type
    RoomID          id.RoomID
    Sender          id.UserID
    Timestamp       time.Time
    Content         json.RawMessage
    PrevContent     json.RawMessage  // For state events
    StateKey        *string          // nil for timeline events
    Unsigned        json.RawMessage
    Redacts         string           // Event ID being redacted
    TimelineRowID   int64           // For pagination
    DecryptedType   event.Type      // Original type if decrypted
    DecryptionError error           // If decryption failed
}
```

#### Member

```go
type Member struct {
    UserID      id.UserID
    Displayname string
    AvatarURL   string
    Membership  string  // join, leave, invite, ban, knock
    PowerLevel  int
    LastSeen    time.Time
    Presence    string  // online, offline, unavailable
}
```

### Configuration Options

#### Config

```go
type Config struct {
    DataDir         string              // SQLite database location
    CacheDir        string              // Media and file cache
    DeviceName      string              // Device identifier
    PickleKey       []byte              // Encryption key for crypto DB
    EventHandler    func(interface{})   // Event callback
    LogoutFunc      func()              // Called on forced logout

    // Optional settings
    UserAgent       string              // HTTP user agent
    StateStore      StateStore          // Custom state storage
    Logger          zerolog.Logger      // Custom logger
    ProxyURL        string              // HTTP proxy
    Timeout         time.Duration       // Request timeout
    RetryCount      int                 // Failed request retries

    // Sync settings
    SyncTimeout     time.Duration       // Long poll timeout
    FilterID        string              // Sync filter

    // Encryption settings
    EnableE2EE      bool                // Enable encryption
    TrustFirstUse   bool                // Auto-trust first device
    ShareKeysToUnverifiedDevices bool   // Risky but convenient
}
```

### Error Types

#### Common Errors

- `ErrNotLoggedIn`: Operation requires authentication
- `ErrRoomNotFound`: Room doesn't exist or not joined
- `ErrPermissionDenied`: Insufficient power level
- `ErrRateLimited`: Too many requests
- `ErrEncryptionFailed`: Could not encrypt message
- `ErrDecryptionFailed`: Could not decrypt event
- `ErrInvalidContent`: Malformed event content
- `ErrDatabaseError`: SQLite operation failed

### Database Schema

#### Key Tables

- `events`: All Matrix events (immutable log)
- `timeline`: Event ordering within rooms
- `current_state`: Latest state for each (room, type, state_key)
- `rooms`: Computed room metadata
- `members`: Room membership cache
- `media`: File download cache
- `sync_state`: Sync tokens and positions

#### Indexes

- `events(room_id, timestamp)`: Timeline queries
- `events(type, room_id)`: State event lookups
- `timeline(room_id, row_id)`: Pagination
- `current_state(room_id, type, state_key)`: State queries
- `rooms(last_activity)`: Room list sorting

### Performance Characteristics

#### Database Operations

- Room list query: O(1) with proper indexes
- Timeline loading: O(log n) for any position
- State lookups: O(1) hash table access
- Full-text search: O(n) scan (use FTS extension)

#### Memory Usage

- Base overhead: ~10MB
- Per room: ~1KB metadata + events
- Per event: ~500 bytes average
- Crypto sessions: ~1KB per device

#### Network Usage

- Initial sync: Variable (depends on room count)
- Incremental sync: ~1-10KB typically
- Message send: ~500 bytes
- Media upload: File size + ~1KB overhead
