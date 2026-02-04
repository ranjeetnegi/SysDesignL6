# Chapter 32. Messaging platform

---

# Introduction

Messaging platforms are among the most demanding distributed systems to design and operate. I've spent years building and scaling messaging infrastructure at Google, and I can tell you: the fundamental challenge isn't sending a message from A to B—any undergraduate can build that. The challenge is doing it reliably for billions of users, with sub-second latency, perfect ordering, exactly-once delivery semantics, and graceful degradation when (not if) things fail.

This chapter covers messaging platform design as Staff Engineers practice it: with deep understanding of the consistency-availability trade-offs unique to messaging, awareness of the ordering problems that seem simple but aren't, and judgment about when to sacrifice guarantees for user experience.

**The Staff Engineer's First Law of Messaging**: Users care about three things in order: (1) Did my message get delivered? (2) Did it arrive quickly? (3) Is the conversation in the right order? Everything else—read receipts, typing indicators, reactions—is polish. Get the fundamentals wrong, and no amount of features saves you.

---

## Quick Visual: Messaging Platform at a Glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  MESSAGING PLATFORM: THE STAFF ENGINEER VIEW                │
│                                                                             │
│   WRONG Framing: "Send messages between users"                              │
│   RIGHT Framing: "Deliver messages reliably with ordering guarantees,       │
│                   sub-second latency, and graceful offline handling"        │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Before designing, understand:                                      │   │
│   │                                                                     │   │
│   │  1. 1:1 messaging or group messaging or both?                       │   │
│   │  2. What ordering guarantees? (Per-sender? Per-conversation?)       │   │
│   │  3. How long can users be offline? (Minutes? Days? Months?)         │   │
│   │  4. What's the message size distribution? (Text? Media? Files?)     │   │
│   │  5. Synchronous delivery (online) or async (offline)?               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   THE UNCOMFORTABLE TRUTH:                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Perfect ordering + perfect availability + low latency is           │   │
│   │  impossible across regions. You must choose which to sacrifice.     │   │
│   │  Most messaging apps sacrifice global ordering for availability     │   │
│   │  and use sender-ordering + local clocks for "good enough" UX.       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## L5 vs L6 Messaging Platform Decisions

| Scenario | L5 Approach | L6 Approach |
|----------|-------------|-------------|
| **Message ordering** | "Use a single message queue for total ordering" | "Total ordering doesn't scale. Use per-conversation ordering with Lamport timestamps. Users only see their own conversation anyway." |
| **Delivery guarantees** | "Use exactly-once delivery with 2PC" | "Exactly-once is expensive. Use at-least-once with idempotent receivers. Duplicate detection at client is cheaper than prevention." |
| **Offline message storage** | "Store all messages forever in hot storage" | "Hot storage for 30 days, warm for 1 year, cold archive beyond. 99% of reads are last 24 hours." |
| **Group messaging** | "Fan-out on write for fast reads" | "Fan-out on read for small groups, fan-out on write for large groups. Threshold at ~50 members." |
| **Presence (online status)** | "Real-time presence with WebSocket heartbeats" | "Presence is eventually consistent. 30-second staleness is acceptable. Don't make presence a scaling bottleneck." |

**Key Difference**: L6 engineers recognize that messaging systems have fundamentally different requirements than traditional request-response systems. They design for the unique challenges of long-lived connections, offline users, and conversation-local consistency.

---

# Part 1: Foundations — What a Messaging Platform Is and Why It Exists

## What Is a Messaging Platform?

A messaging platform enables asynchronous communication between users through discrete messages. Unlike email (store-and-forward, minutes-to-hours delivery) or phone calls (synchronous, real-time), messaging platforms occupy a middle ground: near-real-time when users are online, reliably stored when they're offline.

### The Simplest Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  MESSAGING: THE POSTAL SYSTEM ANALOGY                       │
│                                                                             │
│   Think of messaging as a modern postal system:                             │
│                                                                             │
│   SENDER writes a letter (message)                                          │
│   ↓                                                                         │
│   POST OFFICE (message service) receives it                                 │
│   ↓                                                                         │
│   If RECIPIENT is home (online): Deliver immediately                        │
│   If RECIPIENT is away (offline): Hold at post office, deliver on return   │
│   ↓                                                                         │
│   RECIPIENT receives letter, sends acknowledgment (read receipt)            │
│                                                                             │
│   COMPLICATIONS AT SCALE:                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  What if there are millions of post offices (distributed system)?  │   │
│   │  → Need coordination for which office holds which letters          │   │
│   │                                                                     │   │
│   │  What if recipient has multiple homes (multiple devices)?          │   │
│   │  → Need to track delivery to ALL devices, not just one             │   │
│   │                                                                     │   │
│   │  What if letters arrive out of order?                               │   │
│   │  → Need sequence numbers to reorder at recipient                   │   │
│   │                                                                     │   │
│   │  What if sender sends to 1000 recipients (group chat)?             │   │
│   │  → Need efficient fan-out, not 1000 individual deliveries          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Why Messaging Platforms Exist

### 1. Asynchronous Communication at Human Speed

Email is too slow for conversations. Phone calls require both parties simultaneously. Messaging bridges the gap:

```
Scenario: Coordinating dinner plans

EMAIL:
T+0min:   Alice sends "Dinner at 7?"
T+5min:   Bob checks email, replies "Sure, where?"
T+12min:  Alice checks email, replies "That Italian place"
T+20min:  Bob confirms
TOTAL: 20+ minutes for simple coordination

PHONE:
T+0min:   Alice calls Bob
          Bob doesn't answer (in meeting)
T+30min:  Bob calls back
          Alice doesn't answer (now in meeting)
T+60min:  Finally connect, 2-minute conversation
TOTAL: 60+ minutes wall clock time

MESSAGING:
T+0min:   Alice sends "Dinner at 7?"
T+0min:   Bob sees notification, replies "Sure, where?"
T+1min:   Alice replies "That Italian place"
T+1min:   Bob sends 👍
TOTAL: 1-2 minutes, no synchronization required
```

### 2. Persistent Conversation History

Unlike phone calls, messages create a record:

```
Benefits:
• Search old conversations for decisions/agreements
• Onboard new group members with context
• Reference shared links, files, addresses
• Legal/compliance record-keeping
• Memory augmentation ("What did we decide last week?")
```

### 3. Multi-Device, Multi-Platform Access

Modern messaging works across all devices simultaneously:

```
User's devices:
├── Phone (primary, always connected)
├── Tablet (occasional use)
├── Laptop (work hours)
└── Desktop (home)

Message arrives:
→ Delivered to ALL devices
→ Read status synced across devices
→ User can respond from any device
→ Conversation continues seamlessly
```

### 4. Rich Communication Beyond Text

Modern platforms support:

```
Content types:
├── Text (with emoji, formatting)
├── Images (with compression, thumbnails)
├── Videos (with streaming, transcoding)
├── Voice messages (with waveform preview)
├── Files/documents
├── Location sharing
├── Contact cards
├── Polls and interactive elements
└── Reactions and replies
```

## What Happens If a Messaging Platform Does NOT Exist (or Fails)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  MESSAGING PLATFORM FAILURES                                │
│                                                                             │
│   FAILURE MODE 1: MESSAGE LOSS                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  User sends critical message → Message disappears → No notification │   │
│   │  → User believes message was delivered → Relationship damaged       │   │
│   │                                                                     │   │
│   │  Real example: "I told you I couldn't make it!" / "I never got it" │   │
│   │  This is THE cardinal sin of messaging platforms                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   FAILURE MODE 2: MESSAGE REORDERING                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Messages arrive out of order → Conversation is confusing          │   │
│   │                                                                     │   │
│   │  Alice: "Should we go to dinner?"                                   │   │
│   │  Bob: "Yes"                                                         │   │
│   │  Alice: "Or maybe just drinks?"                                     │   │
│   │                                                                     │   │
│   │  Bob sees: "Or maybe just drinks?" then "Should we go to dinner?"  │   │
│   │  Bob's "Yes" now makes no sense                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   FAILURE MODE 3: DUPLICATE MESSAGES                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Same message delivered multiple times → User confusion/annoyance  │   │
│   │                                                                     │   │
│   │  Less severe than loss, but still breaks trust                     │   │
│   │  "Why did you send that 5 times?"                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   FAILURE MODE 4: PRESENCE LIES                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Shows "online" when offline or vice versa                         │   │
│   │  → Social friction ("Why aren't you responding? You're online!")   │   │
│   │                                                                     │   │
│   │  Staff insight: This is why many apps show "last seen" instead     │   │
│   │  of "online now"—it's more forgiving of staleness                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   FAILURE MODE 5: SYNC DIVERGENCE                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Different devices show different message history                  │   │
│   │  → User confusion about what was actually said                     │   │
│   │  → "I can see it on my phone but not my laptop"                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: Functional Requirements

## Core Use Cases

### 1. Send a Message (1:1 Conversation)

```
Use Case: Alice sends "Hello" to Bob

Input: 
  - Sender: Alice
  - Recipient: Bob  
  - Content: "Hello"
  - Timestamp: 2024-01-15T10:30:00Z

Process:
  1. Authenticate Alice
  2. Validate message (size, content policy)
  3. Generate message ID (globally unique)
  4. Store message durably
  5. If Bob online: Push to Bob's device(s)
  6. If Bob offline: Queue for later delivery
  7. Acknowledge to Alice (single checkmark)
  8. When Bob receives: Update to delivered (double checkmark)
  9. When Bob reads: Update to read (blue checkmarks)

Output:
  - Message ID for reference
  - Delivery status updates
```

### 2. Send a Message (Group Conversation)

```
Use Case: Alice sends "Meeting at 3" to group "Project Team" (50 members)

Input:
  - Sender: Alice
  - Group: project_team_123
  - Content: "Meeting at 3"

Process:
  1. Authenticate Alice
  2. Verify Alice is group member
  3. Generate message ID
  4. Store message (once, not 50 times)
  5. Fan-out delivery to 50 members
  6. Track per-member delivery status

Complications:
  - Some members online, some offline
  - Some members have notifications muted
  - Some members blocked Alice
  - Members in different timezones
```

### 3. Receive Messages (Real-time)

```
Use Case: Bob receives messages while app is open

Mechanism: Long-lived WebSocket connection

Process:
  1. Bob's device establishes WebSocket to message gateway
  2. Gateway registers Bob's connection in presence service
  3. When message arrives for Bob:
     a. Look up Bob's active connections
     b. Push message through WebSocket
     c. Wait for ACK from device
     d. Mark message as delivered
  4. If connection drops: Re-establish with resume token

Latency target: < 500ms from send to receive
```

### 4. Receive Messages (Offline Sync)

```
Use Case: Bob opens app after 3 days offline

Process:
  1. Bob's device connects with last_sync_timestamp
  2. Server queries: "All messages for Bob since timestamp"
  3. Return messages in batches (pagination)
  4. For each conversation: Return messages in order
  5. Client merges with local state
  6. Update sync checkpoint

Complications:
  - Bob might have 10,000 unread messages
  - Some messages might be deleted by sender
  - Media might need re-download
  - Group memberships might have changed
```

### 5. Typing Indicators

```
Use Case: Alice sees "Bob is typing..."

Mechanism: Ephemeral, unreliable signaling

Process:
  1. Bob types → Client sends "typing" signal
  2. Signal routed to Alice (if online)
  3. Alice displays indicator
  4. Indicator auto-expires after 5 seconds
  5. No persistence, no retry, no guarantee

Staff insight: Typing indicators are cosmetic.
- Lost typing indicators = no harm
- Stale typing indicators = minor annoyance
- Don't optimize for this; focus on message delivery
```

### 6. Read Receipts

```
Use Case: Alice sees that Bob read her message

Process:
  1. Bob's device renders message on screen
  2. Client sends read_receipt(message_id, timestamp)
  3. Receipt stored and forwarded to Alice
  4. Alice's UI updates to show "read"

Privacy considerations:
  - Some users disable read receipts
  - Group read receipts: Show count, not names
  - Read != comprehended (user may have scrolled past)
```

### 7. Message Editing and Deletion

```
Use Case: Alice edits typo / deletes embarrassing message

Edit process:
  1. Alice sends edit(message_id, new_content)
  2. Validate: Alice is author, within edit window
  3. Update message content
  4. Push edit to all recipients
  5. UI shows "edited" indicator

Delete process:
  1. Alice sends delete(message_id)
  2. Options:
     a. Delete for self only (just hide in Alice's view)
     b. Delete for everyone (remove content, show "deleted")
  3. Delete window: Typically 1-48 hours

Staff insight: "Delete for everyone" is contentious.
- Recipient may have already seen/screenshotted
- Creates strange conversation gaps
- Some platforms show "Alice deleted a message"
```

## Read Paths

```
// Get conversation history (hot path)
FUNCTION get_messages(conversation_id, user_id, cursor, limit):
    // Verify access
    IF NOT is_member(user_id, conversation_id):
        RETURN ERROR("Access denied")
    
    // Fetch messages
    messages = query_messages(
        conversation_id, 
        before_cursor=cursor, 
        limit=limit
    )
    
    // Fetch media URLs (signed, time-limited)
    FOR message IN messages:
        IF message.has_media:
            message.media_url = generate_signed_url(message.media_id)
    
    RETURN {
        messages: messages,
        next_cursor: messages.last().id,
        has_more: count > limit
    }

// Get conversation list
FUNCTION get_conversations(user_id, cursor, limit):
    conversations = query_user_conversations(
        user_id,
        order_by=last_message_time DESC,
        limit=limit
    )
    
    // Include snippet of last message
    FOR conv IN conversations:
        conv.last_message = get_last_message(conv.id)
        conv.unread_count = get_unread_count(conv.id, user_id)
    
    RETURN conversations
```

