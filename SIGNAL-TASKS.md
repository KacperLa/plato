# Signal Messaging Integration - Task Breakdown

This document lists all tasks for implementing Signal messaging in Cadmus, organized sequentially for autonomous agent execution.

**Epic ID:** `td-7c2359`  
**Total Tasks:** 24  
**Priority:** P0 (Critical)

---

## Task Execution Guide for Autonomous Agents

### Prerequisites
- Read `SIGNAL-MESSAGING-DESIGN.md` for comprehensive design details
- Reference nanobot Signal implementation: `/home/kacper/nanobot/nanobot/channels/signal.py`
- Understand Cadmus architecture from exploration

### Workflow
1. Start with `td-06746a` (Phase 1.1)
2. Complete tasks sequentially following dependency chain
3. After each task:
   - Run verification steps listed in task description
   - Mark as complete: `/home/kacper/go/bin/td review {task-id}`
   - Move to next task (dependencies automatically tracked)
4. Use `td status` to see current progress
5. Use `td show {task-id}` to see full task details

---

## Phase 1: Bridge Server (Host-Side) - Days 1-3

### td-06746a - Phase 1.1: Initialize signal-bridge crate
**Status:** Open  
**Depends on:** None  
**Blocks:** All Phase 1 tasks

**Description:**
Create new Rust crate for bridge server with dependencies and basic structure.

**Key Actions:**
- Create `crates/signal-bridge/` directory structure
- Write `Cargo.toml` with dependencies (axum, tokio, sqlx, reqwest, serde, etc.)
- Add to workspace in root `Cargo.toml`
- Create basic `main.rs` with hello world axum server
- Create `README.md` and `config.example.toml`

**Verification:**
```bash
cargo build -p signal-bridge
```

---

### td-113eb8 - Phase 1.2: Implement database schema and migrations
**Status:** Open  
**Depends on:** td-06746a  
**Blocks:** td-9ca2ed

**Description:**
Create SQLite schema for conversations and messages with migrations.

**Key Actions:**
- Create `migrations/001_init.sql` with full schema
- Implement `src/db.rs` with sqlx connection pool
- Migration runner on startup
- Create conversations and messages tables
- Create indexes for performance

**Verification:**
```bash
sqlite3 signal_bridge.db ".schema"
```

---

### td-d79497 - Phase 1.3: Implement configuration system
**Status:** Open  
**Depends on:** td-06746a  
**Blocks:** td-11ec7b

**Description:**
TOML-based configuration system with validation.

**Key Actions:**
- Create `src/config.rs` with Config struct
- Sections: signal_cli, server, database, logging
- TOML deserialization
- Validation and error messages
- Load in main.rs

**Verification:**
Test with valid and invalid config files.

---

### td-11ec7b - Phase 1.4: Implement signal-cli HTTP client
**Status:** Open  
**Depends on:** td-d79497  
**Blocks:** td-9ca2ed

**Description:**
HTTP client for signal-cli JSON-RPC API with SSE support.

**Key Actions:**
- Create `src/signal_client.rs` with SignalClient
- Create `src/models.rs` for Envelope, DataMessage structures
- Methods: send_message(), send_group_message(), subscribe_events()
- SSE stream parsing
- Error handling and retry logic

**Reference:**
- `/home/kacper/nanobot/nanobot/channels/signal.py`
- signal-cli documentation

**Verification:**
Unit tests with mock responses, integration test with signal-cli daemon.

---

### td-9ca2ed - Phase 1.5: Implement database operations (CRUD)
**Status:** Open  
**Depends on:** td-113eb8, td-11ec7b  
**Blocks:** td-56a0f8

**Description:**
Database CRUD operations for conversations and messages.

**Key Actions:**
- Implement in `src/db.rs`:
  - insert_conversation(), update_conversation_last_message()
  - increment_unread_count(), reset_unread_count()
  - list_conversations()
  - insert_message(), get_messages() with pagination
  - mark_messages_read(), get_unread_count()
- Use sqlx prepared statements
- Transaction support

**Verification:**
Unit tests with in-memory SQLite, test pagination and concurrency.

---

### td-56a0f8 - Phase 1.6: Implement REST API endpoints
**Status:** Open  
**Depends on:** td-9ca2ed  
**Blocks:** td-328ca7

**Description:**
REST API routes using axum framework.

