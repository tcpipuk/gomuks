# hicli Examples and Patterns

This document provides practical examples of using the hicli package for building Matrix clients,
with focus on common patterns and real-world usage scenarios.

## Table of Contents

1. [Basic Setup Examples](#basic-setup-examples)
2. [Authentication Patterns](#authentication-patterns)
3. [Message Handling](#message-handling)
4. [Room Management](#room-management)
5. [Encryption Examples](#encryption-examples)
6. [Event Processing](#event-processing)
7. [TUI Integration Patterns](#tui-integration-patterns)
8. [Error Handling Patterns](#error-handling-patterns)
9. [Performance Patterns](#performance-patterns)
10. [Complete Applications](#complete-applications)

## Basic Setup Examples

### Minimal Client Setup

```go
package main

import (
    "context"
    "log"
    "os"
    "path/filepath"

    "github.com/gomuks/gomuks/pkg/hicli"
)

func main() {
    // Setup data directories
    homeDir, _ := os.UserHomeDir()
    dataDir := filepath.Join(homeDir, ".config", "mymatrix")

    // Ensure directories exist
    os.MkdirAll(dataDir, 0700)

    // Create configuration
    cfg := &hicli.Config{
        DataDir:      dataDir,
        CacheDir:     filepath.Join(homeDir, ".cache", "mymatrix"),
        DeviceName:   "My Matrix Client",
        PickleKey:    generatePickleKey(),
        EventHandler: handleMatrixEvent,
    }

    // Create client
    client, err := hicli.NewHiClient(cfg)
    if err != nil {
        log.Fatal("Failed to create client:", err)
    }

    // Start client
    ctx := context.Background()
    if err := client.Start(ctx); err != nil {
        log.Fatal("Failed to start client:", err)
    }

    log.Println("Matrix client started successfully")

    // Keep running
    select {}
}

func generatePickleKey() []byte {
    // In production, derive from password or read from secure storage
    return []byte("secure-32-byte-key-for-encryption!!")
}

func handleMatrixEvent(evt any) {
    log.Printf("Received event: %T", evt)
}
```

### Configuration with Environment Variables

```go
package main

import (
    "crypto/rand"
    "encoding/hex"
    "os"
    "strconv"

    "github.com/gomuks/gomuks/pkg/hicli"
)

type Config struct {
    DataDir    string
    CacheDir   string
    DeviceName string
    PickleKey  []byte
    LogLevel   string
}

func loadConfig() *Config {
    cfg := &Config{
        DataDir:    getEnvOrDefault("MATRIX_DATA_DIR", "./data"),
        CacheDir:   getEnvOrDefault("MATRIX_CACHE_DIR", "./cache"),
        DeviceName: getEnvOrDefault("MATRIX_DEVICE_NAME", "hicli Client"),
        LogLevel:   getEnvOrDefault("MATRIX_LOG_LEVEL", "info"),
    }

    // Load or generate pickle key
    pickleKeyHex := os.Getenv("MATRIX_PICKLE_KEY")
    if pickleKeyHex != "" {
        key, err := hex.DecodeString(pickleKeyHex)
        if err != nil {
            log.Fatal("Invalid pickle key format:", err)
        }
        cfg.PickleKey = key
    } else {
        // Generate new key and warn user
        cfg.PickleKey = generateRandomKey(32)
        log.Printf("Generated new pickle key: %s", hex.EncodeToString(cfg.PickleKey))
        log.Println("Set MATRIX_PICKLE_KEY environment variable to persist this key")
    }

    return cfg
}

func getEnvOrDefault(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func generateRandomKey(length int) []byte {
    key := make([]byte, length)
    rand.Read(key)
    return key
}

func main() {
    config := loadConfig()

    hicliCfg := &hicli.Config{
        DataDir:      config.DataDir,
        CacheDir:     config.CacheDir,
        DeviceName:   config.DeviceName,
        PickleKey:    config.PickleKey,
        EventHandler: handleEvent,
    }

    client, err := hicli.NewHiClient(hicliCfg)
    if err != nil {
        log.Fatal(err)
    }

    // Start and run
    ctx := context.Background()
    client.Start(ctx)

    // Application logic here
}
```

## Authentication Patterns

### Interactive Login Flow

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "os"
    "strings"
    "syscall"

    "github.com/gomuks/gomuks/pkg/hicli"
    "golang.org/x/term"
)

type LoginManager struct {
    client *hicli.HiClient
    reader *bufio.Reader
}

func NewLoginManager(client *hicli.HiClient) *LoginManager {
    return &LoginManager{
        client: client,
        reader: bufio.NewReader(os.Stdin),
    }
}

func (lm *LoginManager) InteractiveLogin(ctx context.Context) error {
    // Check if already logged in
    if lm.client.Account != nil && lm.client.Account.AccessToken != "" {
        fmt.Println("Already logged in as", lm.client.Account.UserID)
        return nil
    }

    fmt.Println("Matrix Login")
    fmt.Println("============")

    // Get homeserver
    fmt.Print("Homeserver (e.g., matrix.org): ")
    homeserver, _ := lm.reader.ReadString('\n')
    homeserver = strings.TrimSpace(homeserver)
    if !strings.HasPrefix(homeserver, "http") {
        homeserver = "https://" + homeserver
    }

    // Get username
    fmt.Print("Username: ")
    username, _ := lm.reader.ReadString('\n')
    username = strings.TrimSpace(username)

    // Get password securely
    fmt.Print("Password: ")
    passwordBytes, err := term.ReadPassword(int(syscall.Stdin))
    if err != nil {
        return fmt.Errorf("failed to read password: %w", err)
    }
    password := string(passwordBytes)
    fmt.Println() // New line after password input

    // Attempt login
    fmt.Println("Logging in...")
    loginReq := &hicli.LoginRequest{
        Homeserver: homeserver,
        Username:   username,
        Password:   password,
    }

    resp, err := lm.client.Login(ctx, loginReq)
    if err != nil {
        return fmt.Errorf("login failed: %w", err)
    }

    fmt.Printf("Successfully logged in as %s\n", resp.UserID)
    fmt.Printf("Device ID: %s\n", resp.DeviceID)

    return nil
}

func (lm *LoginManager) HandleDeviceVerification(ctx context.Context) error {
    if lm.client.Verified {
        fmt.Println("Device is already verified")
        return nil
    }

    fmt.Println("\nDevice Verification Required")
    fmt.Println("============================")
    fmt.Println("This device needs to be verified for full functionality.")
    fmt.Println("Options:")
    fmt.Println("1. Use recovery key")
    fmt.Println("2. Verify with another device")
    fmt.Println("3. Skip verification (limited functionality)")

    fmt.Print("Choose option (1-3): ")
    choice, _ := lm.reader.ReadString('\n')
    choice = strings.TrimSpace(choice)

    switch choice {
    case "1":
        return lm.verifyWithRecoveryKey(ctx)
    case "2":
        fmt.Println("Interactive verification not implemented in this example")
        return nil
    case "3":
        fmt.Println("Verification skipped")
        return nil
    default:
        fmt.Println("Invalid choice")
        return lm.HandleDeviceVerification(ctx)
    }
}

func (lm *LoginManager) verifyWithRecoveryKey(ctx context.Context) error {
    fmt.Print("Enter recovery key: ")
    recoveryKey, _ := lm.reader.ReadString('\n')
    recoveryKey = strings.TrimSpace(recoveryKey)

    fmt.Println("Verifying device...")
    err := lm.client.VerifyWithRecoveryKey(ctx, recoveryKey)
    if err != nil {
        return fmt.Errorf("verification failed: %w", err)
    }

    fmt.Println("Device verified successfully!")
    return nil
}

func main() {
    // Setup client
    cfg := &hicli.Config{
        DataDir:      "./data",
        CacheDir:     "./cache",
        DeviceName:   "Interactive CLI Client",
        PickleKey:    []byte("secure-key-32-bytes-long-example!"),
        EventHandler: func(evt any) { /* handle events */ },
    }

    client, err := hicli.NewHiClient(cfg)
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()
    client.Start(ctx)

    // Handle login
    loginMgr := NewLoginManager(client)
    if err := loginMgr.InteractiveLogin(ctx); err != nil {
        log.Fatal("Login failed:", err)
    }

    // Handle verification
    if err := loginMgr.HandleDeviceVerification(ctx); err != nil {
        log.Fatal("Verification failed:", err)
    }

    fmt.Println("Ready to use Matrix!")
    select {} // Keep running
}
```

### Token-Based Login

```go
func loginWithToken(client *hicli.HiClient, homeserver, token string) error {
    ctx := context.Background()

    req := &hicli.LoginRequest{
        Homeserver: homeserver,
        Token:      token,
        DeviceName: "Token-based Client",
    }

    resp, err := client.Login(ctx, req)
    if err != nil {
        return fmt.Errorf("token login failed: %w", err)
    }

    log.Printf("Logged in as %s with device %s", resp.UserID, resp.DeviceID)
    return nil
}
```

## Message Handling

### Rich Message Sending

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "mime"
    "os"
    "path/filepath"

    "github.com/gomuks/gomuks/pkg/hicli"
    "maunium.net/go/mautrix/event"
    "maunium.net/go/mautrix/id"
)

type MessageSender struct {
    client *hicli.HiClient
}

func NewMessageSender(client *hicli.HiClient) *MessageSender {
    return &MessageSender{client: client}
}

// Send plain text message
func (ms *MessageSender) SendText(roomID id.RoomID, text string) error {
    req := &hicli.SendMessageRequest{
        RoomID:      roomID,
        Content:     text,
        ContentType: "text/plain",
    }

    ctx := context.Background()
    resp, err := ms.client.SendMessage(ctx, req)
    if err != nil {
        return err
    }

    fmt.Printf("Sent message %s\n", resp.EventID)
    return nil
}

// Send markdown message
func (ms *MessageSender) SendMarkdown(roomID id.RoomID, markdown string) error {
    req := &hicli.SendMessageRequest{
        RoomID:      roomID,
        Content:     markdown,
        ContentType: "text/markdown",
    }

    ctx := context.Background()
    _, err := ms.client.SendMessage(ctx, req)
    return err
}

// Send emote action
func (ms *MessageSender) SendEmote(roomID id.RoomID, action string) error {
    content := map[string]any{
        "msgtype": "m.emote",
        "body":    action,
    }

    req := &hicli.SendEventRequest{
        RoomID:    roomID,
        EventType: event.EventMessage,
        Content:   content,
    }

    ctx := context.Background()
    _, err := ms.client.SendEvent(ctx, req)
    return err
}

// Send reply to another message
func (ms *MessageSender) SendReply(roomID id.RoomID, replyToEventID id.EventID, text string) error {
    req := &hicli.SendMessageRequest{
        RoomID:      roomID,
        Content:     text,
        ContentType: "text/markdown",
        ReplyTo:     replyToEventID,
    }

    ctx := context.Background()
    _, err := ms.client.SendMessage(ctx, req)
    return err
}

// Edit existing message
func (ms *MessageSender) EditMessage(roomID id.RoomID, eventID id.EventID, newText string) error {
    req := &hicli.SendMessageRequest{
        RoomID:      roomID,
        Content:     newText,
        ContentType: "text/markdown",
        Edit:        eventID,
    }

    ctx := context.Background()
    _, err := ms.client.SendMessage(ctx, req)
    return err
}

// Send file
func (ms *MessageSender) SendFile(roomID id.RoomID, filePath string) error {
    // Read file
    data, err := os.ReadFile(filePath)
    if err != nil {
        return fmt.Errorf("failed to read file: %w", err)
    }

    // Get file info
    filename := filepath.Base(filePath)
    mimeType := mime.TypeByExtension(filepath.Ext(filePath))
    if mimeType == "" {
        mimeType = "application/octet-stream"
    }

    // Upload file
    ctx := context.Background()
    uploadResp, err := ms.client.UploadMedia(ctx, data, mimeType, filename)
    if err != nil {
        return fmt.Errorf("failed to upload file: %w", err)
    }

    // Send file message
    content := map[string]any{
        "msgtype": "m.file",
        "body":    filename,
        "url":     uploadResp.ContentURI,
        "info": map[string]any{
            "mimetype": mimeType,
            "size":     len(data),
        },
    }

    req := &hicli.SendEventRequest{
        RoomID:    roomID,
        EventType: event.EventMessage,
        Content:   content,
    }

    _, err = ms.client.SendEvent(ctx, req)
    if err != nil {
        return fmt.Errorf("failed to send file message: %w", err)
    }

    fmt.Printf("Sent file %s (%d bytes)\n", filename, len(data))
    return nil
}

// Send reaction
func (ms *MessageSender) SendReaction(roomID id.RoomID, eventID id.EventID, emoji string) error {
    content := map[string]any{
        "m.relates_to": map[string]any{
            "rel_type": "m.annotation",
            "event_id": eventID,
            "key":      emoji,
        },
    }

    req := &hicli.SendEventRequest{
        RoomID:    roomID,
        EventType: event.EventReaction,
        Content:   content,
    }

    ctx := context.Background()
    _, err := ms.client.SendEvent(ctx, req)
    return err
}

func main() {
    // Setup client (configuration omitted for brevity)
    client := setupClient()
    sender := NewMessageSender(client)

    roomID := id.RoomID("!example:matrix.org")

    // Examples of different message types
    sender.SendText(roomID, "Hello, World!")
    sender.SendMarkdown(roomID, "**Bold** and *italic* text")
    sender.SendEmote(roomID, "waves enthusiastically")
    sender.SendFile(roomID, "./document.pdf")

    // React to a message
    eventID := id.EventID("$example:matrix.org")
    sender.SendReaction(roomID, eventID, "👍")
}
```

### Message Processing Pipeline

```go
type MessageProcessor struct {
    client *hicli.HiClient
    filters []MessageFilter
}

type MessageFilter interface {
    ProcessMessage(event *database.Event) (*database.Event, error)
    ShouldProcess(event *database.Event) bool
}

// URL Link Preview Filter
type LinkPreviewFilter struct {
    client *hicli.HiClient
}

func (f *LinkPreviewFilter) ShouldProcess(event *database.Event) bool {
    if event.Type != event.EventMessage {
        return false
    }

    var content struct {
        MsgType string `json:"msgtype"`
        Body    string `json:"body"`
    }
    json.Unmarshal(event.Content, &content)

    return content.MsgType == "m.text" && containsURL(content.Body)
}

func (f *LinkPreviewFilter) ProcessMessage(event *database.Event) (*database.Event, error) {
    var content struct {
        MsgType string `json:"msgtype"`
        Body    string `json:"body"`
    }
    json.Unmarshal(event.Content, &content)

    urls := extractURLs(content.Body)
    for _, url := range urls {
        preview, err := generatePreview(url)
        if err != nil {
            continue
        }

        // Store preview data (implementation specific)
        storePreview(event.ID, url, preview)
    }

    return event, nil
}

// Mention Detection Filter
type MentionFilter struct {
    userID id.UserID
}

func (f *MentionFilter) ShouldProcess(event *database.Event) bool {
    return event.Type == event.EventMessage && event.Sender != f.userID
}

func (f *MentionFilter) ProcessMessage(event *database.Event) (*database.Event, error) {
    var content struct {
        Body string `json:"body"`
    }
    json.Unmarshal(event.Content, &content)

    // Check for @username mentions
    if strings.Contains(content.Body, f.userID.String()) {
        event.Metadata = map[string]any{
            "mentioned": true,
            "highlight": true,
        }
    }

    return event, nil
}

func (mp *MessageProcessor) ProcessEvent(event *database.Event) error {
    for _, filter := range mp.filters {
        if filter.ShouldProcess(event) {
            processed, err := filter.ProcessMessage(event)
            if err != nil {
                log.Printf("Filter error: %v", err)
                continue
            }
            event = processed
        }
    }
    return nil
}
```

## Room Management

### Room Discovery and Joining

```go
type RoomManager struct {
    client *hicli.HiClient
}

func NewRoomManager(client *hicli.HiClient) *RoomManager {
    return &RoomManager{client: client}
}

// Join room by ID or alias
func (rm *RoomManager) JoinRoom(roomIDOrAlias string, via []string) error {
    ctx := context.Background()

    req := &hicli.JoinRoomRequest{
        RoomID: roomIDOrAlias,
        Via:    via,
    }

    resp, err := rm.client.JoinRoom(ctx, req)
    if err != nil {
        return fmt.Errorf("failed to join room %s: %w", roomIDOrAlias, err)
    }

    fmt.Printf("Joined room: %s\n", resp.RoomID)
    return nil
}

// Create a new room with specific settings
func (rm *RoomManager) CreateRoom(options RoomCreateOptions) (id.RoomID, error) {
    req := &hicli.CreateRoomRequest{
        Name:       options.Name,
        Topic:      options.Topic,
        Visibility: options.Visibility,
        Preset:     options.Preset,
        Invite:     options.Invites,
    }

    // Add initial state events
    if options.Encrypted {
        req.InitialState = append(req.InitialState, event.Event{
            Type:     event.StateEncryption,
            StateKey: "",
            Content: event.EncryptionEventContent{
                Algorithm: "m.megolm.v1.aes-sha2",
            },
        })
    }

    ctx := context.Background()
    resp, err := rm.client.CreateRoom(ctx, req)
    if err != nil {
        return "", fmt.Errorf("failed to create room: %w", err)
    }

    fmt.Printf("Created room: %s (%s)\n", resp.RoomID, options.Name)
    return resp.RoomID, nil
}

type RoomCreateOptions struct {
    Name       string
    Topic      string
    Visibility string     // "public" or "private"
    Preset     string     // "private_chat", "public_chat", "trusted_private_chat"
    Encrypted  bool
    Invites    []id.UserID
}

// Invite user to room
func (rm *RoomManager) InviteUser(roomID id.RoomID, userID id.UserID) error {
    content := map[string]any{
        "membership": "invite",
    }

    req := &hicli.SetStateRequest{
        RoomID:    roomID,
        EventType: event.StateMember,
        StateKey:  userID.String(),
        Content:   content,
    }

    ctx := context.Background()
    err := rm.client.SetState(ctx, req)
    if err != nil {
        return fmt.Errorf("failed to invite %s: %w", userID, err)
    }

    fmt.Printf("Invited %s to room\n", userID)
    return nil
}

// Set room topic
func (rm *RoomManager) SetRoomTopic(roomID id.RoomID, topic string) error {
    content := map[string]any{
        "topic": topic,
    }

    req := &hicli.SetStateRequest{
        RoomID:    roomID,
        EventType: event.StateRoomTopic,
        StateKey:  "",
        Content:   content,
    }

    ctx := context.Background()
    return rm.client.SetState(ctx, req)
}

// Get room members
func (rm *RoomManager) GetRoomMembers(roomID id.RoomID) ([]*RoomMember, error) {
    members, err := rm.client.GetRoomMembers(roomID)
    if err != nil {
        return nil, err
    }

    var result []*RoomMember
    for _, member := range members {
        var content struct {
            Membership  string `json:"membership"`
            Displayname string `json:"displayname,omitempty"`
            AvatarURL   string `json:"avatar_url,omitempty"`
        }

        if err := json.Unmarshal(member.Content, &content); err != nil {
            continue
        }

        result = append(result, &RoomMember{
            UserID:      id.UserID(member.StateKey),
            Membership:  content.Membership,
            Displayname: content.Displayname,
            AvatarURL:   content.AvatarURL,
        })
    }

    return result, nil
}

type RoomMember struct {
    UserID      id.UserID
    Membership  string
    Displayname string
    AvatarURL   string
}
```

### Space Management

```go
type SpaceManager struct {
    client *hicli.HiClient
}

func NewSpaceManager(client *hicli.HiClient) *SpaceManager {
    return &SpaceManager{client: client}
}

// Create a new space
func (sm *SpaceManager) CreateSpace(name, topic string, public bool) (id.RoomID, error) {
    visibility := "private"
    if public {
        visibility = "public"
    }

    req := &hicli.CreateRoomRequest{
        Name:       name,
        Topic:      topic,
        Visibility: visibility,
        CreationContent: map[string]any{
            "type": "m.space",
        },
    }

    ctx := context.Background()
    resp, err := sm.client.CreateRoom(ctx, req)
    if err != nil {
        return "", fmt.Errorf("failed to create space: %w", err)
    }

    fmt.Printf("Created space: %s\n", resp.RoomID)
    return resp.RoomID, nil
}

// Add room to space
func (sm *SpaceManager) AddRoomToSpace(spaceID, roomID id.RoomID, order string, suggested bool) error {
    ctx := context.Background()
    return sm.client.AddRoomToSpace(ctx, spaceID, roomID, order, suggested)
}

// Remove room from space
func (sm *SpaceManager) RemoveRoomFromSpace(spaceID, roomID id.RoomID) error {
    ctx := context.Background()
    return sm.client.RemoveRoomFromSpace(ctx, spaceID, roomID)
}

// Get space hierarchy
func (sm *SpaceManager) GetSpaceHierarchy(spaceID id.RoomID) (*SpaceHierarchy, error) {
    ctx := context.Background()
    return sm.client.GetSpaceHierarchy(ctx, spaceID)
}

// List all spaces user is a member of
func (sm *SpaceManager) GetUserSpaces() ([]*database.Room, error) {
    rooms, err := sm.client.GetRooms()
    if err != nil {
        return nil, err
    }

    var spaces []*database.Room
    for _, room := range rooms {
        if room.Type == "m.space" {
            spaces = append(spaces, room)
        }
    }

    return spaces, nil
}
```

## Encryption Examples

### Device Verification Flow

```go
type VerificationManager struct {
    client *hicli.HiClient
}

func NewVerificationManager(client *hicli.HiClient) *VerificationManager {
    return &VerificationManager{client: client}
}

// Start SAS verification with another device
func (vm *VerificationManager) StartSASVerification(userID id.UserID, deviceID id.DeviceID) error {
    req := &hicli.VerificationRequest{
        UserID:   userID,
        DeviceID: deviceID,
        Methods:  []string{"m.sas.v1"},
    }

    ctx := context.Background()
    verification, err := vm.client.StartVerification(ctx, req)
    if err != nil {
        return fmt.Errorf("failed to start verification: %w", err)
    }

    fmt.Printf("Started verification: %s\n", verification.ID)
    fmt.Println("Follow the verification process in your other client")

    return nil
}

// Bootstrap cross-signing for account
func (vm *VerificationManager) BootstrapCrossSigning(password string) error {
    // In a real implementation, you'd collect auth data properly
    authData := map[string]any{
        "type":     "m.login.password",
        "password": password,
    }

    ctx := context.Background()
    err := vm.client.BootstrapCrossSigning(ctx, authData)
    if err != nil {
        return fmt.Errorf("failed to bootstrap cross-signing: %w", err)
    }

    fmt.Println("Cross-signing bootstrapped successfully")
    return nil
}

// Verify using recovery key
func (vm *VerificationManager) VerifyWithRecoveryKey(recoveryKey string) error {
    ctx := context.Background()
    err := vm.client.VerifyWithRecoveryKey(ctx, recoveryKey)
    if err != nil {
        return fmt.Errorf("verification with recovery key failed: %w", err)
    }

    fmt.Println("Device verified with recovery key")
    return nil
}

// Setup key backup
func (vm *VerificationManager) SetupKeyBackup(passphrase string) (*KeyBackupInfo, error) {
    ctx := context.Background()
    backup, err := vm.client.CreateKeyBackup(ctx, passphrase)
    if err != nil {
        return nil, fmt.Errorf("failed to create key backup: %w", err)
    }

    // Generate recovery key from passphrase (simplified)
    recoveryKey := generateRecoveryKey(passphrase, backup.Version)

    return &KeyBackupInfo{
        Version:     backup.Version,
        RecoveryKey: recoveryKey,
    }, nil
}

type KeyBackupInfo struct {
    Version     string
    RecoveryKey string
}

func generateRecoveryKey(passphrase, version string) string {
    // This is a simplified example - real implementation would use proper key derivation
    return fmt.Sprintf("recovery-key-from-%s-%s", passphrase, version)
}
```

### Manual Key Management

```go
type KeyManager struct {
    client *hicli.HiClient
}

func NewKeyManager(client *hicli.HiClient) *KeyManager {
    return &KeyManager{client: client}
}

// Upload one-time keys
func (km *KeyManager) UploadKeys() error {
    ctx := context.Background()
    err := km.client.UploadKeys(ctx)
    if err != nil {
        return fmt.Errorf("failed to upload keys: %w", err)
    }

    fmt.Println("Keys uploaded successfully")
    return nil
}

// Query keys for users
func (km *KeyManager) QueryUserKeys(userIDs []id.UserID) error {
    ctx := context.Background()
    err := km.client.QueryKeys(ctx, userIDs)
    if err != nil {
        return fmt.Errorf("failed to query keys: %w", err)
    }

    fmt.Printf("Queried keys for %d users\n", len(userIDs))
    return nil
}

// Share room keys with specific users
func (km *KeyManager) ShareRoomKeys(roomID id.RoomID, userIDs []id.UserID) error {
    ctx := context.Background()
    err := km.client.ShareRoomKeys(ctx, roomID, userIDs)
    if err != nil {
        return fmt.Errorf("failed to share room keys: %w", err)
    }

    fmt.Printf("Shared room keys with %d users\n", len(userIDs))
    return nil
}

// Export room keys (for backup)
func (km *KeyManager) ExportRoomKeys(passphrase string) ([]byte, error) {
    // This would export encrypted room keys
    // Implementation depends on crypto library
    fmt.Println("Exporting room keys...")
    return nil, fmt.Errorf("not implemented")
}

// Import room keys (from backup)
func (km *KeyManager) ImportRoomKeys(keyData []byte, passphrase string) error {
    // This would import encrypted room keys
    // Implementation depends on crypto library
    fmt.Println("Importing room keys...")
    return fmt.Errorf("not implemented")
}
```

## Event Processing

### Event Filter and Router

```go
type EventRouter struct {
    handlers map[string][]EventHandler
    filters  []EventFilter
}

type EventHandler func(event any) error
type EventFilter func(event any) bool

func NewEventRouter() *EventRouter {
    return &EventRouter{
        handlers: make(map[string][]EventHandler),
        filters:  make([]EventFilter, 0),
    }
}

func (er *EventRouter) AddHandler(eventType string, handler EventHandler) {
    er.handlers[eventType] = append(er.handlers[eventType], handler)
}

func (er *EventRouter) AddFilter(filter EventFilter) {
    er.filters = append(er.filters, filter)
}

func (er *EventRouter) RouteEvent(event any) error {
    // Apply filters
    for _, filter := range er.filters {
        if !filter(event) {
            return nil // Event filtered out
        }
    }

    // Determine event type
    eventType := fmt.Sprintf("%T", event)

    // Route to handlers
    handlers, exists := er.handlers[eventType]
    if !exists {
        return nil // No handlers for this event type
    }

    for _, handler := range handlers {
        if err := handler(event); err != nil {
            log.Printf("Handler error for %s: %v", eventType, err)
        }
    }

    return nil
}

// Example usage
func setupEventRouter(client *hicli.HiClient) *EventRouter {
    router := NewEventRouter()

    // Add filters
    router.AddFilter(func(event any) bool {
        // Only process events during business hours
        hour := time.Now().Hour()
        return hour >= 9 && hour <= 17
    })

    // Add handlers
    router.AddHandler("*jsoncmd.SyncComplete", func(event any) error {
        syncEvent := event.(*jsoncmd.SyncComplete)
        fmt.Printf("Sync complete: %d rooms updated\n", len(syncEvent.Rooms))
        return nil
    })

    router.AddHandler("*jsoncmd.EventsDecrypted", func(event any) error {
        decryptedEvent := event.(*jsoncmd.EventsDecrypted)
        for _, evt := range decryptedEvent.Events {
            fmt.Printf("New message in %s: %s\n", evt.RoomName, evt.Event.ID)
        }
        return nil
    })

    router.AddHandler("*jsoncmd.SyncStatus", func(event any) error {
        statusEvent := event.(*jsoncmd.SyncStatus)
        if statusEvent.State == "error" {
            fmt.Printf("Sync error: %s\n", statusEvent.Error)
        }
        return nil
    })

    return router
}
```

### Event Persistence and Replay

```go
type EventStore struct {
    events []StoredEvent
    mu     sync.RWMutex
}

type StoredEvent struct {
    Timestamp time.Time
    Type      string
    Data      any
}

func NewEventStore() *EventStore {
    return &EventStore{
        events: make([]StoredEvent, 0),
    }
}

func (es *EventStore) StoreEvent(event any) {
    es.mu.Lock()
    defer es.mu.Unlock()

    stored := StoredEvent{
        Timestamp: time.Now(),
        Type:      fmt.Sprintf("%T", event),
        Data:      event,
    }

    es.events = append(es.events, stored)

    // Keep only last 1000 events
    if len(es.events) > 1000 {
        es.events = es.events[len(es.events)-1000:]
    }
}

func (es *EventStore) GetEvents(since time.Time) []StoredEvent {
    es.mu.RLock()
    defer es.mu.RUnlock()

    var result []StoredEvent
    for _, event := range es.events {
        if event.Timestamp.After(since) {
            result = append(result, event)
        }
    }

    return result
}

func (es *EventStore) ReplayEvents(since time.Time, handler func(any)) {
    events := es.GetEvents(since)
    for _, event := range events {
        handler(event.Data)
    }
}
```

## TUI Integration Patterns

### Reactive UI Updates

```go
type ReactiveUI struct {
    app         *mauview.Application
    roomList    *mauview.TextView
    messageArea *mauview.TextView
    inputField  *mauview.InputField

    // State
    rooms       []*database.Room
    currentRoom id.RoomID
    messages    []*database.Event

    // Update channels
    roomUpdates    chan []*database.Room
    messageUpdates chan *database.Event
    statusUpdates  chan string
}

func NewReactiveUI() *ReactiveUI {
    ui := &ReactiveUI{
        app:            mauview.NewApplication(),
        roomList:       mauview.NewTextView(),
        messageArea:    mauview.NewTextView(),
        inputField:     mauview.NewInputField(),
        roomUpdates:    make(chan []*database.Room, 10),
        messageUpdates: make(chan *database.Event, 100),
        statusUpdates:  make(chan string, 10),
    }

    ui.setupLayout()
    ui.startUpdateLoop()

    return ui
}

func (ui *ReactiveUI) setupLayout() {
    // Setup mauview layout (omitted for brevity)
    // ...
}

func (ui *ReactiveUI) startUpdateLoop() {
    go func() {
        ticker := time.NewTicker(100 * time.Millisecond)
        defer ticker.Stop()

        var pendingRoomUpdate bool
        var pendingMessages []*database.Event
        var pendingStatus string

        for {
            select {
            case rooms := <-ui.roomUpdates:
                ui.rooms = rooms
                pendingRoomUpdate = true

            case msg := <-ui.messageUpdates:
                if msg.RoomID == ui.currentRoom {
                    pendingMessages = append(pendingMessages, msg)
                }

            case status := <-ui.statusUpdates:
                pendingStatus = status

            case <-ticker.C:
                // Batch updates
                if pendingRoomUpdate {
                    ui.updateRoomList()
                    pendingRoomUpdate = false
                }

                if len(pendingMessages) > 0 {
                    ui.updateMessages(pendingMessages)
                    pendingMessages = nil
                }

                if pendingStatus != "" {
                    ui.updateStatus(pendingStatus)
                    pendingStatus = ""
                }

                // Trigger redraw if any updates occurred
                if pendingRoomUpdate || len(pendingMessages) > 0 || pendingStatus != "" {
                    ui.app.RedrawSoon()
                }
            }
        }
    }()
}

func (ui *ReactiveUI) HandleMatrixEvent(event any) {
    switch e := event.(type) {
    case *jsoncmd.SyncComplete:
        ui.roomUpdates <- e.Rooms

    case *jsoncmd.EventsDecrypted:
        for _, evt := range e.Events {
            ui.messageUpdates <- evt.Event
        }

    case *jsoncmd.SyncStatus:
        status := fmt.Sprintf("Status: %s", e.State)
        ui.statusUpdates <- status
    }
}

func (ui *ReactiveUI) updateRoomList() {
    var content strings.Builder
    for _, room := range ui.rooms {
        name := room.Name
        if name == "" {
            name = room.ID.String()
        }

        unread := ""
        if room.UnreadCount > 0 {
            unread = fmt.Sprintf(" (%d)", room.UnreadCount)
        }

        content.WriteString(fmt.Sprintf("%s%s\n", name, unread))
    }
    ui.roomList.SetText(content.String())
}

func (ui *ReactiveUI) updateMessages(messages []*database.Event) {
    for _, msg := range messages {
        ui.messages = append(ui.messages, msg)
    }

    // Render messages
    var content strings.Builder
    for _, msg := range ui.messages {
        ui.renderMessage(&content, msg)
    }
    ui.messageArea.SetText(content.String())
    ui.messageArea.ScrollToEnd()
}

func (ui *ReactiveUI) renderMessage(content *strings.Builder, event *database.Event) {
    timestamp := event.Timestamp.Format("15:04")
    sender := event.Sender.String()

    // Extract message body
    var msgContent struct {
        Body string `json:"body"`
    }
    json.Unmarshal(event.Content, &msgContent)

    content.WriteString(fmt.Sprintf("[gray]%s[white] [cyan]%s:[white] %s\n",
        timestamp, sender, msgContent.Body))
}

func (ui *ReactiveUI) updateStatus(status string) {
    // Update status bar or similar
    ui.app.SetTitle(status)
}
```

### Command Interface

```go
type CommandInterface struct {
    client   *hicli.HiClient
    commands map[string]Command
}

type Command interface {
    Execute(args []string) error
    Help() string
}

func NewCommandInterface(client *hicli.HiClient) *CommandInterface {
    ci := &CommandInterface{
        client:   client,
        commands: make(map[string]Command),
    }

    // Register built-in commands
    ci.RegisterCommand("join", &JoinCommand{client: client})
    ci.RegisterCommand("leave", &LeaveCommand{client: client})
    ci.RegisterCommand("create", &CreateRoomCommand{client: client})
    ci.RegisterCommand("invite", &InviteCommand{client: client})
    ci.RegisterCommand("help", &HelpCommand{ci: ci})

    return ci
}

func (ci *CommandInterface) RegisterCommand(name string, cmd Command) {
    ci.commands[name] = cmd
}

func (ci *CommandInterface) ExecuteCommand(input string) error {
    parts := strings.Fields(input)
    if len(parts) == 0 {
        return fmt.Errorf("empty command")
    }

    cmdName := parts[0]
    args := parts[1:]

    cmd, exists := ci.commands[cmdName]
    if !exists {
        return fmt.Errorf("unknown command: %s", cmdName)
    }

    return cmd.Execute(args)
}

// Example command implementations
type JoinCommand struct {
    client *hicli.HiClient
}

func (c *JoinCommand) Execute(args []string) error {
    if len(args) != 1 {
        return fmt.Errorf("usage: join <room-id-or-alias>")
    }

    req := &hicli.JoinRoomRequest{
        RoomID: args[0],
    }

    ctx := context.Background()
    _, err := c.client.JoinRoom(ctx, req)
    return err
}

func (c *JoinCommand) Help() string {
    return "join <room-id-or-alias> - Join a room"
}

type CreateRoomCommand struct {
    client *hicli.HiClient
}

func (c *CreateRoomCommand) Execute(args []string) error {
    if len(args) != 1 {
        return fmt.Errorf("usage: create <room-name>")
    }

    req := &hicli.CreateRoomRequest{
        Name:       args[0],
        Visibility: "private",
        Preset:     "private_chat",
    }

    ctx := context.Background()
    resp, err := c.client.CreateRoom(ctx, req)
    if err != nil {
        return err
    }

    fmt.Printf("Created room: %s\n", resp.RoomID)
    return nil
}

func (c *CreateRoomCommand) Help() string {
    return "create <room-name> - Create a new private room"
}
```

## Error Handling Patterns

### Retry Logic with Exponential Backoff

```go
type RetryManager struct {
    maxRetries int
    baseDelay  time.Duration
}

func NewRetryManager(maxRetries int, baseDelay time.Duration) *RetryManager {
    return &RetryManager{
        maxRetries: maxRetries,
        baseDelay:  baseDelay,
    }
}

func (rm *RetryManager) RetryWithBackoff(operation func() error) error {
    var lastErr error

    for attempt := 0; attempt <= rm.maxRetries; attempt++ {
        err := operation()
        if err == nil {
            return nil // Success
        }

        lastErr = err

        // Check if error is retryable
        if !isRetryableError(err) {
            return err
        }

        if attempt < rm.maxRetries {
            delay := rm.baseDelay * time.Duration(1<<attempt) // Exponential backoff
            log.Printf("Operation failed (attempt %d/%d), retrying in %v: %v",
                attempt+1, rm.maxRetries+1, delay, err)
            time.Sleep(delay)
        }
    }

    return fmt.Errorf("operation failed after %d attempts: %w", rm.maxRetries+1, lastErr)
}

func isRetryableError(err error) bool {
    if httpErr, ok := err.(*mautrix.HTTPError); ok {
        switch httpErr.ResponseCode {
        case 429: // Rate limited
            return true
        case 500, 502, 503, 504: // Server errors
            return true
        default:
            return false
        }
    }

    // Network errors are generally retryable
    if strings.Contains(err.Error(), "connection refused") ||
       strings.Contains(err.Error(), "timeout") {
        return true
    }

    return false
}

// Usage example
func sendMessageWithRetry(client *hicli.HiClient, roomID id.RoomID, content string) error {
    retryMgr := NewRetryManager(3, time.Second)

    return retryMgr.RetryWithBackoff(func() error {
        req := &hicli.SendMessageRequest{
            RoomID:  roomID,
            Content: content,
        }

        ctx := context.Background()
        _, err := client.SendMessage(ctx, req)
        return err
    })
}
```

### Circuit Breaker Pattern

```go
type CircuitBreaker struct {
    maxFailures  int
    resetTimeout time.Duration

    failures    int
    lastFailure time.Time
    state       CircuitState
    mu          sync.RWMutex
}

type CircuitState int

const (
    CircuitClosed CircuitState = iota
    CircuitOpen
    CircuitHalfOpen
)

func NewCircuitBreaker(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures:  maxFailures,
        resetTimeout: resetTimeout,
        state:        CircuitClosed,
    }
}

func (cb *CircuitBreaker) Execute(operation func() error) error {
    if !cb.canExecute() {
        return fmt.Errorf("circuit breaker is open")
    }

    err := operation()
    cb.recordResult(err)
    return err
}

func (cb *CircuitBreaker) canExecute() bool {
    cb.mu.RLock()
    defer cb.mu.RUnlock()

    switch cb.state {
    case CircuitClosed:
        return true
    case CircuitOpen:
        return time.Since(cb.lastFailure) > cb.resetTimeout
    case CircuitHalfOpen:
        return true
    default:
        return false
    }
}

func (cb *CircuitBreaker) recordResult(err error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = CircuitOpen
        }
    } else {
        cb.failures = 0
        cb.state = CircuitClosed
    }
}
```

## Performance Patterns

### Event Batching

```go
type EventBatcher struct {
    batchSize   int
    flushDelay  time.Duration
    processor   func([]*database.Event) error

    batch  []*database.Event
    timer  *time.Timer
    mu     sync.Mutex
}

func NewEventBatcher(batchSize int, flushDelay time.Duration, processor func([]*database.Event) error) *EventBatcher {
    return &EventBatcher{
        batchSize:  batchSize,
        flushDelay: flushDelay,
        processor:  processor,
        batch:      make([]*database.Event, 0, batchSize),
    }
}

func (eb *EventBatcher) AddEvent(event *database.Event) {
    eb.mu.Lock()
    defer eb.mu.Unlock()

    eb.batch = append(eb.batch, event)

    if len(eb.batch) >= eb.batchSize {
        eb.flush()
    } else if eb.timer == nil {
        eb.timer = time.AfterFunc(eb.flushDelay, func() {
            eb.mu.Lock()
            defer eb.mu.Unlock()
            eb.flush()
        })
    }
}

func (eb *EventBatcher) flush() {
    if len(eb.batch) == 0 {
        return
    }

    if eb.timer != nil {
        eb.timer.Stop()
        eb.timer = nil
    }

    batch := eb.batch
    eb.batch = make([]*database.Event, 0, eb.batchSize)

    go func() {
        if err := eb.processor(batch); err != nil {
            log.Printf("Batch processing error: %v", err)
        }
    }()
}

func (eb *EventBatcher) Flush() {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    eb.flush()
}
```

### Memory Pool for Events

```go
type EventPool struct {
    pool sync.Pool
}

func NewEventPool() *EventPool {
    return &EventPool{
        pool: sync.Pool{
            New: func() interface{} {
                return &database.Event{}
            },
        },
    }
}

func (ep *EventPool) GetEvent() *database.Event {
    event := ep.pool.Get().(*database.Event)
    // Reset event fields
    *event = database.Event{}
    return event
}

func (ep *EventPool) PutEvent(event *database.Event) {
    ep.pool.Put(event)
}

// Usage example
func processEventsWithPool(events []*database.Event, pool *EventPool) {
    for _, event := range events {
        // Process event
        processEvent(event)

        // Return to pool when done
        pool.PutEvent(event)
    }
}
```

## Complete Applications

### Simple Chat Bot

```go
package main

import (
    "context"
    "log"
    "regexp"
    "strings"
    "time"

    "github.com/gomuks/gomuks/pkg/hicli"
    "maunium.net/go/mautrix/id"
)

type ChatBot struct {
    client   *hicli.HiClient
    commands map[string]BotCommand
}

type BotCommand func(roomID id.RoomID, sender id.UserID, args []string) error

func NewChatBot(client *hicli.HiClient) *ChatBot {
    bot := &ChatBot{
        client:   client,
        commands: make(map[string]BotCommand),
    }

    // Register commands
    bot.commands["ping"] = bot.pingCommand
    bot.commands["time"] = bot.timeCommand
    bot.commands["echo"] = bot.echoCommand
    bot.commands["help"] = bot.helpCommand

    return bot
}

func (bot *ChatBot) HandleEvent(event any) {
    switch e := event.(type) {
    case *jsoncmd.EventsDecrypted:
        for _, evt := range e.Events {
            bot.processMessage(evt.Event)
        }
    }
}

func (bot *ChatBot) processMessage(event *database.Event) {
    if event.Type != "m.room.message" {
        return
    }

    // Don't respond to own messages
    if event.Sender == bot.client.Account.UserID {
        return
    }

    var content struct {
        MsgType string `json:"msgtype"`
        Body    string `json:"body"`
    }

    if err := json.Unmarshal(event.Content, &content); err != nil {
        return
    }

    if content.MsgType != "m.text" {
        return
    }

    // Check for bot commands (prefixed with !)
    if strings.HasPrefix(content.Body, "!") {
        bot.handleCommand(event.RoomID, event.Sender, content.Body[1:])
    }
}

func (bot *ChatBot) handleCommand(roomID id.RoomID, sender id.UserID, command string) {
    parts := strings.Fields(command)
    if len(parts) == 0 {
        return
    }

    cmdName := strings.ToLower(parts[0])
    args := parts[1:]

    cmd, exists := bot.commands[cmdName]
    if !exists {
        bot.sendMessage(roomID, "Unknown command. Use !help for available commands.")
        return
    }

    if err := cmd(roomID, sender, args); err != nil {
        log.Printf("Command error: %v", err)
        bot.sendMessage(roomID, "Error executing command.")
    }
}

func (bot *ChatBot) pingCommand(roomID id.RoomID, sender id.UserID, args []string) error {
    return bot.sendMessage(roomID, "Pong!")
}

func (bot *ChatBot) timeCommand(roomID id.RoomID, sender id.UserID, args []string) error {
    timeStr := time.Now().Format("2006-01-02 15:04:05 MST")
    return bot.sendMessage(roomID, fmt.Sprintf("Current time: %s", timeStr))
}

func (bot *ChatBot) echoCommand(roomID id.RoomID, sender id.UserID, args []string) error {
    if len(args) == 0 {
        return bot.sendMessage(roomID, "Usage: !echo <message>")
    }

    message := strings.Join(args, " ")
    return bot.sendMessage(roomID, fmt.Sprintf("Echo: %s", message))
}

func (bot *ChatBot) helpCommand(roomID id.RoomID, sender id.UserID, args []string) error {
    help := "Available commands:\n" +
           "!ping - Respond with pong\n" +
           "!time - Show current time\n" +
           "!echo <message> - Echo a message\n" +
           "!help - Show this help"

    return bot.sendMessage(roomID, help)
}

func (bot *ChatBot) sendMessage(roomID id.RoomID, text string) error {
    req := &hicli.SendMessageRequest{
        RoomID:  roomID,
        Content: text,
    }

    ctx := context.Background()
    _, err := bot.client.SendMessage(ctx, req)
    return err
}

func main() {
    // Setup client
    cfg := &hicli.Config{
        DataDir:      "./bot-data",
        CacheDir:     "./bot-cache",
        DeviceName:   "Chat Bot",
        PickleKey:    []byte("bot-pickle-key-32-bytes-long-!!!"),
        EventHandler: nil, // Will be set below
    }

    client, err := hicli.NewHiClient(cfg)
    if err != nil {
        log.Fatal(err)
    }

    // Create bot
    bot := NewChatBot(client)
    cfg.EventHandler = bot.HandleEvent

    // Start client
    ctx := context.Background()
    if err := client.Start(ctx); err != nil {
        log.Fatal(err)
    }

    // Login (hardcoded for example - use secure method in production)
    loginReq := &hicli.LoginRequest{
        Homeserver: "https://matrix.org",
        Username:   "botusername",
        Password:   "botpassword",
        DeviceName: "Chat Bot",
    }

    _, err = client.Login(ctx, loginReq)
    if err != nil {
        log.Fatal("Login failed:", err)
    }

    log.Println("Chat bot is running...")
    select {} // Keep running
}
```

### Room Monitor and Logger

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "path/filepath"
    "time"

    "github.com/gomuks/gomuks/pkg/hicli"
)

type RoomLogger struct {
    client    *hicli.HiClient
    logDir    string
    logFiles  map[id.RoomID]*os.File
    roomNames map[id.RoomID]string
}

func NewRoomLogger(client *hicli.HiClient, logDir string) *RoomLogger {
    return &RoomLogger{
        client:    client,
        logDir:    logDir,
        logFiles:  make(map[id.RoomID]*os.File),
        roomNames: make(map[id.RoomID]string),
    }
}

func (rl *RoomLogger) Start() error {
    // Create log directory
    if err := os.MkdirAll(rl.logDir, 0755); err != nil {
        return fmt.Errorf("failed to create log directory: %w", err)
    }

    return nil
}

func (rl *RoomLogger) HandleEvent(event any) {
    switch e := event.(type) {
    case *jsoncmd.SyncComplete:
        rl.updateRoomNames(e.Rooms)

    case *jsoncmd.EventsDecrypted:
        for _, evt := range e.Events {
            rl.logEvent(evt.Event)
        }
    }
}

func (rl *RoomLogger) updateRoomNames(rooms []*database.Room) {
    for _, room := range rooms {
        rl.roomNames[room.ID] = room.Name
    }
}

func (rl *RoomLogger) logEvent(event *database.Event) {
    roomID := event.RoomID

    // Get or create log file for room
    logFile, err := rl.getLogFile(roomID)
    if err != nil {
        log.Printf("Failed to get log file for room %s: %v", roomID, err)
        return
    }

    // Create log entry
    logEntry := LogEntry{
        Timestamp: event.Timestamp,
        EventID:   event.ID,
        EventType: event.Type,
        Sender:    event.Sender,
        Content:   event.Content,
        RoomID:    roomID,
        RoomName:  rl.roomNames[roomID],
    }

    // Write to file
    jsonData, err := json.Marshal(logEntry)
    if err != nil {
        log.Printf("Failed to marshal log entry: %v", err)
        return
    }

    if _, err := logFile.WriteString(string(jsonData) + "\n"); err != nil {
        log.Printf("Failed to write log entry: %v", err)
    }
}

type LogEntry struct {
    Timestamp time.Time       `json:"timestamp"`
    EventID   id.EventID      `json:"event_id"`
    EventType string          `json:"event_type"`
    Sender    id.UserID       `json:"sender"`
    Content   json.RawMessage `json:"content"`
    RoomID    id.RoomID       `json:"room_id"`
    RoomName  string          `json:"room_name"`
}

func (rl *RoomLogger) getLogFile(roomID id.RoomID) (*os.File, error) {
    if file, exists := rl.logFiles[roomID]; exists {
        return file, nil
    }

    // Create filename from room ID (sanitized)
    filename := fmt.Sprintf("room_%s.jsonl", sanitizeFilename(string(roomID)))
    filepath := filepath.Join(rl.logDir, filename)

    // Open file for append
    file, err := os.OpenFile(filepath, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        return nil, err
    }

    rl.logFiles[roomID] = file
    return file, nil
}

func sanitizeFilename(filename string) string {
    // Replace invalid characters with underscores
    re := regexp.MustCompile(`[<>:"/\\|?*]`)
    return re.ReplaceAllString(filename, "_")
}

func (rl *RoomLogger) Close() {
    for _, file := range rl.logFiles {
        file.Close()
    }
}

func main() {
    // Setup client and logger
    cfg := &hicli.Config{
        DataDir:    "./monitor-data",
        CacheDir:   "./monitor-cache",
        DeviceName: "Room Monitor",
        PickleKey:  []byte("monitor-key-32-bytes-long-!!!!!"),
    }

    client, err := hicli.NewHiClient(cfg)
    if err != nil {
        log.Fatal(err)
    }

    logger := NewRoomLogger(client, "./room-logs")
    if err := logger.Start(); err != nil {
        log.Fatal(err)
    }
    defer logger.Close()

    // Set event handler
    cfg.EventHandler = logger.HandleEvent

    // Start and run
    ctx := context.Background()
    if err := client.Start(ctx); err != nil {
        log.Fatal(err)
    }

    log.Println("Room monitor started. Logging all events...")
    select {} // Keep running
}
```

These examples demonstrate various patterns and approaches for building Matrix clients with hicli.
They cover basic setup, authentication, message handling, room management, encryption, event
processing, TUI integration, error handling, performance optimization, and complete applications.

Each pattern can be adapted and combined to build sophisticated Matrix clients tailored to specific
use cases and requirements.