## Write Paths

```
// Send message (critical path)
FUNCTION send_message(sender_id, conversation_id, content, media):
    // Validate
    IF NOT is_member(sender_id, conversation_id):
        RETURN ERROR("Not a member")
    
    IF length(content) > MAX_MESSAGE_SIZE:
        RETURN ERROR("Message too long")
    
    // Generate IDs
    message_id = generate_snowflake_id()
    
    // Handle media upload (async)
    IF media:
        media_id = upload_media_async(media)
    
    // Create message
    message = {
        id: message_id,
        conversation_id: conversation_id,
        sender_id: sender_id,
        content: content,
        media_id: media_id,
        created_at: now(),
        sequence: get_next_sequence(conversation_id)
    }
    
    // Persist (must succeed before ACK)
    store_message(message)
    
    // Update conversation metadata
    update_conversation_last_message(conversation_id, message)
    
    // Trigger delivery (async)
    enqueue_delivery(message)
    
    RETURN {
        message_id: message_id,
        status: "sent",
        timestamp: message.created_at
    }
```

## Control / Admin Paths

```
// Create group conversation
FUNCTION create_group(creator_id, name, member_ids):
    IF len(member_ids) > MAX_GROUP_SIZE:
        RETURN ERROR("Group too large")
    
    conversation_id = generate_id()
    
    create_conversation({
        id: conversation_id,
        type: "group",
        name: name,
        created_by: creator_id,
        created_at: now()
    })
    
    // Add members (including creator)
    FOR member_id IN [creator_id] + member_ids:
        add_member(conversation_id, member_id, role="member")
    
    set_admin(conversation_id, creator_id)
    
    // Send system message
    send_system_message(conversation_id, "Group created")
    
    RETURN conversation_id

// Block user
FUNCTION block_user(blocker_id, blocked_id):
    create_block(blocker_id, blocked_id)
    
    // Prevent future messages
    // Hide existing conversations (don't delete)
    // Remove from each other's contact lists
    
    RETURN SUCCESS

// Report content
FUNCTION report_message(reporter_id, message_id, reason):
    message = get_message(message_id)
    
    create_report({
        reporter_id: reporter_id,
        message_id: message_id,
        conversation_id: message.conversation_id,
        content_snapshot: message.content,
        reason: reason,
        created_at: now()
    })
    
    // Trigger async review workflow
    enqueue_moderation_review(report)
    
    RETURN SUCCESS
```

## Edge Cases

### Edge Case 1: Sending to Blocked User

```
Alice blocked Bob. Bob tries to send message to Alice.

Options:
A) Error: "Cannot send to this user" (reveals block)
B) Silent success: Message appears sent but never delivered
C) Delayed failure: "Message could not be delivered"

Staff approach: Option B (silent success)
- Prevents harassment escalation
- Blocks are private information
- Bob sees sent status, Alice never sees message
```

### Edge Case 2: Group Message to Mixed Online/Offline

```
Group has 100 members:
- 40 online
- 30 offline (will be back today)
- 20 offline (haven't opened app in months)
- 10 have uninstalled app

Message delivery:
1. Immediate push to 40 online members
2. Store for 30 recently-active offline members
3. Store but don't push-notify 20 inactive members
4. Detect uninstalled via push failure, mark for cleanup
```

### Edge Case 3: Message During Network Partition

```
Alice sends message, but network fails before ACK

Client behavior:
1. Show "sending..." indicator
2. Retry with exponential backoff
3. If server ACKs: Update to "sent"
4. If server rejects as duplicate: Already delivered, update status
5. If timeout after N retries: Show "failed to send, tap to retry"

Server behavior:
- Use client-generated message ID for idempotency
- If same ID received twice, return success (already stored)
```

### Edge Case 4: Device Clock Far in Future

```
Alice's phone clock is 1 year ahead.
Message timestamp: 2025-01-15 (actual date: 2024-01-15)

Problems:
- Message sorts to wrong position
- "Last seen" shows future date
- Message expiry confused

Staff approach:
- Server assigns authoritative timestamp
- Client timestamp used only for offline sorting
- On sync, server timestamp wins
```

## What Is Intentionally OUT of Scope

| Excluded | Why |
|----------|-----|
| Voice/video calling | Different infrastructure (WebRTC, media servers) |
| Stories/status updates | Different content model (broadcast, ephemeral) |
| End-to-end encryption | Massive complexity, covered separately |
| Content moderation ML | Separate system, messaging just provides data |
| Payment/commerce | Different compliance, security requirements |
| Bots/integrations platform | API layer on top of core messaging |

---

# Part 3: Non-Functional Requirements

## Latency Expectations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LATENCY REQUIREMENTS                                     │
│                                                                             │
│   MESSAGE SEND (User presses send → sees "sent"):                           │
│   • P50: < 100ms (feels instant)                                            │
│   • P95: < 300ms (acceptable)                                               │
│   • P99: < 1000ms (user notices but tolerates)                              │
│   • > 2000ms: User retries, potential duplicate                             │
│                                                                             │
│   MESSAGE DELIVERY (Send → recipient sees):                                 │
│   • P50: < 500ms (real-time feel)                                           │
│   • P95: < 2000ms (still responsive)                                        │
│   • P99: < 5000ms (degraded but working)                                    │
│                                                                             │
│   CONVERSATION LOAD (Open chat → see messages):                             │
│   • P50: < 200ms (instant)                                                  │
│   • P99: < 500ms (smooth)                                                   │
│                                                                             │
│   TYPING INDICATORS:                                                        │
│   • < 300ms to appear (feels real-time)                                     │
│   • Can drop indicators under load (cosmetic)                               │
│                                                                             │
│   STAFF INSIGHT:                                                            │
│   Message send latency is sacred—it's the user's signal that the           │
│   system is working. Everything else can degrade.                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Availability Expectations

```
Target: 99.99% availability (52 minutes downtime/year)

What "available" means for messaging:
1. Users can send messages (even if delivery is delayed)
2. Users can read existing messages
3. New messages eventually arrive

Acceptable degradations during partial outage:
• Delayed delivery (minutes, not hours)
• Missing typing indicators
• Stale presence information
• Slow media upload/download
• Missing read receipts

Unacceptable during any outage:
• Message loss
• Message corruption
• Unauthorized access to messages
```

## Consistency Needs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY MODEL                                        │
│                                                                             │
│   STRONG CONSISTENCY REQUIRED:                                              │
│   • Message ordering within conversation (per-sender at minimum)            │
│   • Membership changes (who can see which messages)                         │
│   • Block/unblock (must take effect immediately)                            │
│                                                                             │
│   EVENTUAL CONSISTENCY ACCEPTABLE:                                          │
│   • Read receipts (can lag by seconds)                                      │
│   • Typing indicators (ephemeral anyway)                                    │
│   • Presence status (30-second staleness OK)                                │
│   • Unread counts (can lag)                                                 │
│   • Cross-device sync (seconds of delay OK)                                 │
│                                                                             │
│   THE FUNDAMENTAL TRADE-OFF:                                                │
│   Global total ordering of messages is impossible at scale.                │
│   We use conversation-local ordering with Lamport-style timestamps.        │
│                                                                             │
│   Within single conversation: Messages appear in consistent order           │
│   Across conversations: No global ordering guarantee                        │
│   Across regions: May see messages from other region with lag               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Durability

```
Message durability requirements:
• Messages must NEVER be lost after ACK to sender
• Durability: 99.999999999% (11 nines) for message content
• This requires synchronous replication before ACK

Strategy:
1. Write to primary storage (with sync replication)
2. Only ACK to sender after durable write confirmed
3. Async replication to secondary regions for disaster recovery

Media durability:
• Same 11 nines for media files
• Store in blob storage with cross-region replication
• Keep metadata even if media expires (show "media expired")
```

## Correctness vs User Experience Trade-offs

```
CORRECTNESS:                           USER EXPERIENCE:
─────────────────────────────────────────────────────────
Wait for full sync before showing      Show cached data immediately,
conversation                           sync in background
                                       → Choose UX (stale > slow)

Block all operations during            Allow reads, queue writes
network issues                         → Choose UX (partial > nothing)

Show precise message timestamps        Show "2 minutes ago"
(10:42:37.123)                         → Choose UX (human-readable)

Enforce strict ordering                Allow out-of-order display,
(wait for missing messages)            reorder when gap fills
                                       → Depends: strict for 1:1,
                                         relaxed for busy groups

Verify read receipt delivery           Best-effort read receipts
before showing "read"                  → Choose UX (fast > accurate)
```

## Security Implications

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURITY REQUIREMENTS                                    │
│                                                                             │
│   CONFIDENTIALITY:                                                          │
│   • Messages visible only to conversation participants                      │
│   • No leakage through errors, logs, or side channels                       │
│   • Encryption in transit (TLS)                                             │
│   • Encryption at rest (storage encryption)                                 │
│   • Optional: End-to-end encryption (E2EE)                                  │
│                                                                             │
│   INTEGRITY:                                                                │
│   • Messages cannot be modified in transit                                  │
│   • Message history cannot be tampered with                                 │
│   • Sender identity is authentic                                            │
│                                                                             │
│   ACCESS CONTROL:                                                           │
│   • Only authenticated users can send/receive                               │
│   • Group membership enforced                                               │
│   • Block lists enforced                                                    │
│   • Admin actions properly authorized                                       │
│                                                                             │
│   STAFF INSIGHT:                                                            │
│   Messaging has strict privacy expectations. A single bug that              │
│   leaks messages to wrong recipient is a PR disaster and potential          │
│   legal liability. Defense in depth is essential.                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 4: Scale & Load Modeling

## User Scale

```
Reference: WhatsApp-scale messaging platform

Users:
• Total registered users: 2 billion
• Daily active users (DAU): 500 million
• Monthly active users (MAU): 1.5 billion
• Concurrent connections: 100 million (peak)

Conversations:
• Average conversations per user: 20
• Total 1:1 conversations: 10 billion
• Total group conversations: 500 million
• Average group size: 10 members
• Large groups (>100 members): 5 million
```

## Message Volume

```
Messages per day:
• Total messages: 100 billion/day
• Average per DAU: 200 messages sent+received
• Peak hour: 10 billion messages (10x average)

Message breakdown:
• Text only: 70% (avg 100 bytes)
• Images: 20% (avg 200KB compressed)
• Video: 5% (avg 5MB)
• Voice: 3% (avg 100KB)
• Other (files, location): 2%
```

## QPS Calculations

```
STEADY STATE:
Messages: 100B/day ÷ 86,400 sec = 1.15M messages/sec
Peak: ~10M messages/sec

Per message, multiple operations:
• 1 write (store message)
• N reads (N recipients, plus sender confirmation)
• 1-3 push notifications
• 1 unread count update

Effective operations:
• Write QPS: 10M/sec (peak)
• Read QPS: 50M/sec (peak) - 5 reads per message avg
• Push QPS: 30M/sec (peak)

CONNECTION MANAGEMENT:
• 100M concurrent WebSocket connections
• Each connection: heartbeat every 30 seconds
• Heartbeat QPS: 3.3M/sec
```

## Storage Requirements

```
MESSAGE STORAGE:
Daily text: 70B messages × 100 bytes = 7TB/day
Daily images: 20B × 200KB = 4PB/day
Daily video: 5B × 5MB = 25PB/day
Daily total: ~30PB/day

With retention:
• Hot storage (30 days): 900PB
• Warm storage (1 year): 10EB
• Cold storage (archive): ∞

METADATA STORAGE:
• Per message: ~500 bytes metadata
• 100B messages × 500 bytes = 50TB/day
• Indexes: 2x metadata = 100TB/day

CONVERSATION METADATA:
• 10B conversations × 1KB = 10TB
• Relatively static, heavily cached
```

## Burst Behavior

```
PREDICTABLE BURSTS:
• New Year's midnight: 100x normal (by timezone)
• Major sporting events: 10-50x
• Celebrity deaths/announcements: 20x
• Religious holidays: 5-10x

UNPREDICTABLE BURSTS:
• Breaking news: 10-20x
• Viral content: Localized 50x spikes
• Platform outage recovery: 10x (message backlog)

SCALING IMPLICATIONS:
• Must handle 100x burst without data loss
• Acceptable to delay delivery during extreme bursts
• Must shed load gracefully (typing indicators first)
```

## What Breaks First

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SCALING BOTTLENECKS (In Order)                           │
│                                                                             │
│   1. CONNECTION SERVERS (First to break)                                    │
│      • 100M WebSocket connections = massive memory                          │
│      • Connection storms after outage                                       │
│      • Solution: Horizontal scaling, connection limits per user             │
│                                                                             │
│   2. MESSAGE FANOUT FOR LARGE GROUPS                                        │
│      • 1000-member group = 1000 deliveries per message                      │
│      • Active 1000-member group = 1M deliveries/day                        │
│      • Solution: Read fanout for large groups, lazy delivery               │
│                                                                             │
│   3. PRESENCE SERVICE                                                       │
│      • 100M users each updating presence                                    │
│      • Naive approach: N² presence checks                                   │
│      • Solution: Approximate presence, subscription limits                  │
│                                                                             │
│   4. HOT CONVERSATIONS                                                      │
│      • Celebrity/influencer conversations                                   │
│      • Millions of fans messaging same person                               │
│      • Solution: Queue and rate-limit inbound, special handling             │
│                                                                             │
│   5. MEDIA STORAGE COSTS                                                    │
│      • 30PB/day is $$$                                                      │
│      • Solution: Aggressive compression, expiry, tiered storage             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 5: High-Level Architecture