**Key Actions:**
- Create `src/api.rs` with route handlers
- Create `src/error.rs` for custom errors
- Endpoints:
  - GET /api/conversations
  - GET /api/conversations/:id/messages
  - POST /api/conversations/:id/messages
  - POST /api/conversations/:id/read
  - POST /api/sync
- CORS middleware
- JSON serialization
- Error handling with proper HTTP codes

**Verification:**
curl tests for all endpoints, test error cases.

---

### td-328ca7 - Phase 1.7: Implement Server-Sent Events broadcast
**Status:** Open  
**Depends on:** td-56a0f8  
**Blocks:** td-9ebbc4

**Description:**
SSE stream for real-time updates to Cadmus clients.

**Key Actions:**
- Create `src/sse.rs` with broadcast system
- GET /api/events endpoint (text/event-stream)
- Client registry, broadcast channel
- Event types: new_message, message_read, conversation_updated
- Keepalive pings every 30s
- Graceful disconnect handling

**Verification:**
```bash
curl -N http://localhost:3030/api/events
```
Test multiple clients, verify event delivery.

---

### td-9ebbc4 - Phase 1.8: Implement background sync loop
**Status:** Open  
**Depends on:** td-328ca7  
**Blocks:** td-7035ca

**Description:**
Background task that syncs messages from signal-cli to database.

**Key Actions:**
- Create `src/sync.rs`
- Subscribe to signal-cli SSE on startup
- Parse message envelopes
- Store messages in DB
- Update conversation metadata
- Broadcast to SSE clients
- Reconnection logic

**Reference:**
nanobot `_handle_receive_notification()` method

**Verification:**
Send from Signal mobile, verify in DB and SSE clients receive.

---

### td-7035ca - Phase 1.9: Bridge server integration and testing
**Status:** Open  
**Depends on:** td-9ebbc4  
**Blocks:** Phase 2 tasks

**Description:**
Complete bridge server with all components integrated.

**Key Actions:**
- Integrate all modules in main.rs
- Startup sequence: config, DB, signal-cli, API, sync
- Graceful shutdown
- Structured logging
- Health check endpoint
- End-to-end testing

**Verification:**
```bash
cargo build --release
./target/release/signal-bridge
curl http://localhost:3030/api/conversations
curl http://localhost:3030/health
```

---

## Phase 2: Cadmus Client (Device-Side) - Days 4-7

### td-35dd57 - Phase 2.1: Create signal client module in core
**Status:** Open  
**Depends on:** td-7035ca  
**Blocks:** td-b0c047

**Description:**
HTTP client module for Cadmus to communicate with bridge.

**Key Actions:**
- Create `crates/core/src/signal/` module
- Files: mod.rs, client.rs, models.rs, error.rs
- SignalClient with reqwest::blocking
- Methods: fetch_conversations(), fetch_messages(), send_message(), mark_read()
- Error handling, timeouts
- Add to lib.rs exports

**Verification:**
Unit tests with mock HTTP, integration test against bridge.

---

### td-b0c047 - Phase 2.2: Add Signal settings to configuration
**Status:** Open  
**Depends on:** td-35dd57  
**Blocks:** td-a15ba3

**Description:**
Add Signal configuration to Cadmus settings system.

**Key Actions:**
- Modify `crates/core/src/settings/mod.rs`
- Add SignalSettings struct
- Fields: enabled, bridge_url, auto_refresh_interval, messages_per_page, timeout
- Update Settings-sample.toml with [signal] section

**Verification:**
Load Settings.toml with signal section, verify defaults.

---

### td-a15ba3 - Phase 2.3: Implement conversation list view
**Status:** Open  
**Depends on:** td-b0c047  
**Blocks:** td-a3f810

**Description:**
Scrollable list of Signal conversations with unread counts.

**Key Actions:**
- Create `crates/core/src/view/signal/` module
- Files: mod.rs, conversations.rs, common.rs
- ConversationList implementing View trait
- Components: TopBar, scrollable list, ConversationRow
- Show name, last message, timestamp, unread badge
- Events: Refresh, OpenConversation

**Verification:**
Renders conversation list, tap opens conversation, refresh works.

---

### td-a3f810 - Phase 2.4: Implement message view with pagination
**Status:** Open  
**Depends on:** td-a15ba3  
**Blocks:** td-e7b612

**Description:**
Scrollable message history with pagination support.

**Key Actions:**
- Create `crates/core/src/view/signal/messages.rs`
- MessageView implementing View trait
- Components: TopBar, scrollable message list, LoadMoreButton
- Message bubbles: outgoing right, incoming left
- Pagination: fetch 50, load more before oldest
- Auto-scroll to bottom

