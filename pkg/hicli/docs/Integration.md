# hicli Integration Guide

This guide explains how to integrate the hicli package with TUI applications, specifically focusing
on integration with mauview and best practices for building Matrix clients.

## Table of Contents

1. [Table of Contents](#table-of-contents)
2. [Integration Overview](#integration-overview)
   1. [Architecture Diagram](#architecture-diagram)
3. [Setting Up hicli](#setting-up-hicli)
   1. [Basic Configuration](#basic-configuration)
   2. [Initialization Sequence](#initialization-sequence)
4. [Event Loop Integration](#event-loop-integration)
   1. [Event Handler Structure](#event-handler-structure)
   2. [Thread Safety](#thread-safety)
5. [UI State Management](#ui-state-management)
   1. [Room List Management](#room-list-management)
   2. [Message Timeline UI](#message-timeline-ui)
6. [Message Display](#message-display)
   1. [Rich Text Processing](#rich-text-processing)
   2. [Message Types](#message-types)
7. [Input Handling](#input-handling)
   1. [Message Input Component](#message-input-component)
   2. [Command Processing](#command-processing)
8. [Room Management](#room-management)
   1. [Room Selection](#room-selection)
   2. [Room Creation](#room-creation)
9. [Error Handling](#error-handling)
   1. [Network Error Handling](#network-error-handling)
   2. [Sync Error Recovery](#sync-error-recovery)
10. [Performance Optimization](#performance-optimization)
    1. [Lazy Loading](#lazy-loading)
    2. [Efficient UI Updates](#efficient-ui-updates)
11. [Complete Example](#complete-example)
12. [Advanced Integration Topics](#advanced-integration-topics)

## Integration Overview

hicli is designed to work seamlessly with TUI frameworks through its event-driven architecture.
The integration pattern follows these principles:

1. **Event-Driven Updates**: hicli sends events to your application via callbacks
2. **Command-Response Pattern**: Your UI sends commands to hicli and receives responses
3. **Asynchronous Operations**: Network operations happen in the background
4. **State Consistency**: hicli maintains consistent state across components

### Architecture Diagram

```plain
┌─────────────────┐    Events    ┌─────────────────┐
│   mauview UI    │◄─────────────│     hicli       │
│                 │              │                 │
│ • Room List     │   Commands   │ • Matrix Client │
│ • Message View  │─────────────►│ • Database      │
│ • Input Field   │              │ • Encryption    │
└─────────────────┘              └─────────────────┘
```

## Setting Up hicli

### Basic Configuration

```go
package main

import (
    "context"
    "log"
    "os"
    "path/filepath"

    "github.com/gomuks/gomuks/pkg/hicli"
    "github.com/rs/zerolog"
    "github.com/tulir/mauview"
)

type MatrixClient struct {
    hicli  *hicli.HiClient
    app    *mauview.Application

    // UI components will be added here
}

func NewMatrixClient() (*MatrixClient, error) {
    // Create data directories
    homeDir, _ := os.UserHomeDir()
    dataDir := filepath.Join(homeDir, ".config", "mymatrix")
    cacheDir := filepath.Join(homeDir, ".cache", "mymatrix")

    os.MkdirAll(dataDir, 0700)
    os.MkdirAll(cacheDir, 0755)

    client := &MatrixClient{
        app: mauview.NewApplication(),
    }

    // Configure hicli
    cfg := &hicli.Config{
        DataDir:      dataDir,
        CacheDir:     cacheDir,
        DeviceName:   "My Matrix TUI Client",
        PickleKey:    derivePickleKey(), // Implement secure key derivation
        EventHandler: client.handleHicliEvent,
        LogoutFunc:   client.handleLogout,
    }

    var err error
    client.hicli, err = hicli.NewHiClient(cfg)
    if err != nil {
        return nil, err
    }

    return client, nil
}

func derivePickleKey() []byte {
    // Implement secure key derivation based on user password/keyfile
    // This is a placeholder - use proper key derivation in production
    return []byte("secure-pickle-key-32-bytes-long!")
}
```

### Initialization Sequence

```go
func (mc *MatrixClient) Start() error {
    // Start hicli
    ctx := context.Background()
    if err := mc.hicli.Start(ctx); err != nil {
        return err
    }

    // Check if already logged in
    if mc.hicli.Account != nil && mc.hicli.Account.AccessToken != "" {
        // Already logged in, show main UI
        mc.showMainUI()
    } else {
        // Show login UI
        mc.showLoginUI()
    }

    // Start mauview event loop
    return mc.app.Start()
}
```

## Event Loop Integration

### Event Handler Structure

```go
func (mc *MatrixClient) handleHicliEvent(evt any) {
    // All hicli events come through this function
    // Process them and update UI accordingly

    switch e := evt.(type) {
    case *jsoncmd.SyncComplete:
        mc.handleSyncComplete(e)
    case *jsoncmd.SyncStatus:
        mc.handleSyncStatus(e)
    case *jsoncmd.EventsDecrypted:
        mc.handleEventsDecrypted(e)
    case *jsoncmd.SendComplete:
        mc.handleSendComplete(e)
    case *jsoncmd.Typing:
        mc.handleTyping(e)
    case *jsoncmd.ClientState:
        mc.handleClientState(e)
    default:
        log.Printf("Unhandled event type: %T", evt)
    }

    // Trigger UI redraw
    mc.app.RedrawSoon()
}
```

### Thread Safety

Since hicli events come from background goroutines, ensure UI updates are thread-safe:

```go
func (mc *MatrixClient) handleSyncComplete(evt *jsoncmd.SyncComplete) {
    // Process in background, then update UI
    go func() {
        // Process rooms
        for _, room := range evt.Rooms {
            mc.processRoomUpdate(room)
        }

        // Schedule UI update on main thread
        mc.app.GetScreen().PostEvent(tcell.NewEventInterrupt(nil))
    }()
}
```

## UI State Management

### Room List Management

```go
type RoomListUI struct {
    *mauview.TextView
    rooms   []*database.Room
    selected int
}

func NewRoomListUI() *RoomListUI {
    return &RoomListUI{
        TextView: mauview.NewTextView().
            SetDynamicColors(true).
            SetScrollable(true),
        rooms: make([]*database.Room, 0),
    }
}

func (rl *RoomListUI) UpdateRooms(rooms []*database.Room) {
    rl.rooms = rooms
    rl.renderRoomList()
}

func (rl *RoomListUI) renderRoomList() {
    var content strings.Builder

    for i, room := range rl.rooms {
        // Highlight selected room
        prefix := "  "
        if i == rl.selected {
            prefix = "[black:white]>[white:-] "
        }

        // Unread indicator
        unreadIndicator := ""
        if room.UnreadCount > 0 {
            unreadIndicator = fmt.Sprintf(" [red](%d)[white]", room.UnreadCount)
        }

        // Room name with optional topic
        name := room.Name
        if name == "" {
            name = room.ID.String() // Fallback to room ID
        }

        content.WriteString(fmt.Sprintf("%s%s%s\n", prefix, name, unreadIndicator))
    }

    rl.SetText(content.String())
}

func (rl *RoomListUI) OnKeyEvent(event mauview.KeyEvent) bool {
    switch event.Key() {
    case tcell.KeyUp:
        if rl.selected > 0 {
            rl.selected--
            rl.renderRoomList()
        }
        return true
    case tcell.KeyDown:
        if rl.selected < len(rl.rooms)-1 {
            rl.selected++
            rl.renderRoomList()
        }
        return true
    case tcell.KeyEnter:
        if rl.selected < len(rl.rooms) {
            // Trigger room selection
            if rl.onRoomSelected != nil {
                rl.onRoomSelected(rl.rooms[rl.selected])
            }
        }
        return true
    }
    return false
}
```

### Message Timeline UI

```go
type MessageTimelineUI struct {
    *mauview.TextView
    roomID   id.RoomID
    events   []*database.Event
    client   *hicli.HiClient
}

func NewMessageTimelineUI(client *hicli.HiClient) *MessageTimelineUI {
    return &MessageTimelineUI{
        TextView: mauview.NewTextView().
            SetDynamicColors(true).
            SetScrollable(true).
            SetWordWrap(true),
        client: client,
        events: make([]*database.Event, 0),
    }
}

func (mt *MessageTimelineUI) SetRoom(roomID id.RoomID) error {
    mt.roomID = roomID

    // Load initial messages
    room, err := mt.client.GetRoom(roomID)
    if err != nil {
        return err
    }

    // Get recent timeline events
    events, err := mt.client.DB.GetTimelineEvents(roomID, 50, room.LastMessageID)
    if err != nil {
        return err
    }

    mt.events = events
    mt.renderMessages()
    mt.ScrollToEnd()

    return nil
}

func (mt *MessageTimelineUI) AddEvent(event *database.Event) {
    if event.RoomID != mt.roomID {
        return
    }

    mt.events = append(mt.events, event)
    mt.renderMessages()
    mt.ScrollToEnd()
}

func (mt *MessageTimelineUI) renderMessages() {
    var content strings.Builder

    for _, event := range mt.events {
        mt.renderEvent(&content, event)
    }

    mt.SetText(content.String())
}

func (mt *MessageTimelineUI) renderEvent(content *strings.Builder, event *database.Event) {
    timestamp := event.Timestamp.Format("15:04")

    switch event.Type {
    case event.EventMessage:
        // Parse message content
        var msgContent struct {
            MsgType string `json:"msgtype"`
            Body    string `json:"body"`
            Format  string `json:"format,omitempty"`
            FormattedBody string `json:"formatted_body,omitempty"`
        }

        if err := json.Unmarshal(event.Content, &msgContent); err != nil {
            content.WriteString(fmt.Sprintf("[gray]%s[white] [red]<malformed message>[white]\n", timestamp))
            return
        }

        // Get sender display name
        senderName := event.Sender.String()
        if member := mt.getRoomMember(event.Sender); member != nil {
            if member.Displayname != "" {
                senderName = member.Displayname
            }
        }

        // Format message
        body := msgContent.Body
        if msgContent.Format == "org.matrix.custom.html" && msgContent.FormattedBody != "" {
            // Convert HTML to plain text for TUI display
            body = htmlToPlaintext(msgContent.FormattedBody)
        }

        content.WriteString(fmt.Sprintf("[gray]%s[white] [cyan]%s:[white] %s\n",
            timestamp, senderName, body))

    case event.EventRoomMember:
        // Membership changes
        var memberContent struct {
            Membership string `json:"membership"`
            Displayname string `json:"displayname,omitempty"`
        }

        if err := json.Unmarshal(event.Content, &memberContent); err != nil {
            return
        }

        action := memberContent.Membership
        if action == "join" && event.PrevContent != nil {
            // Check if it's a name change
            var prevContent struct {
                Membership string `json:"membership"`
                Displayname string `json:"displayname,omitempty"`
            }
            json.Unmarshal(event.PrevContent, &prevContent)

            if prevContent.Membership == "join" && prevContent.Displayname != memberContent.Displayname {
                content.WriteString(fmt.Sprintf("[gray]%s[white] [yellow]%s changed name to %s[white]\n",
                    timestamp, prevContent.Displayname, memberContent.Displayname))
                return
            }
        }

        content.WriteString(fmt.Sprintf("[gray]%s[white] [yellow]%s %sed the room[white]\n",
            timestamp, event.Sender, action))

    case event.EventRoomName:
        var nameContent struct {
            Name string `json:"name"`
        }
        json.Unmarshal(event.Content, &nameContent)
        content.WriteString(fmt.Sprintf("[gray]%s[white] [yellow]Room name changed to: %s[white]\n",
            timestamp, nameContent.Name))

    case event.EventRoomTopic:
        var topicContent struct {
            Topic string `json:"topic"`
        }
        json.Unmarshal(event.Content, &topicContent)
        content.WriteString(fmt.Sprintf("[gray]%s[white] [yellow]Room topic: %s[white]\n",
            timestamp, topicContent.Topic))
    }
}

func (mt *MessageTimelineUI) getRoomMember(userID id.UserID) *database.Event {
    // Get member state event for user
    member, _ := mt.client.DB.GetStateEvent(mt.roomID, event.StateMember, userID.String())
    return member
}
```

## Message Display

### Rich Text Processing

```go
func htmlToPlaintext(html string) string {
    // Simple HTML to plaintext conversion for TUI
    // In production, use a proper HTML parser

    // Replace common HTML tags
    replacements := map[string]string{
        "<strong>": "[yellow]",
        "</strong>": "[white]",
        "<em>": "[cyan]",
        "</em>": "[white]",
        "<code>": "[green]",
        "</code>": "[white]",
        "<br>": "\n",
        "<br/>": "\n",
        "&lt;": "<",
        "&gt;": ">",
        "&amp;": "&",
    }

    result := html
    for old, new := range replacements {
        result = strings.ReplaceAll(result, old, new)
    }

    // Remove remaining HTML tags
    re := regexp.MustCompile(`<[^>]*>`)
    result = re.ReplaceAllString(result, "")

    return result
}
```

### Message Types

```go
func (mt *MessageTimelineUI) renderMessageContent(content *strings.Builder, event *database.Event) {
    var msgContent struct {
        MsgType string `json:"msgtype"`
        Body    string `json:"body"`
        URL     string `json:"url,omitempty"`
        Info    struct {
            MimeType string `json:"mimetype,omitempty"`
            Size     int64  `json:"size,omitempty"`
        } `json:"info,omitempty"`
    }

    if err := json.Unmarshal(event.Content, &msgContent); err != nil {
        content.WriteString("[red]<malformed message>[white]")
        return
    }

    switch msgContent.MsgType {
    case "m.text":
        content.WriteString(msgContent.Body)

    case "m.emote":
        senderName := mt.getSenderDisplayName(event.Sender)
        content.WriteString(fmt.Sprintf("[purple]* %s %s[white]", senderName, msgContent.Body))

    case "m.image":
        content.WriteString(fmt.Sprintf("[blue]🖼️  Image: %s[white]", msgContent.Body))
        if msgContent.Info.Size > 0 {
            content.WriteString(fmt.Sprintf(" (%s)", formatBytes(msgContent.Info.Size)))
        }

    case "m.file":
        content.WriteString(fmt.Sprintf("[green]📎 File: %s[white]", msgContent.Body))
        if msgContent.Info.Size > 0 {
            content.WriteString(fmt.Sprintf(" (%s)", formatBytes(msgContent.Info.Size)))
        }

    case "m.audio":
        content.WriteString(fmt.Sprintf("[yellow]🎵 Audio: %s[white]", msgContent.Body))

    case "m.video":
        content.WriteString(fmt.Sprintf("[magenta]🎬 Video: %s[white]", msgContent.Body))

    default:
        content.WriteString(fmt.Sprintf("[gray]Unsupported message type: %s[white]", msgContent.MsgType))
    }
}

func formatBytes(bytes int64) string {
    const unit = 1024
    if bytes < unit {
        return fmt.Sprintf("%d B", bytes)
    }
    div, exp := int64(unit), 0
    for n := bytes / unit; n >= unit; n /= unit {
        div *= unit
        exp++
    }
    return fmt.Sprintf("%.1f %cB", float64(bytes)/float64(div), "KMGTPE"[exp])
}
```

## Input Handling

### Message Input Component

```go
type MessageInputUI struct {
    *mauview.InputArea
    client      *hicli.HiClient
    currentRoom id.RoomID
    sendFunc    func(string)
}

func NewMessageInputUI(client *hicli.HiClient) *MessageInputUI {
    input := &MessageInputUI{
        InputArea: mauview.NewInputArea().
            SetPlaceholder("Type a message...").
            SetWordWrap(true),
        client: client,
    }

    input.SetSubmitFunc(input.handleSend)
    input.SetChangedFunc(input.handleTyping)

    return input
}

func (mi *MessageInputUI) SetRoom(roomID id.RoomID) {
    mi.currentRoom = roomID
}

func (mi *MessageInputUI) handleSend(text string) {
    if text == "" || mi.currentRoom == "" {
        return
    }

    // Clear input immediately for responsive feel
    mi.SetText("")

    // Send message asynchronously
    go func() {
        req := &hicli.SendMessageRequest{
            RoomID:      mi.currentRoom,
            Content:     text,
            ContentType: "text/markdown", // Enable markdown processing
        }

        ctx := context.Background()
        _, err := mi.client.SendMessage(ctx, req)
        if err != nil {
            log.Printf("Failed to send message: %v", err)
            // Could show error in UI
        }
    }()
}

func (mi *MessageInputUI) handleTyping(text string) {
    if mi.currentRoom == "" {
        return
    }

    // Send typing notification
    go func() {
        req := &hicli.SetTypingRequest{
            RoomID:  mi.currentRoom,
            Typing:  len(text) > 0,
            Timeout: 5000, // 5 seconds
        }

        ctx := context.Background()
        mi.client.SetTyping(ctx, req)
    }()
}
```

### Command Processing

```go
func (mi *MessageInputUI) processCommand(text string) bool {
    if !strings.HasPrefix(text, "/") {
        return false // Not a command
    }

    parts := strings.SplitN(text[1:], " ", 2)
    command := parts[0]
    args := ""
    if len(parts) > 1 {
        args = parts[1]
    }

    switch command {
    case "join":
        mi.handleJoinCommand(args)
    case "leave":
        mi.handleLeaveCommand()
    case "invite":
        mi.handleInviteCommand(args)
    case "kick":
        mi.handleKickCommand(args)
    case "ban":
        mi.handleBanCommand(args)
    case "me":
        mi.handleEmoteCommand(args)
    default:
        log.Printf("Unknown command: %s", command)
        return false
    }

    return true
}

func (mi *MessageInputUI) handleJoinCommand(roomID string) {
    if roomID == "" {
        return
    }

    go func() {
        req := &hicli.JoinRoomRequest{
            RoomID: roomID,
        }

        ctx := context.Background()
        _, err := mi.client.JoinRoom(ctx, req)
        if err != nil {
            log.Printf("Failed to join room: %v", err)
        }
    }()
}

func (mi *MessageInputUI) handleEmoteCommand(action string) {
    if action == "" || mi.currentRoom == "" {
        return
    }

    go func() {
        req := &hicli.SendEventRequest{
            RoomID:    mi.currentRoom,
            EventType: event.EventMessage,
            Content: map[string]any{
                "msgtype": "m.emote",
                "body":    action,
            },
        }

        ctx := context.Background()
        _, err := mi.client.SendEvent(ctx, req)
        if err != nil {
            log.Printf("Failed to send emote: %v", err)
        }
    }()
}
```

## Room Management

### Room Selection

```go
func (mc *MatrixClient) handleRoomSelection(room *database.Room) {
    // Update current room
    mc.currentRoom = room.ID

    // Update message timeline
    if err := mc.messageTimeline.SetRoom(room.ID); err != nil {
        log.Printf("Failed to load room timeline: %v", err)
        return
    }

    // Update input field
    mc.messageInput.SetRoom(room.ID)

    // Mark room as read
    go func() {
        if room.LastMessage != nil {
            req := &hicli.MarkReadRequest{
                RoomID:  room.ID,
                EventID: room.LastMessage.ID,
            }

            ctx := context.Background()
            mc.hicli.MarkRead(ctx, req)
        }
    }()

    // Update window title
    windowTitle := fmt.Sprintf("Matrix - %s", room.Name)
    mc.headerUI.SetText(windowTitle)
}
```

### Room Creation

```go
func (mc *MatrixClient) createRoom(name, topic string, private bool, invites []string) {
    visibility := "public"
    preset := "public_chat"
    if private {
        visibility = "private"
        preset = "private_chat"
    }

    inviteList := make([]id.UserID, len(invites))
    for i, invite := range invites {
        inviteList[i] = id.UserID(invite)
    }

    go func() {
        req := &hicli.CreateRoomRequest{
            Name:       name,
            Topic:      topic,
            Visibility: visibility,
            Preset:     preset,
            Invite:     inviteList,
        }

        ctx := context.Background()
        resp, err := mc.hicli.CreateRoom(ctx, req)
        if err != nil {
            log.Printf("Failed to create room: %v", err)
            return
        }

        log.Printf("Created room: %s", resp.RoomID)
    }()
}
```

## Error Handling

### Network Error Handling

```go
func (mc *MatrixClient) handleNetworkError(err error) {
    if httpErr, ok := err.(*mautrix.HTTPError); ok {
        switch httpErr.ResponseCode {
        case 401:
            // Unauthorized - token expired
            mc.showLoginUI()
            mc.showError("Session expired. Please login again.")

        case 403:
            // Forbidden
            mc.showError("Access denied. Check your permissions.")

        case 429:
            // Rate limited
            mc.showError("Rate limited. Please slow down.")

        case 500, 502, 503:
            // Server errors
            mc.showError("Server error. Please try again later.")

        default:
            mc.showError(fmt.Sprintf("Network error: %s", httpErr.Message))
        }
    } else {
        // Network connectivity issues
        mc.showError(fmt.Sprintf("Connection error: %v", err))
    }
}

func (mc *MatrixClient) showError(message string) {
    // Show error in status bar or popup
    mc.statusBar.SetText(fmt.Sprintf("[red]Error: %s[white]", message))

    // Auto-clear after 5 seconds
    go func() {
        time.Sleep(5 * time.Second)
        mc.statusBar.SetText("Ready")
    }()
}
```

### Sync Error Recovery

```go
func (mc *MatrixClient) handleSyncStatus(evt *jsoncmd.SyncStatus) {
    switch evt.State {
    case "running":
        mc.statusBar.SetText("[green]Connected[white]")

    case "stopped":
        mc.statusBar.SetText("[yellow]Disconnected[white]")

    case "error":
        mc.statusBar.SetText(fmt.Sprintf("[red]Sync error: %s[white]", evt.Error))

        // Attempt reconnection after delay
        go func() {
            time.Sleep(5 * time.Second)
            ctx := context.Background()
            mc.hicli.StartSync(ctx)
        }()
    }
}
```

## Performance Optimization

### Lazy Loading

```go
func (mc *MatrixClient) loadOlderMessages() {
    if mc.currentRoom == "" {
        return
    }

    // Get current oldest message
    if len(mc.messageTimeline.events) == 0 {
        return
    }

    oldestEvent := mc.messageTimeline.events[0]

    go func() {
        req := &hicli.PaginateRequest{
            RoomID:        mc.currentRoom,
            MaxTimelineID: oldestEvent.TimelineRowID,
            Limit:         50,
            Reset:         false,
        }

        ctx := context.Background()
        resp, err := mc.hicli.Paginate(ctx, req)
        if err != nil {
            log.Printf("Failed to load older messages: %v", err)
            return
        }

        // Prepend events to timeline
        mc.messageTimeline.events = append(resp.Events, mc.messageTimeline.events...)
        mc.messageTimeline.renderMessages()
    }()
}
```

### Efficient UI Updates

```go
func (mc *MatrixClient) handleEventsDecrypted(evt *jsoncmd.EventsDecrypted) {
    // Group events by room for efficient updates
    eventsByRoom := make(map[id.RoomID][]*database.Event)

    for _, event := range evt.Events {
        roomID := event.Event.RoomID
        eventsByRoom[roomID] = append(eventsByRoom[roomID], event.Event)
    }

    // Update only affected rooms
    for roomID, events := range eventsByRoom {
        if roomID == mc.currentRoom {
            // Update current room timeline
            for _, event := range events {
                mc.messageTimeline.AddEvent(event)
            }
        }

        // Update room in room list (for unread counts, last message, etc.)
        mc.roomList.UpdateRoomEvents(roomID, events)
    }
}
```

## Complete Example

Here's a complete minimal Matrix TUI client:

```go
package main

import (
    "context"
    "log"
    "os"
    "path/filepath"

    "github.com/gdamore/tcell/v2"
    "github.com/gomuks/gomuks/pkg/hicli"
    "github.com/tulir/mauview"
    "maunium.net/go/mautrix/id"
)

type MatrixTUI struct {
    app           *mauview.Application
    hicli         *hicli.HiClient

    // UI components
    roomList      *RoomListUI
    messageArea   *MessageTimelineUI
    inputField    *MessageInputUI
    statusBar     *mauview.TextField

    // State
    currentRoom   id.RoomID
}

func main() {
    tui, err := NewMatrixTUI()
    if err != nil {
        log.Fatal(err)
    }

    if err := tui.Start(); err != nil {
        log.Fatal(err)
    }
}

func NewMatrixTUI() (*MatrixTUI, error) {
    tui := &MatrixTUI{
        app: mauview.NewApplication(),
    }

    // Setup hicli
    homeDir, _ := os.UserHomeDir()
    cfg := &hicli.Config{
        DataDir:      filepath.Join(homeDir, ".config", "matrix-tui"),
        CacheDir:     filepath.Join(homeDir, ".cache", "matrix-tui"),
        DeviceName:   "Matrix TUI Client",
        PickleKey:    []byte("secure-pickle-key-32-bytes-long!"),
        EventHandler: tui.handleHicliEvent,
    }

    var err error
    tui.hicli, err = hicli.NewHiClient(cfg)
    if err != nil {
        return nil, err
    }

    // Setup UI
    tui.setupUI()

    return tui, nil
}

func (tui *MatrixTUI) setupUI() {
    // Create components
    tui.roomList = NewRoomListUI()
    tui.messageArea = NewMessageTimelineUI(tui.hicli)
    tui.inputField = NewMessageInputUI(tui.hicli)
    tui.statusBar = mauview.NewTextField().SetText("Ready")

    // Room selection callback
    tui.roomList.onRoomSelected = func(room *database.Room) {
        tui.selectRoom(room)
    }

    // Layout
    leftPanel := mauview.NewBox(tui.roomList).
        SetBorder(true).
        SetTitle("Rooms")

    rightPanel := mauview.NewFlex().
        SetDirection(mauview.FlexColumn).
        AddProportionalComponent(
            mauview.NewBox(tui.messageArea).
                SetBorder(true).
                SetTitle("Messages"),
            1,
        ).
        AddFixedComponent(
            mauview.NewBox(tui.inputField).
                SetBorder(true).
                SetTitle("Compose"),
            5,
        )

    mainArea := mauview.NewFlex().
        SetDirection(mauview.FlexRow).
        AddFixedComponent(leftPanel, 30).
        AddProportionalComponent(rightPanel, 1)

    root := mauview.NewFlex().
        SetDirection(mauview.FlexColumn).
        AddProportionalComponent(mainArea, 1).
        AddFixedComponent(
            mauview.NewBox(tui.statusBar).SetTitle("Status"),
            1,
        )

    tui.app.SetRoot(root)
}

func (tui *MatrixTUI) Start() error {
    // Start hicli
    ctx := context.Background()
    if err := tui.hicli.Start(ctx); err != nil {
        return err
    }

    // Login if needed
    if tui.hicli.Account == nil || tui.hicli.Account.AccessToken == "" {
        // In a real client, show login UI
        // For this example, we'll just log an error
        log.Println("Not logged in. Please implement login UI.")
    }

    // Start mauview
    return tui.app.Start()
}

func (tui *MatrixTUI) handleHicliEvent(evt any) {
    switch e := evt.(type) {
    case *jsoncmd.SyncComplete:
        tui.roomList.UpdateRooms(e.Rooms)

    case *jsoncmd.EventsDecrypted:
        for _, event := range e.Events {
            if event.Event.RoomID == tui.currentRoom {
                tui.messageArea.AddEvent(event.Event)
            }
        }

    case *jsoncmd.SyncStatus:
        if e.Connected {
            tui.statusBar.SetText("[green]Connected[white]")
        } else {
            tui.statusBar.SetText("[red]Disconnected[white]")
        }
    }

    tui.app.RedrawSoon()
}

func (tui *MatrixTUI) selectRoom(room *database.Room) {
    tui.currentRoom = room.ID
    tui.messageArea.SetRoom(room.ID)
    tui.inputField.SetRoom(room.ID)

    // Mark as read
    go func() {
        if room.LastMessage != nil {
            req := &hicli.MarkReadRequest{
                RoomID:  room.ID,
                EventID: room.LastMessage.ID,
            }
            ctx := context.Background()
            tui.hicli.MarkRead(ctx, req)
        }
    }()
}
```

This integration guide provides a foundation for building sophisticated Matrix TUI clients using
hicli and mauview. The event-driven architecture ensures responsive UIs while the command-based
interface provides clean separation between UI and Matrix protocol handling.

## Advanced Integration Topics

For more advanced integration scenarios, consider:

1. **Multi-account support**: Managing multiple hicli instances
2. **Plugin architecture**: Extending functionality with plugins
3. **Custom event types**: Handling application-specific events
4. **Performance profiling**: Monitoring memory and CPU usage
5. **Accessibility**: Supporting screen readers and other assistive technologies

These topics can be explored based on specific application requirements and user needs.