## Core Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGING PLATFORM ARCHITECTURE                          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         CLIENTS                                     │   │
│   │         Mobile Apps  │  Web App  │  Desktop Apps                    │   │
│   └────────────────────────────┬────────────────────────────────────────┘   │
│                                │                                            │
│                                ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      EDGE / API LAYER                               │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│   │  │   API       │  │  WebSocket  │  │   Push      │                  │   │
│   │  │   Gateway   │  │  Gateway    │  │   Gateway   │                  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│   └────────────────────────────┬────────────────────────────────────────┘   │
│                                │                                            │
│                                ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      CORE SERVICES                                  │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│   │  │   Message   │  │  Delivery   │  │  Presence   │                  │   │
│   │  │   Service   │  │  Service    │  │  Service    │                  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│   │  │   Sync      │  │  Group      │  │  User       │                  │   │
│   │  │   Service   │  │  Service    │  │  Service    │                  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│   └────────────────────────────┬────────────────────────────────────────┘   │
│                                │                                            │
│                                ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      DATA LAYER                                     │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│   │  │   Message   │  │   User      │  │   Media     │                  │   │
│   │  │   Store     │  │   Store     │  │   Store     │                  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│   │  ┌─────────────┐  ┌─────────────┐                                   │   │
│   │  │   Cache     │  │   Message   │                                   │   │
│   │  │   Layer     │  │   Queue     │                                   │   │
│   │  └─────────────┘  └─────────────┘                                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

### API Gateway
- HTTP REST endpoints for non-real-time operations
- Authentication and authorization
- Rate limiting
- Request routing

### WebSocket Gateway
- Maintains long-lived connections with clients
- Routes real-time messages to correct connections
- Handles connection lifecycle (connect, heartbeat, disconnect)
- Stateful: knows which users are connected

### Push Gateway
- Sends push notifications to offline users
- Integrates with APNs (iOS), FCM (Android)
- Batches and prioritizes notifications
- Handles token management and failures

### Message Service
- Core message CRUD operations
- Message validation and storage
- Sequence number assignment
- Handles edits and deletes

### Delivery Service
- Routes messages to recipients
- Manages online vs offline delivery
- Tracks delivery and read status
- Handles retries and failures

### Presence Service
- Tracks online/offline status
- Manages "last seen" timestamps
- Subscription model for presence updates
- Handles privacy settings

### Sync Service
- Provides message history for offline clients
- Handles multi-device synchronization
- Manages sync cursors and checkpoints
- Resolves conflicts

### Group Service
- Group CRUD operations
- Membership management
- Admin functions
- Group metadata

### User Service
- User profiles and settings
- Contact lists
- Block lists
- Preferences

## Stateless vs Stateful Decisions

```
STATELESS SERVICES:
• Message Service: Processes requests, stores in DB
• Group Service: CRUD operations, DB-backed
• User Service: Profile operations, DB-backed
• Sync Service: Query and return, no local state

STATEFUL SERVICES:
• WebSocket Gateway: Holds active connections
  - State: user_id → connection mapping
  - Failure: connections drop, clients reconnect
  
• Presence Service: Tracks who's online
  - State: online users, last activity
  - Failure: presence becomes stale, self-heals
  
• Delivery Service: In-flight message tracking
  - State: pending deliveries
  - Failure: re-query from message store, retry

STAFF INSIGHT:
Stateful services are harder to scale and recover.
We minimize state and make it reconstructible.
WebSocket state is client-recoverable (reconnect).
Presence state is ephemeral (rebuild on restart).
Delivery state is checkpointed (resume from checkpoint).
```

## Data Flow: Send Message

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SEND MESSAGE FLOW                                        │
│                                                                             │
│   1. SENDER DEVICE                                                          │
│      │                                                                      │
│      ▼                                                                      │
│   2. API GATEWAY                                                            │
│      • Authenticate request                                                 │
│      • Rate limit check                                                     │
│      │                                                                      │
│      ▼                                                                      │
│   3. MESSAGE SERVICE                                                        │
│      • Validate message                                                     │
│      • Generate message ID + sequence number                                │
│      • Store message in MESSAGE STORE (sync write)                          │
│      • Return ACK to sender                                                 │
│      │                                                                      │
│      ▼                                                                      │
│   4. DELIVERY SERVICE (async)                                               │
│      • Look up conversation members                                         │
│      • For each recipient:                                                  │
│        │                                                                    │
│        ├─► ONLINE: Query PRESENCE SERVICE                                   │
│        │     └─► Send via WEBSOCKET GATEWAY                                 │
│        │                                                                    │
│        └─► OFFLINE: Push notification via PUSH GATEWAY                      │
│                                                                             │
│   5. RECIPIENT DEVICE(S)                                                    │
│      • Receive message via WebSocket or push                                │
│      • Send delivery ACK                                                    │
│      │                                                                      │
│      ▼                                                                      │
│   6. DELIVERY SERVICE                                                       │
│      • Update delivery status                                               │
│      • Notify sender of delivery                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow: Receive Messages (Sync)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SYNC MESSAGE FLOW                                        │
│                                                                             │
│   1. CLIENT connects after being offline                                    │
│      • Sends: last_sync_id = "msg_12345"                                    │
│      │                                                                      │
│      ▼                                                                      │
│   2. SYNC SERVICE                                                           │
│      • Query: all messages for user since msg_12345                         │
│      • Group by conversation                                                │
│      │                                                                      │
│      ▼                                                                      │
│   3. MESSAGE STORE                                                          │
│      • Index: (user_id, message_id) for inbox                               │
│      • Returns paginated results                                            │
│      │                                                                      │
│      ▼                                                                      │
│   4. SYNC SERVICE                                                           │
│      • Enrich with conversation metadata                                    │
│      • Fetch sender info from cache                                         │
│      • Return to client with new sync cursor                                │
│      │                                                                      │
│      ▼                                                                      │
│   5. CLIENT                                                                 │
│      • Merge with local message store                                       │
│      • Update UI                                                            │
│      • Store new sync cursor                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: Deep Component Design

## WebSocket Gateway

### Purpose
Maintains persistent connections with clients for real-time message delivery.

### Internal Data Structures

```
// Per-server connection registry
ConnectionRegistry:
    connections: Map<connection_id, Connection>
    user_connections: Map<user_id, Set<connection_id>>
    
Connection:
    connection_id: string
    user_id: string
    device_id: string
    socket: WebSocket
    connected_at: timestamp
    last_activity: timestamp
    resume_token: string

// Distributed user location
UserLocationService:
    user_locations: Map<user_id, List<ServerAddress>>
    // Backed by Redis with TTL
```

### Connection Lifecycle

```
CONNECT:
1. Client initiates WebSocket handshake
2. Gateway validates auth token
3. Generate connection_id and resume_token
4. Register in local ConnectionRegistry
5. Publish to UserLocationService: "user X on server Y"
6. Send connection ACK with resume_token

HEARTBEAT:
1. Client sends ping every 30 seconds
2. Server responds pong
3. Update last_activity timestamp
4. If no heartbeat for 90 seconds: consider dead

DISCONNECT:
1. Client closes or connection times out
2. Remove from local ConnectionRegistry
3. Remove from UserLocationService
4. Keep resume_token valid for 5 minutes (reconnect window)

RESUME:
1. Client reconnects with resume_token
2. Validate token (not expired, correct user)
3. Replay any messages since disconnect
4. Issue new resume_token
```

### Message Routing

```
FUNCTION route_message_to_user(user_id, message):
    // Find all connections for this user
    locations = UserLocationService.get(user_id)
    
    IF locations is empty:
        RETURN NOT_CONNECTED
    
    FOR location IN locations:
        IF location.server == self:
            // Local delivery
            connections = ConnectionRegistry.get_user_connections(user_id)
            FOR conn IN connections:
                conn.socket.send(message)
        ELSE:
            // Remote delivery via internal RPC
            send_to_server(location.server, user_id, message)
    
    RETURN DELIVERED
```

### Failure Behavior

```
SERVER CRASH:
• All connections on that server drop
• Clients detect via missed heartbeat (30-90 seconds)
• Clients reconnect to different server
• Messages during gap: delivered on reconnect via sync

NETWORK PARTITION:
• Connections appear alive but unresponsive
• Messages queue up, eventually timeout
• Client-side timeout triggers reconnect

OVERLOAD:
• New connections rejected with backoff header
• Existing connections maintained
• Message delivery prioritized over new connections
```

### Why Simpler Alternatives Fail

```
Alternative: HTTP Long Polling
Problems:
• Higher latency (connection setup per message)
• More server load (new connection per poll)
• Harder to maintain ordering guarantees
• More complex client-side logic

Alternative: Server-Sent Events (SSE)
Problems:
• Unidirectional (server→client only)
• Need separate channel for client→server
• Less efficient for bidirectional chat

Alternative: Single WebSocket server
Problems:
• Single point of failure
• Cannot scale beyond one machine
• All eggs in one basket
```

## Message Service

### Purpose
Handles message creation, storage, and retrieval.

### Data Structures

```
Message:
    message_id: snowflake_id          // Globally unique, time-ordered
    conversation_id: string
    sender_id: string
    content: encrypted_bytes
    content_type: enum(text, image, video, ...)
    media_id: string (optional)
    reply_to: message_id (optional)
    sequence: int64                   // Per-conversation sequence number
    created_at: timestamp
    updated_at: timestamp
    status: enum(active, edited, deleted)
    
ConversationSequence:
    conversation_id: string
    next_sequence: atomic_int64
```

### Sequence Number Assignment

```
// Ensures message ordering within conversation
FUNCTION assign_sequence(conversation_id):
    // Atomic increment
    sequence = INCR conversation_sequences:{conversation_id}
    RETURN sequence

// Why this matters:
// - Messages might arrive at different servers
// - Network delays might reorder messages
// - Sequence provides total order within conversation
```

### Message Storage Strategy

```
WRITE PATH:
1. Validate message
2. Assign sequence number (atomic)
3. Write to primary message store (sync)
4. Replicate to secondary (async)
5. ACK to sender
6. Enqueue for delivery (async)

READ PATH:
1. Query by (conversation_id, sequence_range)
2. Cache recent messages per conversation
3. Return in sequence order

INDEXES:
• Primary: (conversation_id, sequence) → message
• Secondary: (user_id, created_at) → message (for inbox)
• Media: (media_id) → message (for media access control)
```

### Failure Behavior

```
WRITE FAILURE:
• Retry with idempotency key (client-generated message_id)
• Duplicate writes detected and deduplicated
• Client shows "retry" if all retries fail

READ FAILURE:
• Serve from cache if available
• Return partial results with "more available" flag
• Client can retry failed ranges
```

## Delivery Service

### Purpose
Routes messages to recipients, handling online/offline cases.

### Data Structures

```
DeliveryTask:
    message_id: string
    conversation_id: string
    recipients: List<RecipientStatus>
    created_at: timestamp
    attempts: int
    next_retry_at: timestamp
    
RecipientStatus:
    user_id: string
    status: enum(pending, delivered, read, failed)
    delivered_at: timestamp (optional)
    device_deliveries: Map<device_id, DeliveryStatus>
```

### Delivery Algorithm

```
FUNCTION deliver_message(message):
    recipients = get_conversation_members(message.conversation_id)
    recipients.remove(message.sender_id)  // Don't deliver to sender
    
    FOR recipient IN recipients:
        // Check blocks
        IF is_blocked(recipient, message.sender_id):
            CONTINUE
        
        // Check online status
        IF is_online(recipient):
            TRY:
                push_via_websocket(recipient, message)
                mark_delivered(message.id, recipient)
            CATCH:
                enqueue_for_retry(message.id, recipient)
        ELSE:
            // Offline: send push notification
            send_push_notification(recipient, message)
            // Message will be fetched on next sync
            
    RETURN delivery_status
```

### Retry Strategy

```
RETRY POLICY:
• Immediate retry: 1 attempt
• After 1 second: 2nd attempt
• After 5 seconds: 3rd attempt
• After 30 seconds: 4th attempt
• After 5 minutes: 5th attempt
• After 1 hour: mark as "will deliver on sync"

RETRY QUEUE:
• Priority queue ordered by next_retry_at
• Separate queues per priority (high: 1:1, low: large groups)
• Dead letter queue for permanent failures
```

## Presence Service

### Purpose
Tracks user online/offline status with acceptable staleness.

### Design Philosophy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PRESENCE ANTI-PATTERNS (What NOT to do)                                   │
│                                                                             │
│   DON'T: Synchronous presence check on every message                        │
│   WHY: Adds latency to critical path                                        │
│                                                                             │
│   DON'T: Broadcast presence changes to all contacts                         │
│   WHY: O(N²) problem—1M users × 100 contacts = 100M updates                 │
│                                                                             │
│   DON'T: Store presence in main database                                    │
│   WHY: Too high write volume, kills database                                │
│                                                                             │
│   DO: Lazy presence with subscription model                                 │
│   DO: Accept 30-60 second staleness                                         │
│   DO: Use in-memory store with TTL                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Structures