**Verification:**
Open conversation shows messages, Load More works, bubbles aligned correctly.

---

### td-e7b612 - Phase 2.5: Implement compose bar and send functionality
**Status:** Open  
**Depends on:** td-a3f810  
**Blocks:** td-80776f

**Description:**
Text input and send button for composing messages.

**Key Actions:**
- Create `crates/core/src/view/signal/compose.rs`
- ComposeBar implementing View trait
- Input field, send button
- On send: call client, clear input, optimistic UI
- Error handling, disable while sending

**Verification:**
Tap input opens keyboard, send works, input clears, error shown on failure.

---

### td-80776f - Phase 2.6: Implement Signal main app view container
**Status:** Open  
**Depends on:** td-e7b612  
**Blocks:** td-4cddae

**Description:**
Main Signal app view managing navigation between list and messages.

**Key Actions:**
- Create `crates/core/src/view/signal/app.rs`
- SignalApp implementing View trait
- State machine: ConversationList or MessageView
- Navigation logic, event routing
- Integration with SignalClient

**Verification:**
Open app shows list, tap conversation shows messages, back returns to list.

---

### td-4cddae - Phase 2.7: Add Signal event types to view event system
**Status:** Open  
**Depends on:** td-80776f  
**Blocks:** td-5c877a

**Description:**
Integrate Signal events into Cadmus event system.

**Key Actions:**
- Modify `crates/core/src/view/mod.rs`
- Add Event::Signal variant
- SignalEvent enum with variants
- Event handling in SignalApp
- Hub/Bus integration

**Verification:**
Events compile, event flow works, pattern matching exhaustive.

---

### td-5c877a - Phase 2.8: Implement local caching for conversations and messages
**Status:** Open  
**Depends on:** td-4cddae  
**Blocks:** Phase 3 tasks

**Description:**
In-memory cache to reduce network requests.

**Key Actions:**
- Create `crates/core/src/signal/cache.rs`
- SignalCache struct with TTL
- Methods: get/set conversations, get/set messages, add message
- Cache invalidation logic
- Optimistic updates

**Verification:**
First fetch hits network, second uses cache, manual refresh forces network.

---

## Phase 3: Integration & Polish - Days 8-10

### td-328e6f - Phase 3.1: Implement SSE client for real-time updates
**Status:** Open  
**Depends on:** td-5c877a  
**Blocks:** td-f51a99

**Description:**
Connect to bridge SSE stream and handle real-time events.

**Key Actions:**
- Create `crates/core/src/signal/sse.rs`
- Background thread for SSE connection
- Parse event stream
- Send events to hub
- Reconnection with backoff
- Graceful shutdown

**Verification:**
Send from Signal mobile, Cadmus receives within 1s, reconnects after bridge restart.

---

### td-f51a99 - Phase 3.2: Implement error handling and retry logic
**Status:** Open  
**Depends on:** td-328e6f  
**Blocks:** td-f8d72a

**Description:**
Robust error handling for network failures.

**Key Actions:**
- Enhance error types in client.rs and error.rs
- Retry with exponential backoff
- Connection status indicator in UI
- User-friendly error notifications
- Timeout handling

**Verification:**
Stop bridge shows offline, network error shows notification, auto-reconnects.

---

### td-f8d72a - Phase 3.3: Optimize e-ink refresh modes and rendering
**Status:** Open  
**Depends on:** td-f51a99  
**Blocks:** td-254118

**Description:**
Optimize rendering for e-ink displays.

**Key Actions:**
- Use appropriate UpdateModes per action
- Batch updates to minimize refreshes
- Virtual scrolling for long lists
- Test on e-ink device/emulator

**UpdateModes:**
- Partial: message append, scroll
- Gui: conversation switch
- Fast: unread badge
- Full: when needed for ghosting

**Verification:**
Minimal flashing, smooth scroll, no unnecessary refreshes, battery acceptable.

---

### td-254118 - Phase 3.4: Add loading states and progress indicators
**Status:** Open  
**Depends on:** td-f8d72a  
**Blocks:** td-532812

**Description:**
Visual feedback for operations.

**Key Actions:**
- Loading spinner when fetching
- Sending indicator
- Pull-to-refresh feedback
- Empty state, error state
- Skeleton loading

**Verification:**
All states render correctly, indicators show at right times.

---

