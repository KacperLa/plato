# Signal Messaging Integration for Cadmus E-Reader

**Version:** 1.0  
**Date:** 2026-02-16  
**Status:** Design Phase

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Component Details](#component-details)
4. [Implementation Phases](#implementation-phases)
5. [File Structure](#file-structure)
6. [API Specifications](#api-specifications)
7. [Database Schema](#database-schema)
8. [Configuration](#configuration)
9. [Security Considerations](#security-considerations)
10. [Testing Strategy](#testing-strategy)
11. [Deployment Guide](#deployment-guide)
12. [Future Enhancements](#future-enhancements)

---

## Executive Summary

This document outlines the design and implementation plan for integrating Signal messaging capabilities into the Cadmus e-reader application. The solution enables users to send and receive Signal messages (DMs and group chats) through a custom UI optimized for e-ink displays.

### Goals

- **Primary**: Enable send/receive of Signal messages on Cadmus device
- **MVP Features**: Direct messages and group chat support
- **Architecture**: Client-server model with local message caching
- **Network**: LAN-based initially, with path to internet access

### Key Design Decisions

1. **Two-component architecture**: Bridge server + Cadmus client
2. **Message persistence**: SQLite database on host for offline support
3. **Real-time updates**: Server-Sent Events (SSE) for live notifications
4. **Based on nanobot pattern**: Proven signal-cli HTTP JSON-RPC integration
5. **Built-in view**: Integrated into Cadmus as a native application view

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     System Architecture                          │
│                                                                  │
│  ┌──────────────────┐         LAN/Internet      ┌─────────────┐│
│  │   Cadmus Device  │ ◄────────────────────────►│ Host Server ││
│  │   (E-Reader)     │    HTTP REST + SSE        │             ││
│  │                  │                            │             ││
│  │ ┌──────────────┐ │                            │┌───────────┐││
│  │ │ Signal View  │ │                            ││signal-cli │││
│  │ │              │ │                            ││  daemon   │││
│  │ │ - Convos     │ │                            │└─────┬─────┘││
│  │ │ - Messages   │ │                            │      │      ││
│  │ │ - Compose    │ │                            │      │      ││
│  │ └──────┬───────┘ │                            │┌─────▼─────┐││
│  │        │         │                            ││  Signal   │││
│  │ ┌──────▼───────┐ │                            ││  Bridge   │││
│  │ │HTTP Client   │ │                            ││  Server   │││
│  │ │ - REST calls │ │                            ││           │││
│  │ │ - SSE stream │ │                            ││- REST API │││
│  │ └──────────────┘ │                            ││- SSE push │││
│  │                  │                            ││- Msg DB   │││
│  │ ┌──────────────┐ │                            │└───────────┘││
│  │ │ Local Cache  │ │                            │             ││
│  │ │ - Convos     │ │                            │┌───────────┐││
│  │ │ - Messages   │ │                            ││  SQLite   │││
│  │ └──────────────┘ │                            ││ Database  │││
│  └──────────────────┘                            │└───────────┘││
│                                                   └─────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Receiving Messages:**
1. Signal message arrives at signal-cli daemon
2. Bridge server receives via SSE (`/api/v1/events`)
3. Bridge stores message in SQLite database
4. Bridge broadcasts event to connected Cadmus clients via SSE
5. Cadmus receives event, updates UI, triggers e-ink refresh

**Sending Messages:**
1. User types message in Cadmus compose bar
2. Cadmus sends HTTP POST to bridge `/api/conversations/:id/messages`
3. Bridge forwards to signal-cli via JSON-RPC
4. signal-cli sends via Signal network
5. Bridge stores sent message in database
6. Success/failure response returned to Cadmus

---

## Component Details

### 1. Signal Bridge Server (Host-Side)

**Location:** `crates/signal-bridge/`  
**Language:** Rust  
**Type:** Standalone binary server

#### Responsibilities

- Interface with signal-cli daemon (JSON-RPC over HTTP)
- Persist all messages in SQLite database
- Provide REST API for Cadmus clients
- Stream real-time updates via Server-Sent Events
- Maintain conversation metadata (names, unread counts)
- Handle contact name resolution

#### Technology Stack

```toml
[dependencies]
axum = "0.7"                    # Web framework
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio"] }
reqwest = { version = "0.13", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.5", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1"
chrono = "0.4"
```

#### Key Modules

- **`main.rs`**: Server initialization, graceful shutdown
- **`api.rs`**: REST endpoint handlers
- **`signal_client.rs`**: signal-cli JSON-RPC client
- **`db.rs`**: Database operations (CRUD for messages/conversations)
- **`sse.rs`**: Server-Sent Events broadcast system
- **`models.rs`**: Shared data structures

### 2. Cadmus Signal Client (Device-Side)

**Location:** `crates/core/src/signal/`  
**Language:** Rust  
**Type:** Library module

#### Responsibilities

- HTTP client for bridge API
- Parse JSON responses
- SSE connection management
- Local caching of conversations/messages
- Error handling and retry logic

#### Key Structures

```rust
pub struct SignalClient {
    http: reqwest::blocking::Client,
    base_url: String,
}

pub struct Conversation {
    pub id: String,
    pub name: String,
    pub is_group: bool,
    pub last_message: String,
    pub last_message_time: i64,
    pub unread_count: u32,
}

pub struct Message {
    pub id: i64,
    pub sender_name: String,
    pub content: String,
    pub timestamp: i64,
    pub is_outgoing: bool,
    pub is_read: bool,
}
```

### 3. Signal UI Views (Device-Side)

**Location:** `crates/core/src/view/signal/`  
**Language:** Rust  
**Type:** View implementations

#### View Hierarchy

```
SignalApp (main container)
├── ConversationList (scrollable list)
│   ├── TopBar (title + refresh button)
│   └── ConversationRow (repeating)
│       ├── Name label
│       ├── Last message preview
│       ├── Timestamp
│       └── Unread badge
│
└── MessageView (conversation detail)
    ├── TopBar (back + conversation name)
    ├── MessageList (scrollable)
    │   └── MessageBubble (repeating)
    │       ├── Sender name (groups only)
    │       ├── Content
    │       └── Timestamp
    └── ComposeBar
        ├── InputField
        └── SendButton
```

#### E-ink Optimization

- **UpdateMode::Partial**: Message append, typing
- **UpdateMode::Gui**: Full screen refresh, conversation switch
- **UpdateMode::Fast**: Unread badge updates
- Virtual scrolling for long message lists
- Batch updates to minimize refreshes

---

## Implementation Phases

### Phase 1: Bridge Server Foundation (Days 1-3)

**Goal:** Working bridge server that connects to signal-cli and persists messages

#### Tasks
1. Project setup and dependencies
2. Database schema and migrations
3. signal-cli JSON-RPC client implementation
4. Message storage logic
5. Basic REST API (conversations, messages)
6. SSE broadcast system
7. Background sync loop

**Deliverable:** Bridge server that syncs messages to database

### Phase 2: Cadmus Client Implementation (Days 4-7)

**Goal:** Functional Signal UI in Cadmus

#### Tasks
1. HTTP client module
2. Settings integration
3. Conversation list view
4. Message view with pagination
5. Compose bar and send functionality
6. Event handling system
7. Local caching

**Deliverable:** Working Signal app in Cadmus

### Phase 3: Integration & Polish (Days 8-10)

**Goal:** Production-ready MVP

#### Tasks
1. Real-time updates via SSE
2. Error handling and retry logic
3. E-ink refresh optimization
4. Loading states and spinners
5. Settings UI
6. Menu integration
7. Testing and bug fixes

**Deliverable:** Polished, stable Signal messaging

### Phase 4: Security & Internet (Future)

**Goal:** Secure remote access

#### Tasks
1. TLS/HTTPS support
2. API key authentication
3. Rate limiting
4. VPN/reverse proxy setup guide

**Deliverable:** Internet-accessible secure bridge

---

## File Structure

```
plato/
├── crates/
│   ├── signal-bridge/                 # NEW: Host server
│   │   ├── Cargo.toml
│   │   ├── README.md
│   │   ├── config.example.toml
│   │   ├── src/
│   │   │   ├── main.rs               # Server entry point
│   │   │   ├── config.rs             # Configuration loading
│   │   │   ├── api.rs                # REST endpoints
│   │   │   ├── db.rs                 # Database operations
│   │   │   ├── signal_client.rs      # signal-cli integration
│   │   │   ├── sse.rs                # Server-Sent Events
│   │   │   ├── models.rs             # Data structures
│   │   │   └── error.rs              # Error types
│   │   ├── migrations/
│   │   │   └── 001_init.sql          # Database schema
│   │   └── systemd/
│   │       └── signal-bridge.service # Systemd unit file
│   │
│   └── core/
│       ├── src/
│       │   ├── signal/               # NEW: Signal module
│       │   │   ├── mod.rs            # Module exports
│       │   │   ├── client.rs         # HTTP client
│       │   │   ├── models.rs         # Client-side models
│       │   │   └── cache.rs          # Local caching
│       │   │
│       │   ├── view/
│       │   │   ├── mod.rs            # MODIFIED: Export signal views
│       │   │   └── signal/           # NEW: Signal UI
│       │   │       ├── mod.rs        # View exports
│       │   │       ├── app.rs        # Main Signal container
│       │   │       ├── conversations.rs  # Conversation list
│       │   │       ├── messages.rs   # Message view
│       │   │       ├── compose.rs    # Input bar
│       │   │       └── common.rs     # Shared UI components
│       │   │
│       │   └── settings/
│       │       └── mod.rs            # MODIFIED: Add SignalSettings
│       │
│       └── Cargo.toml                # MODIFIED: Add signal deps
│
├── Cargo.toml                        # MODIFIED: Add signal-bridge member
├── contrib/
│   └── Settings-sample.toml          # MODIFIED: Add signal section
│
└── doc/
    ├── SIGNAL-SETUP.md               # NEW: Setup guide
    └── SIGNAL-API.md                 # NEW: API documentation
```

---

## API Specifications

### Bridge Server REST API

**Base URL:** `http://{host}:{port}/api`

#### 1. List Conversations

```http
GET /api/conversations
```

**Response:**
```json
{
  "conversations": [
    {
      "id": "+1234567890",
      "name": "John Doe",
      "is_group": false,
      "last_message": "Hello there!",
      "last_message_time": 1707849600,
      "unread_count": 3
    },
    {
      "id": "base64groupid==",
      "name": "Team Chat",
      "is_group": true,
      "last_message": "Meeting at 3pm",
      "last_message_time": 1707849500,
      "unread_count": 0
    }
  ]
}
```

#### 2. Get Messages

```http
GET /api/conversations/{id}/messages?limit=50&before=1707849600
```

**Query Parameters:**
- `limit` (optional): Number of messages to return (default: 50)
- `before` (optional): Timestamp, returns messages before this time

**Response:**
```json
{
  "messages": [
    {
      "id": 42,
      "sender_id": "+1234567890",
      "sender_name": "John Doe",
      "content": "Hello there!",
      "timestamp": 1707849600,
      "is_outgoing": false,
      "is_read": true
    }
  ],
  "has_more": true
}
```

#### 3. Send Message

```http
POST /api/conversations/{id}/messages
Content-Type: application/json

{
  "content": "Hello back!"
}
```

**Response:**
```json
{
  "id": 43,
  "timestamp": 1707849700,
  "status": "sent"
}
```

#### 4. Mark as Read

```http
POST /api/conversations/{id}/read
```

**Response:**
```json
{
  "status": "ok",
  "unread_count": 0
}
```

#### 5. Trigger Sync

```http
POST /api/sync
```

**Response:**
```json
{
  "status": "ok",
  "messages_synced": 15
}
```

#### 6. Server-Sent Events Stream

```http
GET /api/events
Accept: text/event-stream
```

**Events:**

```
event: new_message
data: {"conversation_id": "+1234567890", "message": {...}}

event: message_read
data: {"conversation_id": "+1234567890"}

event: conversation_updated
data: {"conversation_id": "+1234567890", "unread_count": 0}

event: sync_complete
data: {"messages_synced": 5}
```

---

## Database Schema

### Conversations Table

```sql
CREATE TABLE conversations (
    id TEXT PRIMARY KEY,              -- Phone number or group ID
    name TEXT,                         -- Contact/group name
    is_group BOOLEAN NOT NULL,
    last_message TEXT,
    last_message_time INTEGER,        -- Unix timestamp
    unread_count INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

CREATE INDEX idx_conversations_updated 
ON conversations(updated_at DESC);
```

### Messages Table

```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id TEXT NOT NULL,
    sender_id TEXT NOT NULL,          -- Phone number or UUID
    sender_name TEXT,
    content TEXT NOT NULL,
    timestamp INTEGER NOT NULL,       -- Signal message timestamp
    is_outgoing BOOLEAN NOT NULL,     -- Sent by us vs received
    is_read BOOLEAN DEFAULT 0,
    created_at INTEGER NOT NULL,
    FOREIGN KEY (conversation_id) 
        REFERENCES conversations(id) 
        ON DELETE CASCADE
);

CREATE INDEX idx_messages_conversation 
ON messages(conversation_id, timestamp DESC);

CREATE INDEX idx_messages_unread 
ON messages(conversation_id, is_read, is_outgoing) 
WHERE is_read = 0 AND is_outgoing = 0;

CREATE INDEX idx_messages_timestamp 
ON messages(timestamp DESC);
```

### Migration Files

**`migrations/001_init.sql`:**
```sql
-- Create conversations table
CREATE TABLE IF NOT EXISTS conversations (
    id TEXT PRIMARY KEY,
    name TEXT,
    is_group BOOLEAN NOT NULL,
    last_message TEXT,
    last_message_time INTEGER,
    unread_count INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- Create messages table
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id TEXT NOT NULL,
    sender_id TEXT NOT NULL,
    sender_name TEXT,
    content TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    is_outgoing BOOLEAN NOT NULL,
    is_read BOOLEAN DEFAULT 0,
    created_at INTEGER NOT NULL,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);

-- Create indexes
CREATE INDEX IF NOT EXISTS idx_conversations_updated ON conversations(updated_at DESC);
CREATE INDEX IF NOT EXISTS idx_messages_conversation ON messages(conversation_id, timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_messages_unread ON messages(conversation_id, is_read, is_outgoing) WHERE is_read = 0 AND is_outgoing = 0;
CREATE INDEX IF NOT EXISTS idx_messages_timestamp ON messages(timestamp DESC);
```

---

## Configuration

### Bridge Server Configuration

**`config.toml`:**

```toml
[signal_cli]
# Your Signal phone number
account = "+1234567890"

# signal-cli daemon connection
daemon_host = "localhost"
daemon_port = 8080

[server]
# Bridge server binding
host = "0.0.0.0"
port = 3030

# CORS (set to Cadmus device IPs for security)
cors_origins = ["*"]

[database]
# SQLite database path
path = "./signal_bridge.db"

# Message retention (days, 0 = keep forever)
retention_days = 0

[logging]
level = "info"  # trace, debug, info, warn, error
```

### Cadmus Settings

**`Settings.toml` (add section):**

```toml
[signal]
# Enable/disable Signal integration
enabled = true

# Bridge server URL
bridgeUrl = "http://192.168.1.100:3030"

# Auto-refresh interval (seconds, 0 = manual only)
autoRefreshInterval = 60

# Messages to fetch per page
messagesPerPage = 50

# Connection timeout (seconds)
timeout = 30
```

**Rust Settings Structure:**

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default, rename_all = "camelCase")]
pub struct SignalSettings {
    pub enabled: bool,
    pub bridge_url: String,
    pub auto_refresh_interval: u64,
    pub messages_per_page: usize,
    pub timeout: u64,
}

impl Default for SignalSettings {
    fn default() -> Self {
        Self {
            enabled: false,
            bridge_url: "http://localhost:3030".to_string(),
            auto_refresh_interval: 60,
            messages_per_page: 50,
            timeout: 30,
        }
    }
}
```

---

## Security Considerations

### Current (LAN-Only)

- No authentication required on local network
- Trust-based security model
- HTTP (non-encrypted transport)

### Future (Internet Access)

#### 1. Transport Security

- **TLS/HTTPS**: Use rustls for encrypted communication
- **Certificate**: Let's Encrypt or self-signed
- **Configuration**: Automatic cert renewal

#### 2. Authentication

- **API Keys**: Generate unique key per Cadmus device
- **Header**: `Authorization: Bearer {api_key}`
- **Storage**: Securely store in Cadmus settings

#### 3. Authorization

- Rate limiting per API key
- IP whitelisting (optional)
- Request logging for audit

#### 4. Network Security

**Option A: VPN (Recommended)**
- Use Tailscale or WireGuard
- Private network overlay
- No port forwarding needed

**Option B: Reverse Proxy**
- nginx with Let's Encrypt
- Rate limiting via nginx
- DDoS protection

#### 5. Data Security

- Encrypt database at rest (SQLCipher)
- Secure key storage
- Regular backups

---

## Testing Strategy

### Unit Tests

**Bridge Server:**
- Database operations (CRUD)
- signal-cli client mocking
- API endpoint handlers
- SSE broadcast logic

**Cadmus Client:**
- HTTP client error handling
- JSON parsing
- Cache management

### Integration Tests

- End-to-end message flow
- SSE real-time updates
- Database migrations
- Multi-client scenarios

### Manual Testing

**Test Scenarios:**
1. Send DM from Cadmus → appears in Signal mobile
2. Receive DM from Signal mobile → appears in Cadmus
3. Group message with mention → notification in Cadmus
4. Offline → queue messages → sync when online
5. Bridge restart → Cadmus reconnects automatically
6. Large conversation (1000+ messages) → pagination works

**E-ink Testing:**
- Refresh quality at different UpdateModes
- Scrolling performance
- Battery impact
- Network reliability

---

## Deployment Guide

### Prerequisites

**Host Machine:**
- Linux system (Raspberry Pi, VPS, or desktop)
- signal-cli installed and registered
- Rust toolchain (for building bridge)

**Cadmus Device:**
- Network connectivity (Wi-Fi)
- Updated to latest Cadmus version

### Setup Steps

#### 1. Install signal-cli

```bash
# Download latest release
wget https://github.com/AsamK/signal-cli/releases/download/v0.12.8/signal-cli-0.12.8-Linux.tar.gz
tar xf signal-cli-0.12.8-Linux.tar.gz -C /opt
ln -sf /opt/signal-cli-0.12.8/bin/signal-cli /usr/local/bin/

# Register or link your Signal account
signal-cli -a +1234567890 register
signal-cli -a +1234567890 verify CODE
```

#### 2. Build Bridge Server

```bash
cd plato/crates/signal-bridge
cargo build --release
```

#### 3. Configure Bridge

```bash
cp config.example.toml config.toml
# Edit config.toml with your settings
```

#### 4. Start signal-cli Daemon

```bash
signal-cli -a +1234567890 daemon --http localhost:8080
```

#### 5. Start Bridge Server

```bash
./target/release/signal-bridge
```

#### 6. Configure Cadmus

Edit `Settings.toml` on Cadmus device:

```toml
[signal]
enabled = true
bridgeUrl = "http://192.168.1.100:3030"
```

#### 7. Launch Cadmus

Open Cadmus, navigate to Signal from main menu.

### Systemd Service (Optional)

**`/etc/systemd/system/signal-cli.service`:**

```ini
[Unit]
Description=signal-cli daemon
After=network.target

[Service]
Type=simple
User=signal
ExecStart=/usr/local/bin/signal-cli -a +1234567890 daemon --http localhost:8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**`/etc/systemd/system/signal-bridge.service`:**

```ini
[Unit]
Description=Cadmus Signal Bridge
After=network.target signal-cli.service
Requires=signal-cli.service

[Service]
Type=simple
User=signal
WorkingDirectory=/opt/signal-bridge
ExecStart=/opt/signal-bridge/signal-bridge
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable signal-cli signal-bridge
sudo systemctl start signal-cli signal-bridge
```

---

## Future Enhancements

### Phase 5: Rich Media Support

- Image attachments (view inline)
- File attachments (download)
- Audio messages (playback)
- Voice message recording (if device has mic)

### Phase 6: Advanced Features

- **Contacts**: Import Signal contacts, show names instead of numbers
- **Groups**: Create/manage groups, edit group info
- **Search**: Full-text search across all messages
- **Notifications**: Toast notifications for new messages (use existing Cadmus notification system)
- **Read Receipts**: Send read receipts when viewing messages
- **Typing Indicators**: Show when other person is typing
- **Message Reactions**: Emoji reactions (👍, ❤️, etc.)
- **Link Previews**: Extract and display link metadata

### Phase 7: Performance & UX

- **Infinite Scroll**: Seamless pagination with virtual scrolling
- **Message Threading**: Reply to specific messages
- **Drafts**: Save unsent messages
- **Offline Queue**: Queue messages when offline, auto-send when reconnected
- **Sync Status**: Show sync progress indicator
- **Multiple Devices**: Support multiple Cadmus devices with same bridge

### Phase 8: Advanced Security

- **End-to-End Encryption**: Encrypt local cache
- **Biometric Lock**: PIN/fingerprint to access Signal app
- **Self-Destruct**: Disappearing messages support
- **Screenshot Protection**: Disable screenshots in Signal app

---

## Appendix

### A. signal-cli JSON-RPC Reference

**Send Message:**
```json
POST /api/v1/send
{
  "recipient": ["+1234567890"],
  "message": "Hello!"
}
```

**Send Group Message:**
```json
POST /api/v1/send
{
  "groupId": "base64groupid==",
  "message": "Hello group!"
}
```

**Receive Events (SSE):**
```http
GET /api/v1/events
Accept: text/event-stream

data: {"envelope": {"source": "+1234567890", "dataMessage": {...}}}
```

### B. Cadmus View System Reference

**View Trait:**
```rust
pub trait View {
    fn handle_event(&mut self, evt: &Event, hub: &Hub, bus: &mut Bus, 
                    rq: &mut RenderQueue, context: &mut Context) -> bool;
    fn render(&self, fb: &mut dyn Framebuffer, rect: Rectangle, fonts: &mut Fonts);
    fn rect(&self) -> &Rectangle;
    fn children(&self) -> &Vec<Box<dyn View>>;
    fn id(&self) -> Id;
}
```

**Event System:**
- Hub: Broadcast channel (mpsc::Sender)
- Bus: Parent-child queue (VecDeque)
- Events flow top-down, bubble up if unhandled

### C. E-ink UpdateMode Guide

| Mode | Use Case | Speed | Quality |
|------|----------|-------|---------|
| `Gui` | Full screen refresh | Slow | Best (16 levels) |
| `Partial` | Message append | Medium | Good |
| `Fast` | Unread badge | Fast | 1-bit |
| `FastMono` | Animations | Fastest | 1-bit |
| `Full` | Remove ghosting | Slowest | Best |

### D. nanobot Signal Integration Reference

**Key Patterns from nanobot:**
- SSE-based message reception (`/api/v1/events`)
- JSON-RPC send (`/api/v1/send`)
- Group message buffering for context
- Mention detection with `@` or UUID
- Attachment handling via filesystem copy
- Command system with `/` prefix

**Files to Reference:**
- `/home/kacper/nanobot/nanobot/channels/signal.py`

---

## Glossary

- **SSE**: Server-Sent Events, one-way HTTP stream from server to client
- **JSON-RPC**: Remote Procedure Call using JSON over HTTP
- **E-ink**: Electrophoretic display technology (low refresh rate, high readability)
- **UpdateMode**: E-ink refresh mode (tradeoff between speed and quality)
- **View**: UI component in Cadmus framework
- **Hub/Bus**: Cadmus event messaging system
- **signal-cli**: Command-line interface for Signal, daemon mode with HTTP API
- **Bridge**: Intermediary server between Cadmus and signal-cli

---

**End of Design Document**