```
PresenceEntry:
    user_id: string
    status: enum(online, away, offline)
    last_seen: timestamp
    ttl: 120 seconds  // Auto-expire if not refreshed

PresenceSubscription:
    subscriber_id: string
    subscribed_to: Set<user_id>
    // Limited to ~100 subscriptions per user
```

### Presence Update Flow

```
USER COMES ONLINE:
1. WebSocket Gateway notifies Presence Service
2. Update: presence:{user_id} = {status: online, last_seen: now}
3. Set TTL: 120 seconds
4. Find subscribers watching this user
5. Push presence update to subscribers (async, best-effort)

HEARTBEAT REFRESH:
1. Every 60 seconds, client activity refreshes presence
2. Update last_seen, reset TTL
3. No broadcast (status unchanged)

USER GOES OFFLINE:
1. Explicit: Client sends "going offline" → immediate update
2. Implicit: TTL expires (120 seconds no refresh)
3. Status becomes "offline" or "last seen X ago"
```

### Why This Design

```
PROBLEM: Naive presence is O(N²)
• 100M online users
• Average 100 contacts each
• Presence change → notify 100 people
• 100M users × 100 contacts = 10B presence events

SOLUTION: Subscription with limits
• Only track presence for active conversations
• Limit subscriptions to ~100 users
• Only push to users with open app
• Reduces events by 1000x

ACCEPTABLE STALENESS:
• "Online" might be 30-60 seconds stale
• "Last seen 2 hours ago" is fine for messaging
• Typing indicators fill the real-time gap
```

---

# Part 7: Data Model & Storage Decisions

## Core Entities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA MODEL                                               │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  USERS                                                              │   │
│   │  ├── user_id (PK)                                                   │   │
│   │  ├── phone_number / email (unique, indexed)                         │   │
│   │  ├── display_name                                                   │   │
│   │  ├── avatar_url                                                     │   │
│   │  ├── created_at                                                     │   │
│   │  └── settings (JSON: privacy, notifications, etc.)                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CONVERSATIONS                                                      │   │
│   │  ├── conversation_id (PK)                                           │   │
│   │  ├── type (1:1 | group)                                             │   │
│   │  ├── name (for groups)                                              │   │
│   │  ├── created_at                                                     │   │
│   │  ├── created_by                                                     │   │
│   │  └── metadata (JSON: settings, last_activity, etc.)                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CONVERSATION_MEMBERS                                               │   │
│   │  ├── conversation_id (PK)                                           │   │
│   │  ├── user_id (PK)                                                   │   │
│   │  ├── role (member | admin)                                          │   │
│   │  ├── joined_at                                                      │   │
│   │  ├── last_read_sequence (for unread count)                          │   │
│   │  └── muted_until                                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  MESSAGES                                                           │   │
│   │  ├── message_id (PK, snowflake)                                     │   │
│   │  ├── conversation_id (indexed)                                      │   │
│   │  ├── sequence (per-conversation order)                              │   │
│   │  ├── sender_id                                                      │   │
│   │  ├── content (encrypted)                                            │   │
│   │  ├── content_type                                                   │   │
│   │  ├── media_id (FK to media)                                         │   │
│   │  ├── reply_to (FK to message)                                       │   │
│   │  ├── created_at                                                     │   │
│   │  └── status (active | edited | deleted)                             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  USER_INBOX (Denormalized for sync performance)                     │   │
│   │  ├── user_id (partition key)                                        │   │
│   │  ├── message_id (sort key)                                          │   │
│   │  ├── conversation_id                                                │   │
│   │  └── created_at                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Storage Technology Choices

```
MESSAGE STORE: Distributed Wide-Column Store (Cassandra/Bigtable)
WHY:
• Handles massive write throughput (10M/sec)
• Excellent for time-series data (messages ordered by time)
• Good partition strategy: conversation_id
• Easy horizontal scaling
• Tunable consistency (per-query)

USER/CONVERSATION METADATA: Relational Database (Sharded MySQL/PostgreSQL)
WHY:
• Complex queries (find conversation by members)
• Strong consistency needed for membership
• Relatively low write volume
• ACID transactions for group operations

PRESENCE: In-Memory Store (Redis Cluster)
WHY:
• Extremely fast reads/writes
• Built-in TTL for auto-expiry
• Pub/sub for presence updates
• Ephemeral data (can lose and rebuild)

MEDIA: Object Storage (S3/GCS)
WHY:
• Unlimited scale
• Cost-effective for large files
• Built-in CDN integration
• Lifecycle policies for archival

CACHE: Distributed Cache (Redis/Memcached)
WHY:
• Hot conversation metadata
• Recent message cache
• User profile cache
• Session/auth token cache
```

## Partitioning Strategy

```
MESSAGES:
Partition key: conversation_id
Sort key: sequence

Why:
• All messages in conversation are co-located
• Reads are always within single partition
• Writes distributed across conversations
• Hot conversations don't affect others

POTENTIAL ISSUE: Hot conversation (celebrity with millions)
Solution: Sub-partition large conversations by time bucket

USER_INBOX:
Partition key: user_id
Sort key: message_id

Why:
• Sync query is always for single user
• Message_id is time-ordered (snowflake)
• Efficient range queries for "messages since X"
```

## Retention Policies

```
MESSAGE CONTENT:
• Hot storage (fast SSD): 30 days
• Warm storage (HDD): 1 year
• Cold storage (archive): 7 years (legal/compliance)
• User-deleted: Remove from all tiers

MEDIA:
• Original: 30 days
• Thumbnails: 1 year
• After expiry: Show "media expired" placeholder
• User can re-upload if they have local copy

METADATA:
• Conversation metadata: Forever (small)
• Message metadata: Same as content
• Delivery receipts: 30 days
• Read receipts: 7 days

PRESENCE/TYPING:
• No persistence (in-memory only)
• Reconstruct on service restart
```

## Schema Evolution

```
CHALLENGE: Billions of messages, can't do ALTER TABLE

STRATEGY 1: New columns are nullable with defaults
• Add new column with default value
• Old messages have default
• New messages have explicit value
• Backfill async if needed

STRATEGY 2: Schema versioning
• Each message has schema_version field
• Reader handles multiple versions
• Writer always uses latest version
• Gradual migration over time

STRATEGY 3: Flexible schema
• Content stored as JSON/protobuf
• Schema defined by content_type
• New types don't require DB changes

EXAMPLE: Adding reactions
V1: No reactions
V2: reactions: Map<emoji, count>
V3: reactions: List<{emoji, user_id, timestamp}>

Migration:
• V1 messages: reactions = null (no reactions existed)
• V2 messages: Migrate to V3 format on read
• V3 messages: Full data
```

## Why Other Data Models Were Rejected

```
REJECTED: Single MySQL for everything
Problems:
• Cannot handle 10M writes/sec
• Joins become bottleneck
• Sharding MySQL is complex
• No built-in time-series optimization

REJECTED: MongoDB for messages
Problems:
• Document model doesn't fit conversation pattern
• Ordering guarantees weaker
• Operational complexity at scale
• Better options exist for this workload

REJECTED: Storing messages in S3 directly
Problems:
• No efficient range queries
• No real-time writes
• High latency for recent messages
• Works for archive, not hot path
```

---

# Part 8: Consistency, Concurrency & Ordering

## The Core Ordering Challenge

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGE ORDERING: THE HARD PROBLEM                       │
│                                                                             │
│   SCENARIO: Alice and Bob messaging simultaneously                          │
│                                                                             │
│   Alice's phone           Network              Bob's phone                  │
│   10:00:00.001            ────────►            10:00:00.100                 │
│   "Hello"                                      "Hi there"                   │
│                           ◄────────                                         │
│   10:00:00.150                                 10:00:00.050                 │
│   "How are you?"                               "What's up?"                 │
│                                                                             │
│   PROBLEM: Whose "Hello" comes first?                                       │
│   • Alice's clock: Alice first (10:00:00.001 vs 10:00:00.100)               │
│   • Bob's clock: Bob first (10:00:00.050 vs 10:00:00.150)                   │
│   • Server receipt: Depends on network latency                              │
│                                                                             │
│   THERE IS NO "CORRECT" ANSWER                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Ordering Guarantees We Provide

```
GUARANTEE 1: Per-sender ordering (STRICT)
• Messages from same sender appear in send order
• Implemented via per-sender sequence numbers
• Client cannot send message N+1 until N is ACKed

GUARANTEE 2: Per-conversation causal ordering (STRONG)
• If Alice reads Bob's message then replies, reply comes after
• Implemented via Lamport-style logical clocks
• Each message carries "I have seen message X" metadata

GUARANTEE 3: Cross-sender ordering (BEST EFFORT)
• Use server-assigned timestamps as tiebreaker
• Within same millisecond: arbitrary but consistent
• Users rarely notice cross-sender ordering issues

NO GUARANTEE: Cross-conversation ordering
• Message to Group A and message to Friend B have no ordering
• Different conversations are independent
• This is fine—users don't compare across conversations
```

## Implementation: Logical Clocks

```
LAMPORT TIMESTAMP PER CONVERSATION:

Message:
    message_id: string
    logical_time: int64
    sender_sequence: int64
    causal_dependencies: List<message_id>

SEND MESSAGE:
1. Client tracks: max_seen_logical_time
2. new_logical_time = max_seen_logical_time + 1
3. Include: causal_dependencies = [last_message_I_saw]
4. Server validates: new_time > all dependency times
5. If conflict: Server assigns higher time

RECEIVE MESSAGE:
1. Update: max_seen_logical_time = max(current, received)
2. Sort messages by logical_time
3. If gap in sequence: Wait or fetch missing

CONFLICT RESOLUTION:
Same logical time? Use (logical_time, sender_id) as total order
Sender_id comparison is arbitrary but consistent
```

## Race Conditions

### Race 1: Simultaneous Send to Same Conversation

```
Alice and Bob both send at T=0

Server A receives Alice's message
Server B receives Bob's message

WITHOUT COORDINATION:
• Both assigned sequence=1
• Conflict when syncing

SOLUTION: Centralized sequence assignment
• Conversation sequence stored in Redis
• INCR is atomic
• Both servers get unique sequence
• May have gaps if one fails, but ordering preserved
```

### Race 2: Edit During Delivery

```
Alice sends "Hello"
Message in transit to Bob
Alice edits to "Hello!"
Edit arrives before original

Bob's view:
1. Receives edit for message he doesn't have
2. What to do?

SOLUTION: Idempotent operations with version
• Each message has version number
• Edit: version++ with full content
• Client applies edits to latest known version
• If base message arrives after edit: Apply edit
```

### Race 3: Group Membership Change During Send

```
Alice sends to group of 10 members
During fanout, Carol is removed from group

Question: Should Carol receive the message?

SOLUTION: Snapshot membership at send time
• When message stored, snapshot member list
• Fanout uses snapshot, not current membership
• Carol receives (was member at send time)
• Future messages: Carol not included

Alternative: Use current membership
• Carol doesn't receive
• But this creates strange gaps in Carol's history
• Staff choice: Snapshot (more intuitive for users)
```

## Idempotency

```
CLIENT-GENERATED MESSAGE ID:

PROBLEM: 
Client sends message, network fails before ACK
Client retries—is this a duplicate or new message?

SOLUTION:
Client generates unique message_id before first attempt
Server uses message_id for deduplication

FUNCTION receive_message(message):
    existing = get_message_by_id(message.id)
    
    IF existing:
        // Duplicate - return existing
        RETURN existing
    ELSE:
        // New message - store and process
        store(message)
        RETURN message

IDEMPOTENCY WINDOW:
• Keep message_id lookup for 24 hours
• After 24 hours: Unlikely duplicate (user would notice)
• Very late retry: Accept as new (rare edge case)
```

## Clock Assumptions

```
WHAT WE ASSUME:
• Server clocks are synchronized via NTP
• Server clock skew < 100ms (acceptable)
• Client clocks can be arbitrarily wrong
• Network latency is unbounded

IMPLICATIONS:
• Never trust client timestamps for ordering
• Client timestamp for display: "Sent at 10:42am"
• Server timestamp for ordering and sync
• Logical clocks for causal ordering

WHEN CLOCKS GO WRONG:
Client 1 year ahead:
• Server rejects timestamp, uses server time
• Message sorts correctly
• Display time might be wrong briefly

Server 1 hour behind:
• Messages from this server have old timestamps
• Sort among themselves correctly
• May sort before messages from other servers
• Self-heals when NTP corrects

STAFF INSIGHT:
Clock issues are inevitable at scale. Design
so that clock problems cause cosmetic issues
(wrong display time) not correctness issues
(wrong message order).
```

---

# Part 9: Failure Modes & Degradation

## Failure Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FAILURE SEVERITY LEVELS                                  │
│                                                                             │
│   LEVEL 1: INVISIBLE TO USERS                                               │
│   • Single server failure with quick failover                               │
│   • Cache miss (falls back to database)                                     │
│   • Replica lag (reads slightly stale data)                                 │
│                                                                             │
│   LEVEL 2: COSMETIC DEGRADATION                                             │
│   • Typing indicators stop working                                          │
│   • Presence shows stale data                                               │
│   • Read receipts delayed                                                   │
│   • Push notifications delayed                                              │
│                                                                             │
│   LEVEL 3: NOTICEABLE DEGRADATION                                           │
│   • Message delivery delayed (seconds to minutes)                           │
│   • Slow conversation loading                                               │
│   • Media upload/download failing                                           │
│                                                                             │
│   LEVEL 4: SIGNIFICANT IMPACT                                               │
│   • Messages not delivering (but stored)                                    │
│   • Cannot send new messages                                                │
│   • Cannot load conversations                                               │
│                                                                             │
│   LEVEL 5: CATASTROPHIC                                                     │
│   • Message loss (data corruption)                                          │
│   • Messages delivered to wrong users                                       │
│   • Extended complete outage                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Partial Failures