### td-532812 - Phase 3.5: Add Signal to main menu and app launcher
**Status:** Open  
**Depends on:** td-254118  
**Blocks:** td-775db2

**Description:**
Integrate Signal into Cadmus main menu.

**Key Actions:**
- Add AppCmd::Signal variant
- Add menu entry with icon
- Launch handler in app.rs
- View stack management
- Unread badge on menu item

**Verification:**
Menu shows Signal, tap launches app, back returns, badge displays.

---

### td-775db2 - Phase 3.6: Create Signal settings UI screen
**Status:** Open  
**Depends on:** td-532812  
**Blocks:** td-4a135c

**Description:**
Settings screen for Signal configuration.

**Key Actions:**
- Settings UI for Signal config
- Toggle enable/disable
- Input fields: bridge URL, refresh interval, messages per page
- Test connection button
- Save to Settings.toml

**Verification:**
Settings accessible, inputs work, test connection validates, save persists.

---

### td-4a135c - Phase 3.7: End-to-end integration testing and bug fixes
**Status:** Open  
**Depends on:** td-775db2  
**Blocks:** td-1f0fde

**Description:**
Comprehensive testing and bug fixing.

**Key Actions:**
- Set up test environment
- Test all user flows
- Fix discovered bugs
- Performance testing
- Memory leak detection
- Network resilience testing

**Test Scenarios:**
1. Configure and open Signal
2. View conversations
3. Open conversation, view messages
4. Send message
5. Receive message
6. Group message
7. Offline/reconnect
8. Bridge restart
9. Large conversation 1000+ messages
10. Rapid sending

**Verification:**
All scenarios pass, no critical bugs, acceptable performance.

---

### td-1f0fde - Phase 3.8: Create setup documentation and deployment guide
**Status:** Open  
**Depends on:** td-4a135c  
**Blocks:** None (Final task)

**Description:**
Comprehensive documentation for setup and deployment.

**Key Actions:**
- Create doc/SIGNAL-SETUP.md
- Create doc/SIGNAL-API.md
- Update crates/signal-bridge/README.md
- Create systemd service files
- Troubleshooting guide

**Documentation sections:**
1. Prerequisites
2. Bridge setup
3. Cadmus configuration
4. Testing
5. Troubleshooting
6. Systemd services
7. Remote access setup

**Verification:**
Follow guide on fresh system, verify all steps work.

---

## Quick Reference Commands

### View all tasks
```bash
/home/kacper/go/bin/td list --epic td-7c2359
```

### Show current status
```bash
/home/kacper/go/bin/td status
```

### View task details
```bash
/home/kacper/go/bin/td show {task-id}
```

### Start working on a task
```bash
/home/kacper/go/bin/td start {task-id}
```

### Mark task for review (when complete)
```bash
/home/kacper/go/bin/td review {task-id}
```

### Approve and close task
```bash
/home/kacper/go/bin/td approve {task-id}
```

### View next task to work on
```bash
/home/kacper/go/bin/td next
```

### View task dependencies
```bash
/home/kacper/go/bin/td depends-on {task-id}
/home/kacper/go/bin/td blocked-by {task-id}
```

---

## Success Criteria

**Phase 1 Complete:**
- Bridge server runs, connects to signal-cli
- Messages sync to database
- REST API functional
- SSE broadcasts work

**Phase 2 Complete:**
- Cadmus can list conversations
- Can view message history
- Can send messages
- Basic UI functional

**Phase 3 Complete:**
- Real-time updates work
- Error handling robust
- E-ink optimized
- Production-ready
- Documented

**MVP Complete:**
- User can send/receive Signal DMs and group messages
- UI optimized for e-ink
- Reliable on local network
- Documented for setup

---

## Notes for Autonomous Agents

1. **Read design doc first**: `SIGNAL-MESSAGING-DESIGN.md` contains all architectural details
2. **Follow dependencies**: Tasks have strict dependency chains - respect the order
3. **Reference nanobot**: Use `/home/kacper/nanobot/nanobot/channels/signal.py` as reference for signal-cli integration
4. **Test thoroughly**: Each task has verification steps - complete them all
5. **Ask for clarification**: If requirements unclear, ask before implementing
6. **Document as you go**: Update code comments and inline docs
7. **Commit frequently**: Small, logical commits with clear messages
8. **Update td**: Mark tasks as complete using td commands

---

**Last Updated:** 2026-02-16  
**Total Estimated Time:** 9-14 days  
**Current Status:** All tasks open, ready to start with td-06746a