### WebSocket Gateway Failure

```
FAILURE: 10% of WebSocket gateways crash

IMPACT:
• 10% of users disconnected
• Messages to those users queue up
• Reconnections storm to remaining servers

MITIGATION:
• Clients auto-reconnect with exponential backoff
• Load balancer removes dead servers
• Remaining servers handle increased connections
• Message queue drains when users reconnect

DEGRADATION:
• 10% of users: 10-30 second delay
• 90% of users: Unaffected
• No message loss

RECOVERY TIME: 30 seconds to 2 minutes
```

### Message Store Failure

```
FAILURE: Primary message store unavailable

IMPACT:
• Cannot write new messages
• Cannot read recent messages (not in cache)

MITIGATION:
• Switch to secondary replica (async, may be behind)
• Queue writes in memory/local disk
• Return "sending..." to client, complete async
• Serve cached conversations for reads

DEGRADATION:
• Writes: Delayed, client shows "sending..."
• Reads: Recent messages might be missing
• No data loss if primary recovers

RECOVERY TIME: Depends on primary repair
```

### Presence Service Failure

```
FAILURE: Presence service completely down

IMPACT:
• Online status not updating
• "Last seen" stuck at failure time
• Delivery routing guesses online/offline

MITIGATION:
• Cache last-known presence (stale)
• Assume everyone potentially online
• Send both push and WebSocket (wasteful but works)

DEGRADATION:
• Presence shows stale data
• Some unnecessary push notifications
• Core messaging unaffected

RECOVERY TIME: Immediate once service restarts
```

## Slow Dependencies

```
MESSAGE STORE SLOW (P99 > 1 second):

Detection:
• Latency metrics alert
• Queue depth growing
• Timeout rate increasing

Response:
• Shed non-critical operations (analytics events)
• Increase timeout for critical writes
• Serve reads from cache more aggressively
• Consider circuit breaker if persistent

PUSH NOTIFICATION SERVICE SLOW:

Detection:
• Push delivery latency increasing
• Push queue growing

Response:
• Push is non-blocking anyway
• Increase queue capacity
• Drop low-priority pushes (read receipts)
• Batch more aggressively
```

## Retry Storms

```
SCENARIO: Message store recovers after 5-minute outage

PROBLEM:
• 5 minutes of messages queued
• All messages try to deliver at once
• Store overwhelmed again

PREVENTION:
• Jitter in retry timing
• Gradual ramp-up after recovery
• Priority queue (1:1 before groups, recent before old)
• Rate limit writes per second

IMPLEMENTATION:
FUNCTION calculate_retry_delay(attempt, base_delay):
    delay = base_delay * (2 ^ attempt)
    jitter = random(0, delay * 0.1)
    RETURN delay + jitter

// Retry at 1s, 2s, 4s, 8s... with 10% jitter
// Spreads retries over time window
```

## Data Corruption

```
SCENARIO: Bug causes messages stored with wrong conversation_id

DETECTION:
• Users report seeing wrong messages
• Message count mismatch in conversations
• Anomaly detection on message patterns

IMMEDIATE RESPONSE:
1. Identify scope (how many messages affected)
2. Stop the bleeding (fix or rollback bad code)
3. Quarantine affected data
4. Communicate to affected users

RECOVERY:
• If recoverable: Repair conversation_id from metadata
• If not: Mark messages as "recovered" with caveats
• Worst case: Inform users of data loss

PREVENTION:
• Strong typing on conversation_id
• Foreign key constraints (where possible)
• Write-time validation
• Audit log for forensics
```

## Control-Plane Failures

```
CONFIG SERVICE DOWN:

Impact:
• Cannot update rate limits
• Cannot change feature flags
• Cannot modify routing rules

Mitigation:
• Servers cache last-known config
• Stale config is usually fine
• Critical configs have long TTL

GROUP MANAGEMENT SERVICE DOWN:

Impact:
• Cannot create new groups
• Cannot add/remove members
• Existing groups work fine

Mitigation:
• Queue group operations
• Notify users of delay
• Retry on recovery
```

## Failure Timeline Walkthrough

```
T+0:00 - Primary message database becomes unresponsive
│
T+0:05 - Health checks fail, alerts fire
│
T+0:10 - Automatic failover to secondary begins
│        • Writes queue in memory
│        • Reads served from cache
│        • Clients see "sending..." stuck
│
T+0:30 - Failover complete, secondary promoted
│        • New writes go to new primary
│        • Queued writes drain (FIFO)
│        • Some writes during transition may retry
│
T+1:00 - Queue drained, normal operation
│        • Some messages delayed by 30-60 seconds
│        • Order preserved within conversations
│
T+5:00 - Old primary recovered, becomes secondary
│        • Catches up via replication
│        • No user impact
│
POST-MORTEM:
• 60 seconds of elevated latency
• 0 messages lost
• 0.1% of users experienced noticeable delay
• Root cause: Disk I/O saturation from runaway query
```

## Graceful Degradation Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEGRADATION PRIORITY                                     │
│                                                                             │
│   PROTECT AT ALL COSTS:                                                     │
│   1. Message storage (never lose a message)                                 │
│   2. Message delivery (delay OK, lose not OK)                               │
│   3. Message ordering (within conversation)                                 │
│                                                                             │
│   DEGRADE FIRST (SHED LOAD):                                                │
│   1. Typing indicators (cosmetic)                                           │
│   2. Read receipts (cosmetic)                                               │
│   3. Presence updates (stale is fine)                                       │
│   4. Media thumbnails (show placeholder)                                    │
│   5. Search (can fail gracefully)                                           │
│                                                                             │
│   DEGRADATION TOGGLES:                                                      │
│   • disable_typing_indicators: true                                         │
│   • disable_read_receipts: true                                             │
│   • presence_staleness_ok: 300 (seconds)                                    │
│   • disable_media_preview: true                                             │
│   • message_delivery_best_effort: true                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 10: Performance Optimization & Hot Paths

## Critical Paths

```
PATH 1: SEND MESSAGE (Most critical)
User action → ACK visible

Target: < 100ms P50, < 300ms P95

Steps:
1. TLS termination: 5ms
2. Auth validation: 5ms (cached token)
3. Rate limit check: 1ms (local)
4. Message validation: 1ms
5. Sequence assignment: 5ms (Redis)
6. Message write: 30ms (database)
7. Response: 5ms
Total: ~52ms typical

OPTIMIZATIONS APPLIED:
• Auth token cached in gateway
• Rate limit state in local memory
• Async delivery (not in critical path)
• Connection reuse (no handshake per message)

PATH 2: RECEIVE MESSAGE (Second most critical)
Message stored → User sees it

Target: < 500ms P50

Steps:
1. Delivery service picks up: 10ms
2. Find user connections: 5ms
3. Route to WebSocket server: 10ms
4. Push to client: 20ms
5. Client renders: 50ms (client-side)
Total: ~95ms (excluding client)

PATH 3: LOAD CONVERSATION (User experience critical)
User opens chat → Messages visible

Target: < 200ms P50

Steps:
1. Auth + conversation access check: 10ms
2. Query recent messages: 30ms (cached)
3. Fetch member info: 10ms (cached)
4. Response serialization: 5ms
5. Client render: 100ms (client-side)
Total: ~55ms server-side
```

## Caching Strategies

```
CACHE LAYER 1: Client-side
• Full message history for recent conversations
• User profiles for contacts
• Conversation list with snippets
• Media thumbnails

Benefits: 0ms latency for cached data
Invalidation: Server push or pull on open

CACHE LAYER 2: API Gateway (CDN edge)
• Static assets (avatars, stickers)
• Public media
• Not used for messages (personalized)

CACHE LAYER 3: Service Cache (Redis)
Cached:
• Conversation metadata: 60 second TTL
• User profiles: 300 second TTL
• Recent messages per conversation: 60 second TTL
• Group membership: 60 second TTL

Not cached:
• Unread counts (changes too frequently)
• Presence (separate system)
• Auth tokens (security concern)

CACHE LAYER 4: Database Query Cache
• Common query patterns cached at DB level
• Automatic invalidation on write
```

## Cache Invalidation

```
CONVERSATION CACHE INVALIDATION:

Trigger events:
• New message → Invalidate "recent messages" cache
• Member change → Invalidate "membership" cache
• Settings change → Invalidate "metadata" cache

Strategy: Write-through with async invalidation
1. Write to database (synchronous)
2. Update cache (synchronous)
3. Broadcast invalidation to other cache nodes (async)

RACE CONDITION:
Read from stale cache while write in progress

Mitigation:
• Use cache-aside with short TTL
• Accept eventual consistency
• Critical reads bypass cache
```

## Precomputation vs Runtime Work

```
PRECOMPUTED:
• Unread count per conversation
  - Updated on every message write/read
  - Query: O(1) lookup
  - Without: O(messages) count at runtime

• Conversation list order (by last_message_time)
  - Updated on every message
  - Query: Sorted list, O(1)
  - Without: O(conversations) sort at runtime

• Group member list (denormalized)
  - Updated on membership change
  - Query: O(1) lookup
  - Without: O(members) join at runtime

COMPUTED AT RUNTIME:
• Search results (too variable to precompute)
• Media transcoding (done on upload, cached)
• Message formatting (done on client)

HYBRID:
• Read receipts aggregation
  - Individual receipts stored
  - Aggregated "3 people read" computed on query
  - Cached for 30 seconds
```

## Backpressure

```
WEBSOCKET GATEWAY BACKPRESSURE:

Monitor:
• Pending message queue per connection
• If queue > 1000 messages: Connection is slow

Response:
1. Stop queuing new messages for this connection
2. Drop cosmetic messages (typing, presence)
3. Keep message delivery in separate priority queue
4. If sustained: Close connection, force resync

MESSAGE STORE BACKPRESSURE:

Monitor:
• Write latency > threshold
• Queue depth growing

Response:
1. Start rejecting non-critical writes
2. Return "server busy" to clients (429)
3. Clients back off with exponential retry
4. Shed read traffic to replicas
```

## Load Shedding

```
PRIORITY LEVELS:
P0: Message send ACK (never shed)
P1: Message delivery (shed only in emergency)
P2: Message history reads (shed before delivery)
P3: Typing indicators (shed early)
P4: Analytics events (always shed first)

SHEDDING IMPLEMENTATION:

FUNCTION handle_request(request):
    priority = get_priority(request.type)
    load = get_current_load()
    
    IF load > CRITICAL_THRESHOLD:
        IF priority > P1:
            RETURN 503("Service Overloaded")
    
    IF load > HIGH_THRESHOLD:
        IF priority > P2:
            RETURN 503("Service Overloaded")
    
    // Process normally
    process(request)

HEADERS:
Retry-After: 5  // Tell client when to retry
X-Shed-Reason: load-shedding  // For debugging
```

## Optimizations NOT Done (and Why)

```
NOT DONE: Compress all messages
WHY: Most messages are tiny (< 100 bytes)
     Compression overhead > savings
     Only compress media

NOT DONE: Global deduplication of media
WHY: Privacy concerns (hash reveals same photo)
     Computation overhead
     Storage is cheap

NOT DONE: Predictive message prefetch
WHY: Hard to predict which conversation opens next
     Wastes bandwidth for wrong predictions
     Client cache sufficient

NOT DONE: Inline media in message payload
WHY: Delays message delivery for slow media
     Better UX: Show text immediately, load media async

NOT DONE: Real-time analytics
WHY: Hot path shouldn't feed analytics
     Sample or async pipeline instead
     Analytics can be delayed
```

---

# Part 11: Cost & Efficiency

## Major Cost Drivers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COST BREAKDOWN (at WhatsApp scale)                       │
│                                                                             │
│   1. MEDIA STORAGE (40% of cost)                                            │
│      • 30 PB/day new media                                                  │
│      • Hot storage: $0.023/GB/month                                         │
│      • 30 days hot: 900 PB × $0.023 = $20M/month                            │
│                                                                             │
│   2. BANDWIDTH (25% of cost)                                                │
│      • Media download: 100 PB/day outbound                                  │
│      • At $0.05/GB: $5M/day = $150M/month                                   │
│      • With CDN: ~$50M/month                                                │
│                                                                             │
│   3. COMPUTE (20% of cost)                                                  │
│      • WebSocket servers: 10,000 servers                                    │
│      • Message services: 5,000 servers                                      │
│      • Supporting services: 5,000 servers                                   │
│      • Total: ~20,000 servers                                               │
│                                                                             │
│   4. DATABASE (10% of cost)                                                 │
│      • Message store: Petabytes SSD                                         │
│      • User/conversation metadata: Terabytes                                │
│      • Managed database services                                            │
│                                                                             │
│   5. THIRD-PARTY SERVICES (5% of cost)                                      │
│      • Push notification (APNs/FCM)                                         │
│      • CDN                                                                  │
│      • SMS verification                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## How Cost Scales with Traffic

```
LINEAR SCALING:
• Message storage: Each message costs storage
• Bandwidth: Each media download costs bandwidth
• Push notifications: Each notification has marginal cost

SUB-LINEAR SCALING:
• Fixed infrastructure: Doesn't scale with messages
• Caching: More traffic = better cache hit rate
• Batch operations: Larger batches = lower per-item cost

SUPER-LINEAR SCALING (Watch out!):
• Group messaging: 1000-member group = 1000x fanout
• Presence: N users × M contacts = N×M updates
• Search: Larger index = slower queries
```

## Cost vs Reliability Trade-offs

```
TRADE-OFF 1: Replication Factor
• 3x replication: High durability, 3x storage cost
• 2x replication: Acceptable durability, 2x cost
• 1x replication: Risky, lowest cost
DECISION: 3x for messages (critical), 2x for media (can re-upload)

TRADE-OFF 2: Hot vs Cold Storage
• All hot: Fast access, expensive
• Tiered: Fast for recent, slow for old
• All cold: Cheap but slow
DECISION: Tier at 30 days (95% of reads are < 30 days)

TRADE-OFF 3: Push Notification Aggressiveness
• Push every message: Best UX, highest cost
• Batch pushes: Good UX, lower cost
• Push only for direct mentions: Okay UX, lowest cost
DECISION: Immediate for 1:1, batched for active groups

TRADE-OFF 4: Media Quality
• Original quality: Best UX, huge storage
• Aggressive compression: Lower quality, much smaller
• Resolution tiers: Balance
DECISION: Compress to "good enough" quality, offer original download
```

## What Over-Engineering Looks Like

```
OVER-ENGINEERING EXAMPLE 1:
"We need exactly-once delivery with global consensus"

Reality:
• At-least-once with client dedup is sufficient
• Global consensus adds 100ms+ latency
• Duplicates are rare and harmless
• Cost: 10x infrastructure for 0.01% edge case

OVER-ENGINEERING EXAMPLE 2:
"We need sub-10ms message delivery globally"

Reality:
• Physics limits: 100ms cross-continent
• Users don't perceive < 200ms differences
• Optimizing below 100ms requires edge compute
• Cost: 5x infrastructure for imperceptible improvement

OVER-ENGINEERING EXAMPLE 3:
"We need to store all messages forever in hot storage"

Reality:
• 99% of reads are last 7 days
• Users rarely access year-old messages
• Hot storage is 10x cost of cold
• Cost: 10x storage for 0.1% of reads
```

## Cost-Aware Redesign

```
REDESIGN: Media Storage Optimization

BEFORE:
• Store original + 3 thumbnail sizes
• All in hot storage
• 4x storage per image
• Cost: $80M/month

AFTER:
• Store original in hot for 7 days
• Generate thumbnails on-demand (cached)
• Move to cold after 7 days
• Regenerate thumbnails from cold if needed
• Cost: $30M/month

TRADE-OFF:
• Cold media access adds 500ms latency
• Users accept this for old photos
• Savings: $50M/month

REDESIGN: Presence Efficiency

BEFORE:
• Update presence every 10 seconds
• Broadcast to all contacts
• 100M users × 100 contacts × 6/min = 60B updates/min

AFTER:
• Update presence every 60 seconds
• Only broadcast to users with app open
• Lazy presence (query on conversation open)
• Updates: 500M/min (100x reduction)

TRADE-OFF:
• Presence up to 60 seconds stale
• Users don't notice
• Compute savings: 90%
```

---

# Part 12: Multi-Region & Global Considerations

## Data Locality

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA LOCALITY STRATEGY                                   │
│                                                                             │
│   PRINCIPLE: Keep conversation data close to participants                  │
│                                                                             │
│   USER DATA:                                                                │
│   • Stored in user's home region                                            │
│   • Determined by phone number prefix                                       │
│   • Can migrate if user moves (rare)                                        │
│                                                                             │
│   CONVERSATION DATA:                                                        │
│   • 1:1: Stored in region of user with most activity                       │
│   • Group: Stored in region with plurality of members                       │
│   • Cross-region conversation: Replicate to both regions                   │
│                                                                             │
│   EXAMPLE:                                                                  │
│   Alice (US) and Bob (US): Data in US                                       │
│   Alice (US) and Chen (Asia): Replicate US ↔ Asia                          │
│   Group with 8 US, 2 EU: Data in US, async replicate to EU                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Replication Strategies

```
MESSAGE REPLICATION:

SYNCHRONOUS (within region):
• Primary write + 2 replicas in same region
• Ensures durability before ACK
• Latency: +5-10ms

ASYNCHRONOUS (cross-region):
• After ACK, replicate to other regions
• Disaster recovery (not active use)
• Lag: 100ms - 5 seconds typical

USER DATA REPLICATION:
• Master in home region
• Read replicas in other regions
• Writes always go to home region

CONVERSATION METADATA:
• Sync replicated within region (strong consistency)
• Async replicated cross-region (eventual consistency)
```

## Traffic Routing

```
ROUTING LOGIC:

USER CONNECTS:
1. DNS returns nearest edge location
2. Edge handles WebSocket connection
3. Requests routed to user's home region

MESSAGE SEND:
1. Accept at any region (where user is connected)
2. Forward to conversation's home region
3. Store and assign sequence there
4. Fan out delivery from there

CROSS-REGION MESSAGE:
Alice (US) sends to Bob (EU):
1. Alice's client → US edge
2. US edge → US message service (store)
3. US → EU (async replication for Bob's home region)
4. Delivery service → Bob via EU edge

OPTIMIZATION:
If Bob is online at EU edge:
• Push message directly via EU
• Don't wait for replication
• Mark as "delivered but may not be in EU replica yet"
```

## Failure Across Regions

```
SINGLE REGION FAILURE:

Scenario: US-West completely down

Impact:
• Users whose home is US-West: Degraded service
• Users in other regions: Minor impact (some contacts unavailable)

Response:
1. DNS removes US-West from rotation
2. US-West users routed to US-East (higher latency)
3. US-East becomes primary for US-West data
4. Catch up from replicas
5. When US-West recovers: Sync back

Message handling:
• Messages to US-West users queue in other regions
• Messages from US-West users sent via reroute
• Some latency, no message loss

NETWORK PARTITION:

Scenario: US-West and US-East cannot communicate

Impact:
• Split-brain potential
• Users in each region only see users in their region

Response:
1. Each region operates independently
2. Cross-region messages queue
3. When partition heals: Merge messages
4. Conflict resolution: Later timestamp wins

Conflict example:
• During partition, Alice (US-West) and Bob (US-East) both delete same group
• Partition heals
• Resolution: Keep one delete, apply to both
```

## When Multi-Region Is NOT Worth It

```
SKIP MULTI-REGION IF:
• All users in one geography
• < 1M users (cost doesn't justify)
• Can tolerate 30-minute failover RTO
• No regulatory data residency requirements

MULTI-REGION COSTS:
• 2-3x infrastructure (replicate everything)
• 2-3x operational complexity
• Cross-region bandwidth charges
• More complex debugging

SIMPLER ALTERNATIVE:
• Single region with good backups
• Automated restore to new region
• RTO: 30 minutes to 4 hours
• Cost: Much lower

WHEN IT IS WORTH IT:
• Global user base
• < 1 minute RTO requirement
• Data residency requirements (EU data in EU)
• > 10M users
```

---

# Part 13: Security & Abuse Considerations

## Abuse Vectors

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ABUSE VECTORS AND MITIGATIONS                            │
│                                                                             │
│   SPAM:                                                                     │
│   • Mass messaging to strangers                                             │
│   • Group join spam                                                         │
│   • Media/link spam                                                         │
│   Mitigation:                                                               │
│   • Rate limit messages to non-contacts                                     │
│   • Rate limit group joins                                                  │
│   • Link/media scanning                                                     │
│   • ML-based spam detection                                                 │
│                                                                             │
│   HARASSMENT:                                                               │
│   • Repeated unwanted messages                                              │
│   • Account creation to bypass blocks                                       │
│   • Group invite harassment                                                 │
│   Mitigation:                                                               │
│   • Easy blocking                                                           │
│   • Phone number ban (not just account)                                     │
│   • Require mutual contact for DMs                                          │
│                                                                             │
│   ILLEGAL CONTENT:                                                          │
│   • CSAM                                                                    │
│   • Terrorism-related content                                               │
│   • Copyright infringement                                                  │
│   Mitigation:                                                               │
│   • PhotoDNA/hashing for known illegal content                              │
│   • AI detection for new content                                            │
│   • Human review pipeline                                                   │
│   • Law enforcement cooperation                                             │
│                                                                             │
│   FRAUD:                                                                    │
│   • Impersonation                                                           │
│   • Phishing links                                                          │
│   • Scam messages                                                           │
│   Mitigation:                                                               │
│   • Verified badges for businesses                                          │
│   • Link safety warnings                                                    │
│   • Report and review pipeline                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Rate Abuse

```
RATE LIMITS BY CATEGORY:

Messages to contacts: 100/minute
Messages to non-contacts: 10/minute
Group creations: 10/day
Group joins: 50/day
Media uploads: 50/day, 1GB/day
Profile changes: 10/day

ABUSE PATTERNS:

Pattern: Burst of messages to 1000 users
Detection: > 100 unique recipients in 1 hour
Response: Rate limit, flag for review

Pattern: Scripted message sending
Detection: Perfectly regular intervals
Response: CAPTCHA challenge, temporary block

Pattern: Bot creating groups
Detection: Groups with same pattern (name, avatar)
Response: Account suspension
```

## Data Exposure Risks

```
RISK 1: Message Leaked to Wrong Recipient
Cause: Bug in routing logic
Impact: Critical privacy violation
Mitigation:
• Access control check at multiple layers
• Conversation membership verified on every operation
• Audit logging of all message access
• Regular security testing

RISK 2: Metadata Exposure
Cause: Logs containing message content
Impact: Privacy violation, potential legal issues
Mitigation:
• Never log message content
• Minimal metadata in logs
• Log retention limits
• Access controls on logs

RISK 3: Insider Access
Cause: Employee accesses user messages
Impact: Trust violation, legal issues
Mitigation:
• End-to-end encryption (even staff can't read)
• Access audit logs
• Principle of least privilege
• Background checks

RISK 4: API Leaking Data
Cause: API returns more than needed
Impact: Data exposure
Mitigation:
• Minimal response payloads
• Authorization on every field
• Regular API audits
```

## Privilege Boundaries

```
ACCESS CONTROL MODEL:

USER PRIVILEGES:
• Read own messages
• Send to conversations they're member of
• Manage own profile
• Block other users

GROUP ADMIN PRIVILEGES:
• Add/remove members
• Change group settings
• Delete any message in group
• Promote other admins

SYSTEM PRIVILEGES:
• Never access message content
• Can access metadata for:
  - Abuse investigation (with approval)
  - Law enforcement (with legal process)
  - Technical debugging (anonymized)

PRINCIPLE OF LEAST PRIVILEGE:
• Engineers cannot access production user data
• Debugging uses synthetic data
• Access to user data requires approval chain
• All access logged and audited
```

## Why Perfect Security Is Impossible

```
FUNDAMENTAL TENSIONS:

1. Usability vs Security
   • Perfect security: 20-character password, 2FA every time
   • Users want: Quick access, "remember this device"
   • Compromise: Risk-based authentication

2. Privacy vs Abuse Prevention
   • Perfect privacy: E2EE, no server-side inspection
   • Abuse prevention: Need to scan for illegal content
   • Compromise: Client-side scanning, or accept some abuse

3. Availability vs Security
   • Perfect security: Offline verification of everything
   • Availability: Need to work when networks are spotty
   • Compromise: Cache auth tokens, accept staleness

4. Debugging vs Privacy
   • Perfect privacy: No logs, no tracing
   • Debugging: Need to understand failures
   • Compromise: Anonymized logs, minimal retention

STAFF INSIGHT:
Security is always a trade-off. The goal is not perfect
security but appropriate security for the threat model.
Messaging has high privacy expectations—lean toward
privacy when in doubt, accept some operational complexity.
```

---

# Part 14: Evolution Over Time

## V1: Naive Design

```
INITIAL DESIGN (Startup scale: 10K users)

Architecture:
• Single server
• MySQL database
• Long polling for real-time
• Local file storage for media

Data model:
• messages table (id, sender, recipient, content, timestamp)
• users table (id, name, phone)
• No conversations abstraction

Delivery:
• Poll database every 2 seconds for new messages
• Show immediately on poll
• No delivery receipts

What works at 10K users:
• Simple to understand and debug
• Single source of truth
• All features work

What's already straining:
• Long polling creates many connections
• Database scanned on every poll
• No offline message queue
```

## What Breaks First

```
AT 100K USERS:
• Long polling: Too many connections
• Database: Query per poll kills performance
• Media: Local disk running out

FIX:
• Migrate to WebSocket (reduce connections 10x)
• Add message index by recipient
• Move media to S3

AT 1M USERS:
• Single database: Write capacity maxed
• Single server: Cannot handle connections
• Message ordering: Race conditions appearing

FIX:
• Shard database by user_id
• Multiple WebSocket servers
• Add Redis for sequence numbers

AT 10M USERS:
• Cross-shard queries: Killing performance
• Group messaging: Fan-out overwhelming
• Presence: Every status change storms

FIX:
• Denormalize inbox (fan-out on write)
• Async group delivery with queues
• Presence becomes eventually consistent
```

## V2: Improved Design

```
ARCHITECTURE (Scale: 100M users)

Components:
• WebSocket gateway cluster (100 servers)
• Message service (50 servers)
• Delivery service (50 servers)
• Presence service (Redis cluster)
• Message store (Cassandra cluster)
• Media store (S3)
• Cache layer (Redis)

Improvements over V1:
• Proper WebSocket for real-time
• Async delivery pipeline
• Conversation abstraction
• Delivery receipts
• Presence (eventually consistent)
• Multi-region awareness

Still problematic:
• Large groups are slow
• Global ordering not guaranteed
• Sync for long-offline users is slow
• Hot conversations create hotspots
```

## Long-Term Stable Architecture

```
MATURE ARCHITECTURE (Scale: 1B+ users)

Core principles:
1. Separate hot and cold paths
2. Eventual consistency by default, strong where needed
3. Cell-based architecture for isolation
4. Graceful degradation baked in

Components evolved:
• Cells: Self-contained units (10M users each)
• Cross-cell routing for conversations spanning cells
• Tiered storage (hot/warm/cold)
• Sophisticated presence (subscription-based)
• E2E encryption (client-side)
• ML-based abuse detection

Operational maturity:
• Canary deployments
• Automatic rollback on error spike
• Chaos engineering
• Runbook automation
• 24/7 on-call with clear escalation

What's stable:
• Core message flow unchanged for years
• Data model rarely changes
• Most work is optimization, not redesign
```

## How Incidents Drive Redesign

```
INCIDENT 1: New Year's Eve Outage

What happened:
• 50x traffic spike at midnight (by timezone)
• Message queue backed up
• Users saw "sending..." for minutes
• Cascaded across regions

Root cause:
• Queue capacity sized for 10x, not 50x
• No backpressure to slow senders
• Retry storms made it worse

Redesign:
• Backpressure from queue to API
• Client-side queuing with local retry
• Queue capacity 100x average
• Graceful degradation (disable typing, etc.)

INCIDENT 2: Celebrity Account Storm

What happened:
• Celebrity posted controversy
• 10M users tried to message them
• All messages routed to one shard
• Shard overloaded, affected other users on shard

Root cause:
• Sharding by recipient concentrated hot users
• No rate limiting on inbound to public figures
• No isolation between accounts

Redesign:
• Special handling for high-volume recipients
• Inbound rate limits (queue excess)
• Better shard isolation
• "Fan-out on read" for celebrity inboxes
```

---

# Part 15: Alternatives & Explicit Rejections

## Alternative 1: Pure P2P (No Central Server)

```
APPROACH:
• Messages sent directly between devices
• No central server for routing
• Device-to-device encryption built-in

WHY IT SEEMS ATTRACTIVE:
• Perfect privacy (no server access)
• Lower infrastructure cost
• No central point of failure
• Simple mental model

WHY STAFF ENGINEERS REJECT IT:
• Offline delivery: How to reach offline users?
  - Need some always-on component
• NAT traversal: Devices behind firewalls
  - Need STUN/TURN servers anyway
• Multi-device sync: How to sync phone ↔ laptop?
  - Need central sync service
• Group messaging: N² connections don't scale
  - Need central relay
• Abuse prevention: Can't moderate P2P
  - Platform becomes spam haven

VERDICT: Pure P2P doesn't work for mass-market messaging.
         End up building central services anyway.
         Better to design for server-assisted from start.
```

## Alternative 2: Blockchain-Based Messaging

```
APPROACH:
• Messages stored on blockchain
• Decentralized, censorship-resistant
• Cryptocurrency for spam prevention

WHY IT SEEMS ATTRACTIVE:
• "Web3" buzzword appeal
• Censorship resistance
• User ownership of data
• Built-in payment rails

WHY STAFF ENGINEERS REJECT IT:
• Latency: Block time = message delivery time
  - Even fast chains: 1-15 seconds per block
  - Unacceptable for real-time messaging
• Cost: Pay per message (gas fees)
  - $0.01/message × 100B messages/day = $1B/day
• Privacy: Blockchain is public
  - Everyone can see message metadata
• Scale: Blockchains don't scale
  - Even optimistic: 10K TPS vs our 10M TPS
• Complexity: Wallets, keys, etc.
  - Mass market won't manage private keys

VERDICT: Blockchain adds cost, latency, and complexity
         with no real benefit for messaging use case.
         Censorship resistance not needed by 99% of users.
```

## Alternative 3: Actor Model (Erlang/Elixir Style)

```
APPROACH:
• Each user is an actor process
• Messages passed between actors
• Platform built on Erlang/BEAM VM

WHY IT SEEMS ATTRACTIVE:
• Natural fit for messaging domain
• Built-in fault tolerance
• Lightweight processes (millions per server)
• WhatsApp famously uses Erlang

WHY STAFF ENGINEERS ARE CAUTIOUS:
• Hiring: Erlang engineers are rare
  - Harder to scale engineering team
• Debugging: Actor systems are hard to trace
  - Distributed state is opaque
• Persistence: Still need external database
  - Actors are in-memory only
• Libraries: Ecosystem smaller than JVM/Go
  - May need to build more from scratch
• Migration: Hard to move from existing systems
  - Big-bang rewrite needed

NUANCED VERDICT:
• For greenfield: Actor model can work well
• For existing system: Migration cost too high
• Staff choice: Use actor model concepts, implement in Go/Java
  - Get benefits without platform commitment
```

---

# Part 16: Interview Calibration

## How Interviewers Probe This System

```
OPENER: "Design a messaging platform like WhatsApp"

EXPECTED FIRST QUESTIONS (L6 candidates ask these):
• "What's the expected scale? Users, messages per day?"
• "1:1 messaging, group messaging, or both?"
• "What are the latency expectations?"
• "Any regulatory/compliance requirements?"
• "Is E2E encryption in scope?"

RED FLAG FIRST QUESTIONS (L5 candidates ask these):
• "Should I use Kafka?" (jumping to implementation)
• "What database should I use?" (jumping to details)
• Starting to draw without any clarification

COMMON PROBES:
• "How do you ensure message ordering?"
• "What happens when a user is offline?"
• "How do you handle a 1000-person group?"
• "What if the database goes down?"
• "How do you prevent spam?"
• "Walk me through the delivery of a single message"
```

## Common L5 Mistakes

```
MISTAKE 1: Designing for Precision Instead of Scale
L5: "We use 2PC to ensure exactly-once delivery"
Problem: 2PC adds latency and complexity
Staff: "At-least-once with idempotent receivers is sufficient"

MISTAKE 2: Ignoring Offline Users
L5: Shows only WebSocket path
Problem: 40% of users are offline at any time
Staff: "Here's the offline queue and sync mechanism"

MISTAKE 3: Total Ordering
L5: "Single queue for all messages ensures ordering"
Problem: Single queue doesn't scale
Staff: "Per-conversation ordering with logical clocks"

MISTAKE 4: Underestimating Group Complexity
L5: "Send to each member like 1:1"
Problem: 1000-member group = 1000x fanout
Staff: "Different strategy based on group size"

MISTAKE 5: Treating All Messages Equal
L5: "All messages go through same path"
Problem: Media is 1000x larger than text
Staff: "Separate paths for text, media, signals"
```

## Staff-Level Answers

```
ON ORDERING:
"Message ordering is complex. Within a conversation, we need
per-sender ordering as a minimum—messages from Alice to Bob
should appear in the order Alice sent them. For cross-sender
ordering, we use Lamport timestamps to establish causality.
The key insight is that users only care about ordering within
a single conversation view—we don't need global ordering."

ON OFFLINE HANDLING:
"Offline users are actually the common case, not the exception.
When a message arrives for an offline user, we store it in their
inbox and send a push notification. On reconnect, the client
provides its last sync cursor, and we return all messages since
then. The tricky part is handling long-offline users—we paginate
and prioritize recent conversations."

ON LARGE GROUPS:
"Large groups require different fanout strategies. For a 10-person
group, fan-out on write works fine—store one message, deliver to
10 people. For a 10,000-person group, that's 10,000 deliveries
per message. Instead, we use fan-out on read: store the message
once, and each member fetches it when they open the conversation.
The threshold is around 50-100 members."

ON FAILURE:
"The non-negotiable is: never lose a message after ACK. Everything
else can degrade. During a partial outage, we'll continue to accept
and store messages—delivery might be delayed. We prioritize message
storage over delivery, and delivery over cosmetic features like
typing indicators. Users can handle a few seconds of delay; they
can't handle lost messages."
```

## Example Phrases Staff Engineers Use

```
SCOPE DEFINITION:
• "Let's define what 'delivered' means in this context..."
• "I'm going to explicitly exclude E2E encryption for now..."
• "The interesting trade-off here is between X and Y..."

TRADE-OFF ARTICULATION:
• "We could do strong consistency here, but the latency cost is..."
• "I'm choosing eventual consistency because..."
• "The risk of this approach is X, which we mitigate by Y..."

SHOWING DEPTH:
• "In my experience, what actually breaks first is..."
• "The non-obvious problem with that approach is..."
• "At first glance you might do X, but actually Y is better because..."

HANDLING UNCERTAINTY:
• "I'm not sure about the exact numbers, but order of magnitude..."
• "This would need benchmarking, but my intuition says..."
• "Let me reason through this—if we assume X, then..."

DRIVING DISCUSSION:
• "Let me walk you through the happy path first, then failures..."
• "I'll sketch the architecture, then deep-dive on [component]..."
• "Before I go further, should I elaborate on X or move to Y?"
```

---

# Part 17: Diagrams

## Diagram 1: Core Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGING PLATFORM ARCHITECTURE                          │
│                                                                             │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                          CLIENTS                                    │    │
│   │          iOS    Android    Web    Desktop    Tablet                │    │
│   └──────────────────────────┬─────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                       EDGE LAYER                                    │    │
│   │   ┌──────────┐    ┌──────────┐    ┌──────────┐                     │    │
│   │   │   API    │    │WebSocket │    │   Push   │                     │    │
│   │   │ Gateway  │    │ Gateway  │    │ Gateway  │                     │    │
│   │   │  (HTTP)  │    │  (WS)    │    │(APNs/FCM)│                     │    │
│   │   └──────────┘    └──────────┘    └──────────┘                     │    │
│   └──────────────────────────┬─────────────────────────────────────────┘    │
│                              │                                              │
│   ┌──────────────────────────┴─────────────────────────────────────────┐    │
│   │                      SERVICE LAYER                                  │    │
│   │                                                                     │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │    │
│   │  │ Message  │  │ Delivery │  │ Presence │  │  Sync    │            │    │
│   │  │ Service  │  │ Service  │  │ Service  │  │ Service  │            │    │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │    │
│   │                                                                     │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │    │
│   │  │  Group   │  │   User   │  │  Media   │                          │    │
│   │  │ Service  │  │ Service  │  │ Service  │                          │    │
│   │  └──────────┘  └──────────┘  └──────────┘                          │    │
│   └──────────────────────────┬─────────────────────────────────────────┘    │
│                              │                                              │
│   ┌──────────────────────────┴─────────────────────────────────────────┐    │
│   │                       DATA LAYER                                    │    │
│   │                                                                     │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │    │
│   │  │ Message  │  │   User   │  │  Cache   │  │  Media   │            │    │
│   │  │  Store   │  │  Store   │  │ (Redis)  │  │  (S3)    │            │    │
│   │  │(Cassandra│  │ (MySQL)  │  │          │  │          │            │    │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │    │
│   │                                                                     │    │
│   │  ┌──────────────────────────────────────────┐                      │    │
│   │  │           Message Queue (Kafka)           │                      │    │
│   │  │     delivery-tasks | notifications        │                      │    │
│   │  └──────────────────────────────────────────┘                      │    │
│   └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Diagram 2: Message Send Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGE SEND FLOW                                        │
│                                                                             │
│   ALICE (Sender)                                                            │
│   ─────────────                                                             │
│        │                                                                    │
│        │ 1. POST /messages                                                  │
│        │    {conversation_id, content}                                      │
│        ▼                                                                    │
│   ┌────────────┐                                                            │
│   │    API     │  2. Auth check                                             │
│   │  Gateway   │  3. Rate limit                                             │
│   └─────┬──────┘                                                            │
│         │                                                                   │
│         │ 4. Send to Message Service                                        │
│         ▼                                                                   │
│   ┌────────────┐                                                            │
│   │  Message   │  5. Validate message                                       │
│   │  Service   │  6. Get sequence number (Redis INCR)                       │
│   └─────┬──────┘  7. Store message (Cassandra)                              │
│         │                                                                   │
│         │ 8. Return message_id to Alice                                     │
│         │    (Alice sees ✓ sent)                                            │
│         │                                                                   │
│         │ 9. Enqueue delivery task                                          │
│         ▼                                                                   │
│   ┌────────────┐                                                            │
│   │  Delivery  │  10. Get conversation members                              │
│   │  Service   │  11. For each recipient:                                   │
│   └─────┬──────┘                                                            │
│         │                                                                   │
│    ┌────┴────────────────────────────────┐                                  │
│    │                                     │                                  │
│    ▼                                     ▼                                  │
│  ┌────────────┐                    ┌────────────┐                           │
│  │  Presence  │ 12. Is Bob online? │    Push    │                           │
│  │  Service   │                    │  Gateway   │                           │
│  └─────┬──────┘                    └─────┬──────┘                           │
│    YES │                                 │ NO                               │
│        ▼                                 ▼                                  │
│  ┌────────────┐                    ┌────────────┐                           │
│  │ WebSocket  │ 13. Push via WS    │   APNs/    │ 14. Push notification     │
│  │  Gateway   │                    │    FCM     │                           │
│  └─────┬──────┘                    └────────────┘                           │
│        │                                                                    │
│        │ 15. Bob receives message                                           │
│        ▼                                                                    │
│   BOB (Recipient)                                                           │
│   ───────────────                                                           │
│        │                                                                    │
│        │ 16. Send delivery ACK                                              │
│        ▼                                                                    │
│   ┌────────────┐                                                            │
│   │  Delivery  │ 17. Update status → DELIVERED                              │
│   │  Service   │ 18. Notify Alice (Alice sees ✓✓)                           │
│   └────────────┘                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Diagram 3: Failure Propagation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FAILURE PROPAGATION                                      │
│                                                                             │
│   SCENARIO: Message Store Becomes Slow (P99 > 2 seconds)                    │
│                                                                             │
│   T+0: Message Store latency increases                                      │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  MESSAGE SERVICE                                                    │   │
│   │  • Write latency: 30ms → 2000ms                                     │   │
│   │  • Thread pool saturates                                            │   │
│   │  • Requests queue up                                                │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│   T+30s: API Gateway affected    │                                          │
│         ┌────────────────────────┘                                          │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  API GATEWAY                                                        │   │
│   │  • Request timeout rate: 0.1% → 15%                                 │   │
│   │  • Client retries increase load                                     │   │
│   │  • Connection pool to Message Service exhausted                     │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│   T+60s: User impact visible     │                                          │
│         ┌────────────────────────┘                                          │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CLIENTS                                                            │   │
│   │  • Messages stuck at "sending..."                                   │   │
│   │  • Users retry manually (more load)                                 │   │
│   │  • Angry social media posts                                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ═══════════════════════════════════════════════════════════════════════   │
│   MITIGATION ACTIVATED                                                      │
│   ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│   T+2m: Circuit breaker triggers                                            │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  MESSAGE SERVICE                                                    │   │
│   │  • Circuit breaker OPEN                                             │   │
│   │  • Fast-fail new requests (no timeout wait)                         │   │
│   │  • Return 503 with Retry-After header                               │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│   T+3m: Backpressure to clients  │                                          │
│         ┌────────────────────────┘                                          │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CLIENTS                                                            │   │
│   │  • Queue messages locally                                           │   │
│   │  • Show "connection issues" banner                                  │   │
│   │  • Retry with exponential backoff                                   │   │
│   │  • Messages saved, will send when healthy                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   T+10m: Message Store recovers                                             │
│         │                                                                   │
│         ▼                                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  GRADUAL RECOVERY                                                   │   │
│   │  • Circuit breaker: Half-open (test requests)                       │   │
│   │  • Test requests succeed → Circuit CLOSED                           │   │
│   │  • Client queues drain gradually                                    │   │
│   │  • Full recovery in ~5 minutes                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   OUTCOME:                                                                  │
│   • 10-minute degradation, not outage                                       │
│   • Zero messages lost                                                      │   
│   • Messages delayed 5-15 minutes                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Diagram 4: System Evolution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM EVOLUTION                                         │
│                                                                             │
│   V1: MONOLITH (10K users)                                                  │
│   ────────────────────────────────────────────────────                      │
│   ┌──────────────────────────────────────────────────┐                      │
│   │  Single Server                                   │                      │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │                      │
│   │  │   App    │  │  MySQL   │  │  Files   │        │                      │
│   │  │  (PHP)   │  │  (1 DB)  │  │  (local) │        │                      │
│   │  └──────────┘  └──────────┘  └──────────┘        │                      │
│   └──────────────────────────────────────────────────┘                      │
│                                                                             │
│   ▼ Scaling pain: DB bottleneck, file storage full                          │
│                                                                             │
│   V2: SEPARATED TIERS (1M users)                                            │
│   ────────────────────────────────────────────────────                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│   │  API Servers │  │   Database   │  │    Media     │                      │
│   │    (x10)     │  │  (Primary +  │  │    (S3)      │                      │
│   │              │  │   Replica)   │  │              │                      │
│   └──────────────┘  └──────────────┘  └──────────────┘                      │
│                                                                             │
│   ▼ Scaling pain: Long polling, real-time delays                            │
│                                                                             │
│   V3: REAL-TIME ADDED (10M users)                                           │
│   ────────────────────────────────────────────────────                      │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────┐                       │
│   │   API    │  │WebSocket │  │ Database │  │ Redis │                       │
│   │ Servers  │  │ Servers  │  │ (Sharded)│  │ Cache │                       │
│   │  (x50)   │  │  (x20)   │  │          │  │       │                       │
│   └──────────┘  └──────────┘  └──────────┘  └───────┘                       │
│                                                                             │
│   ▼ Scaling pain: Group fanout, presence storms                             │
│                                                                             │
│   V4: MICROSERVICES (100M users)                                            │
│   ────────────────────────────────────────────────────                      │
│   ┌──────────────────────────────────────────────────────────────┐          │
│   │                        Services                               │          │
│   │  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ │          │
│   │  │Message│ │Delivery│ │Presence│ │ Sync │ │ Group │ │ User │ │          │
│   │  └───────┘ └───────┘ └───────┘ └───────┘ └───────┘ └───────┘ │          │
│   └──────────────────────────────────────────────────────────────┘          │
│   ┌──────────────────────────────────────────────────────────────┐          │
│   │                     Data Stores                               │          │
│   │  ┌─────────┐  ┌────────┐  ┌───────┐  ┌───────┐  ┌───────────┐│          │
│   │  │Cassandra│  │ MySQL  │  │ Redis │  │  S3   │  │   Kafka   ││          │
│   │  │(messages│  │(users) │  │(cache)│  │(media)│  │  (queue)  ││          │
│   │  └─────────┘  └────────┘  └───────┘  └───────┘  └───────────┘│          │
│   └──────────────────────────────────────────────────────────────┘          │
│                                                                             │
│   ▼ Scaling pain: Regional latency, compliance requirements                 │
│                                                                             │
│   V5: MULTI-REGION CELLS (1B+ users)                                        │
│   ────────────────────────────────────────────────────                      │
│   ┌────────────────────┐  ┌────────────────────┐  ┌──────────────────┐      │
│   │      US Cell       │  │      EU Cell       │  │    APAC Cell     │      │
│   │  ┌──────────────┐  │  │  ┌──────────────┐  │  │ ┌──────────────┐ │      │
│   │  │Full service  │  │  │  │Full service  │  │  │ │Full service  │ │      │
│   │  │   stack      │  │  │  │   stack      │  │  │ │   stack      │ │      │
│   │  └──────────────┘  │  │  └──────────────┘  │  │ └──────────────┘ │      │
│   │  ┌──────────────┐  │  │  ┌──────────────┐  │  │ ┌──────────────┐ │      │
│   │  │ Full data    │  │  │  │ Full data    │  │  │ │ Full data    │ │      │
│   │  │   replicas   │◄─┼──┼──┼──► replicas  │◄─┼──┼─┼──►replicas   │ │      │
│   │  └──────────────┘  │  │  └──────────────┘  │  │ └──────────────┘ │      │
│   └────────────────────┘  └────────────────────┘  └──────────────────┘      │
│              ▲                      ▲                      ▲                │
│              └──────────── Cross-region replication ───────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 18: Brainstorming, Exercises & Redesigns

## "What If X Changes?" Questions

```
WHAT IF: Message volume 10x overnight (viral event)

Immediate impact:
• Message queues back up
• Database write capacity exceeded
• Push notification system overwhelmed

Short-term response:
• Shed non-critical features (typing, presence)
• Rate limit less-critical paths
• Scale horizontally (pre-provisioned capacity)

Long-term change:
• More aggressive auto-scaling
• Higher base capacity
• Better traffic prediction

WHAT IF: Average message size 100x (voice messages become primary)

Impact:
• Storage costs explode
• Bandwidth costs explode
• Upload/download times increase

Response:
• Aggressive compression (opus codec)
• Streaming upload/download
• Shorter retention for voice
• Separate voice message tier

WHAT IF: Regulatory requirement for message retention

Impact:
• Can't delete user messages on request
• Need audit trail for access
• Legal hold functionality needed

Response:
• Parallel compliance data store
• Access logging and approval workflow
• Separate retained vs user-visible state
• Legal review before deletion

WHAT IF: E2E encryption becomes mandatory

Impact:
• Cannot scan for abuse on server
• Cannot recover messages if user loses key
• More complex key management

Response:
• Client-side content scanning (controversial)
• Key escrow options (with user consent)
• Robust key backup mechanisms
• Accept reduced abuse detection capability
```

## Redesign Under New Constraints

```
CONSTRAINT: Zero trust network (all internal traffic encrypted)

Original: Services communicate over private network
New: Every service-to-service call authenticated + encrypted

Changes needed:
• mTLS between all services
• Service identity certificates
• Certificate rotation automation
• ~5ms latency overhead per hop

CONSTRAINT: Data must stay in user's home country

Original: Global replication, route anywhere
New: Messages stored only in user's country

Changes needed:
• Per-country data stores
• Routing based on sender country
• Cross-country conversations: ???
  - Option A: Store in sender's country (recipient must query cross-border)
  - Option B: Store copy in both countries (compliance risk?)
  - Option C: Store in neutral zone (latency, complexity)
• Significantly more operational complexity

CONSTRAINT: Maximum 10ms P99 message send latency

Original: 100ms P50, 300ms P95
New: 10ms P99

Changes needed:
• No synchronous database write (use async + local ACK)
• Edge message acceptance
• Eventual consistency everywhere
• Risk: Can lose messages if edge fails before sync

Trade-off analysis:
• 10ms P99 is unrealistic for durable messaging
• Would need to sacrifice durability guarantee
• Better: Accept 50ms P99, which is achievable with durability
```

## Failure Injection Exercises

```
EXERCISE 1: Chaos Monkey for WebSocket Gateways

Inject: Kill random WebSocket gateway every hour

Expected behavior:
• Affected users disconnect (10 seconds no messages)
• Clients auto-reconnect to different gateway
• Sync service provides missed messages
• No user-visible message loss

What to monitor:
• Reconnection success rate
• Time to reconnect
• Message loss/duplication
• User complaints

EXERCISE 2: Slow Message Store

Inject: Add 500ms latency to 10% of message store writes

Expected behavior:
• P50 latency mostly unaffected
• P90 increases noticeably
• Circuit breaker should not trigger (only 10%)
• No message loss

What to monitor:
• Latency percentiles
• Timeout rates
• User retry behavior
• Queue depths

EXERCISE 3: Push Notification Failure

Inject: Block all push notifications for 30 minutes

Expected behavior:
• Offline users don't get notified
• Online users (WebSocket) unaffected
• When push recovers, backlog clears
• Users opening app get all messages (sync)

What to monitor:
• Push delivery rate
• App open rate (users check manually?)
• Message delivery lag
• User complaints
```

## Trade-off Debates

```
DEBATE 1: Fanout on Write vs Fanout on Read

For Fanout on Write:
• Fast reads (message already in recipient's inbox)
• Simple read path
• Good for mostly 1:1 messaging

For Fanout on Read:
• Efficient writes (store once)
• Better for large groups
• Easier to handle membership changes

Staff position:
• Hybrid: FOW for small groups (<50), FOR for large groups
• 1:1 is essentially FOW (store in both inboxes)
• Threshold tuned based on actual usage patterns

DEBATE 2: Strong vs Eventual Consistency for Membership

For Strong:
• Immediate effect of blocks/bans
• No leaked messages to removed members
• Simpler mental model

For Eventual:
• Better availability
• Lower latency
• Partition tolerance

Staff position:
• Strong consistency for membership within region
• Accept eventual consistency cross-region
• Err on side of blocking (if unsure if blocked, assume blocked)

DEBATE 3: Client-Generated vs Server-Generated Message IDs

For Client-Generated:
• Idempotency built in (retry with same ID)
• Works offline (generate ID without server)
• Matches client's mental model

For Server-Generated:
• Guaranteed unique (no collision risk)
• Server controls ordering
• Simpler client

Staff position:
• Client generates ID (UUID or similar)
• Server validates uniqueness
• Benefits of idempotency outweigh collision risk
• Collision probability with UUIDv4: negligible
```

---

# Summary

Messaging platforms are deceptively complex systems. The core challenge isn't sending bytes between devices—it's maintaining ordering guarantees, handling offline users gracefully, and scaling fan-out without melting infrastructure.

**Key Staff-Level Insights:**

1. **Message delivery is sacred.** Everything else—typing indicators, presence, read receipts—can degrade. Messages must never be lost after ACK.

2. **Ordering is conversation-local.** Don't try to achieve global ordering—it doesn't scale. Per-sender ordering with Lamport timestamps is sufficient.

3. **Offline is the common case.** Design the offline path first, then optimize the online path. Sync must work for users offline for days.

4. **Large groups are different.** Fan-out strategies must change based on group size. What works for 10 members fails at 10,000.

5. **Presence is eventually consistent.** Don't make presence a scaling bottleneck. Users tolerate 30-60 seconds of staleness.

6. **The client is smart.** Client-side caching, local storage, and queuing reduce server load and improve UX during failures.

**The Staff Engineer Difference:**

An L5 might design a working messaging system. An L6 designs a messaging system that gracefully degrades during failures, scales sub-linearly with traffic, handles the edge cases that break naive implementations, and evolves without requiring rewrites. The difference is in the depth of understanding—not just how messages flow, but what happens when they don't.