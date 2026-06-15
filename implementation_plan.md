# TechTalk — Complete Implementation Plan

> A real-time chat platform for tech enthusiasts with moderation, ML-powered content filtering, and cloud deployment.

---

## Table of Contents

1. [What Are We Building?](#1-what-are-we-building)
2. [System Architecture — The Big Picture](#2-system-architecture--the-big-picture)
3. [Feature Breakdown — Line by Line](#3-feature-breakdown--line-by-line)
4. [Database Schema Design](#4-database-schema-design)
5. [Backend Architecture (Spring Boot)](#5-backend-architecture-spring-boot)
6. [WebSocket Protocol Design](#6-websocket-protocol-design)
7. [REST API Design](#7-rest-api-design)
8. [Frontend Architecture (React)](#8-frontend-architecture-react)
9. [ML Microservice (Python FastAPI)](#9-ml-microservice-python-fastapi)
10. [Security Architecture](#10-security-architecture)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Phased Execution Roadmap](#12-phased-execution-roadmap)

---

## 1. What Are We Building?

**TechTalk** is a real-time chat application built specifically for developers and tech enthusiasts. Think of it as a focused version of Discord/Slack — but dedicated to technology discussions with built-in smart moderation.

### What makes it different from a regular chat app?

| Regular Chat App | TechTalk |
|---|---|
| Anyone can spam anything | ML model detects spam/toxicity in real-time |
| Manual moderation only | Auto-block after 3 reports + AI flagging |
| Generic rooms | Pre-built tech rooms: #ai-ml, #web-dev, #devops, etc. |
| Plain text messages | Code snippet sharing with syntax highlighting |
| No topic awareness | ML classifies messages into tech categories |
| No accountability | Full report → review → warn → mute → ban pipeline |

### The Core Problem We Solve

In open tech communities, conversations degrade fast — spam, off-topic messages, harassment. TechTalk solves this with a **three-layer defense**:

1. **Layer 1 — ML Filter**: Every message passes through a toxicity model before delivery. High-score messages are auto-flagged.
2. **Layer 2 — Community Reports**: Users can report messages. 3+ reports on a user triggers auto-block.
3. **Layer 3 — Moderator Dashboard**: Human moderators review flagged/reported content and take action (warn, mute, ban).

---

## 2. System Architecture — The Big Picture

### 2.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USERS (Browsers)                             │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│   │   User A      │  │   User B      │  │  Moderator   │             │
│   │  (React App)  │  │  (React App)  │  │ (React App)  │             │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│          │                  │                  │                     │
└──────────┼──────────────────┼──────────────────┼────────────────────┘
           │ HTTPS + WSS      │                  │
           ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    CLOUDFLARE PAGES (CDN)                            │
│              Serves React static files globally                      │
│              Mumbai PoP for Indian users (~20ms)                     │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ API Calls (HTTPS + WebSocket)
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  GCP CLOUD RUN — asia-south1 (Mumbai)                │
│                                                                      │
│  ┌─────────────────────────┐    ┌──────────────────────────┐        │
│  │   SPRING BOOT BACKEND   │    │   PYTHON ML SERVICE      │        │
│  │                         │    │                          │        │
│  │  • REST API Controllers │◄──►│  • Toxicity Detection    │        │
│  │  • WebSocket (STOMP)    │    │  • Topic Classification  │        │
│  │  • JWT Auth             │    │  • FastAPI + HuggingFace │        │
│  │  • Business Logic       │    │                          │        │
│  │  • Report/Block Engine  │    └──────────────────────────┘        │
│  └────────┬────────┬───────┘                                        │
│           │        │                                                 │
└───────────┼────────┼─────────────────────────────────────────────────┘
            │        │
     ┌──────┘        └──────┐
     ▼                      ▼
┌──────────────┐    ┌──────────────┐
│  GCP CLOUD   │    │ REDIS CLOUD  │
│  SQL (MySQL) │    │  (Mumbai)    │
│              │    │              │
│  • Users     │    │  • Online    │
│  • Messages  │    │    Status    │
│  • Rooms     │    │  • Session   │
│  • Reports   │    │    Cache     │
│  • Blocks    │    │  • Typing    │
│              │    │    Events    │
│  asia-south1 │    │  (Free tier) │
└──────────────┘    └──────────────┘
```

### 2.2 How Data Flows — A Message's Journey

Let's trace what happens when **User A sends "Check out this Python trick!" in the #web-dev room**:

```
Step 1: User A types message and hits Send
         ↓
Step 2: React app sends message via WebSocket (STOMP protocol)
         Destination: /app/chat.room.web-dev
         ↓
Step 3: Spring Boot ChatController receives the message
         ↓
Step 4: Spring Boot calls ML Service (HTTP POST to /analyze)
         Request body: { "text": "Check out this Python trick!" }
         ↓
Step 5: ML Service returns:
         {
           "toxicity_score": 0.02,     ← Safe (below 0.7 threshold)
           "topic": "web-dev",         ← Correctly classified
           "is_flagged": false
         }
         ↓
Step 6: Spring Boot saves message to MySQL with ML scores
         INSERT INTO messages (sender_id, room_id, content,
                               toxicity_score, topic, ...)
         ↓
Step 7: Spring Boot broadcasts to ALL subscribers of /topic/room.web-dev
         ↓
Step 8: Every user subscribed to #web-dev receives the message instantly
         Their React app renders the new MessageBubble component
```

Now let's trace a **TOXIC message** — User B sends "You're an idiot, get lost":

```
Steps 1-4: Same as above
         ↓
Step 5: ML Service returns:
         {
           "toxicity_score": 0.92,     ← DANGEROUS (above 0.7)
           "topic": "off-topic",
           "is_flagged": true           ← AUTO-FLAGGED
         }
         ↓
Step 6: Spring Boot saves message with is_flagged = true
         Also creates an entry in the moderation_queue table
         ↓
Step 7: Message is STILL delivered (we don't silently censor)
         BUT it's tagged with a warning indicator
         ↓
Step 8: Moderators see a real-time alert on their dashboard
         They can then: approve, delete, warn user, or ban user
```

### 2.3 Why This Architecture?

| Decision | Why |
|---|---|
| **Separate ML Service** | Python has the best ML libraries (HuggingFace, scikit-learn). Java doesn't. Keeping them separate means you can update the ML model without redeploying the entire backend. |
| **WebSocket for chat** | HTTP requires the client to keep asking "any new messages?" (polling). WebSocket keeps a persistent connection — the server **pushes** messages instantly. For a chat app, this is mandatory. |
| **Redis for presence** | "Online/Offline" status changes every few seconds. Writing that to MySQL would destroy database performance. Redis stores it in memory — reads in <1ms. |
| **MySQL for persistence** | Messages, users, reports are permanent data with complex relationships (a message belongs to a user AND a room, a report links 3 entities). Relational databases handle this perfectly. |
| **Cloud Run (not VM)** | Scales to zero when nobody's using TechTalk = ₹0 cost. A VM runs 24/7 eating credits even while idle. |
| **Cloudflare Pages** | Free, globally distributed CDN. Your React files load from Mumbai for Indian users (~20ms). |

---

## 3. Feature Breakdown — Line by Line

### Feature 1: User Authentication & Authorization

#### What it does
- Users register with username, email, password
- Users log in and receive a JWT token
- Every subsequent request carries this token to prove identity
- Three roles: `USER`, `MODERATOR`, `ADMIN`

#### How JWT works (step by step)

```
REGISTRATION:
1. User sends: POST /api/auth/register
   Body: { username: "lokesh", email: "lokesh@gmail.com", password: "MyPass123" }

2. Spring Boot:
   a. Validates input (username unique? email format correct?)
   b. Hashes password using BCrypt (NEVER store plain text passwords)
      "MyPass123" → "$2a$10$N9qo8uLOickgx2ZMRZoMye..."
   c. Saves user to MySQL with role = "USER"
   d. Returns: { message: "Registration successful" }

LOGIN:
1. User sends: POST /api/auth/login
   Body: { email: "lokesh@gmail.com", password: "MyPass123" }

2. Spring Boot:
   a. Finds user by email in MySQL
   b. Compares BCrypt hash of submitted password with stored hash
   c. If match → generates JWT token:

      JWT Token = Header.Payload.Signature
      
      Header:  { "alg": "HS512" }                    ← Algorithm used
      Payload: { "sub": "lokesh",                     ← Subject (username)
                 "roles": ["USER"],                   ← User's roles
                 "iat": 1718200000,                   ← Issued at (timestamp)
                 "exp": 1718286400 }                  ← Expires (24 hours later)
      Signature: HMACSHA512(header + payload, SECRET_KEY)

   d. Returns: { token: "eyJhbGciOi...", refreshToken: "dGhpcyBpcy..." }

EVERY SUBSEQUENT REQUEST:
   Headers: { "Authorization": "Bearer eyJhbGciOi..." }
   
   Spring Boot's JwtAuthFilter intercepts EVERY request:
   1. Extracts token from Authorization header
   2. Verifies signature using SECRET_KEY
   3. Checks if token is expired
   4. Extracts username and roles from payload
   5. Sets SecurityContext → the rest of the app knows who this user is
```

#### Why JWT and not sessions?

| Sessions | JWT |
|---|---|
| Stored on server (memory/DB) | Stored on client (localStorage) |
| Server must look up session on every request | Token is self-contained — no DB lookup needed |
| Doesn't work well with multiple server instances | Works perfectly — any server can verify the token |
| Cloud Run scales horizontally (multiple instances) | ✅ JWT is stateless, works across instances |

---

### Feature 2: Real-Time Messaging (WebSocket + STOMP)

#### What it does
- Instant message delivery (no page refresh needed)
- One-on-one private messaging
- Group messaging in tech rooms
- Typing indicators ("lokesh is typing...")

#### How WebSocket works vs HTTP

```
NORMAL HTTP (Request-Response):
  Client: "Any new messages?"  →  Server: "No"
  Client: "Any new messages?"  →  Server: "No"
  Client: "Any new messages?"  →  Server: "Yes, here's 1 message"
  Client: "Any new messages?"  →  Server: "No"
  ⚠️ Wasteful! 75% of requests return nothing.

WEBSOCKET (Persistent Connection):
  Client: "Let's upgrade to WebSocket"  →  Server: "OK, connection open"
  ... silence (no wasted requests) ...
  Server: "New message from User B!"  →  Client receives instantly
  Server: "User C is typing..."       →  Client receives instantly
  ✅ Server pushes data only when there IS data
```

#### STOMP Protocol — Why not raw WebSocket?

Raw WebSocket is just a pipe — it sends bytes back and forth with no structure. STOMP (Simple Text Oriented Messaging Protocol) adds structure on top:

```
STOMP Frame Structure:
┌─────────────────────────┐
│ COMMAND                 │  ← SEND, SUBSCRIBE, MESSAGE, etc.
│ header1:value1          │  ← destination, content-type, etc.
│ header2:value2          │
│                         │
│ Body content here       │  ← The actual message/data
│ ^@                      │  ← NULL byte (end of frame)
└─────────────────────────┘

Example — Sending a message to #web-dev room:
SEND
destination:/app/chat.room.web-dev
content-type:application/json

{"content":"Hello from TechTalk!","type":"TEXT"}^@

Example — Receiving a message:
MESSAGE
destination:/topic/room.web-dev
content-type:application/json

{"id":42,"sender":"lokesh","content":"Hello from TechTalk!",
 "type":"TEXT","timestamp":"2026-06-12T21:30:00"}^@
```

#### Pub/Sub Pattern (Publish/Subscribe)

```
                    Spring Boot Message Broker
                    ┌───────────────────────┐
                    │                       │
  User A ──SEND──► │  /topic/room.web-dev  │ ──MESSAGE──► User A (echo)
                    │                       │ ──MESSAGE──► User B
                    │                       │ ──MESSAGE──► User C
  User D ──SEND──► │  /topic/room.ai-ml    │ ──MESSAGE──► User D (echo)
                    │                       │ ──MESSAGE──► User E
                    │                       │
  User A ──SEND──► │  /user/B/queue/private │ ──MESSAGE──► User B only
                    └───────────────────────┘

  /topic/*   = broadcast to ALL subscribers (group rooms)
  /user/*/queue/* = deliver to ONE specific user (private messages)
```

#### Message Types

| Type | Example | How it's handled |
|---|---|---|
| `TEXT` | "Hey, anyone using Rust?" | Rendered as plain text bubble |
| `CODE` | ````python\nprint("hi")```` | Rendered with syntax highlighting (Prism.js) |
| `IMAGE` | photo.jpg | Uploaded to GCP Cloud Storage, URL stored in DB |
| `FILE` | resume.pdf | Same as IMAGE, different rendering |
| `SYSTEM` | "User X was muted by moderator" | Styled differently, no sender avatar |

---

### Feature 3: Tech Rooms (Group Channels)

#### What it does
- Pre-built rooms: `#ai-ml`, `#web-dev`, `#devops`, `#open-source`, `#blockchain`, `#cybersecurity`
- Users can create custom rooms
- Each room has: topic, description, pinned messages, member list
- Public rooms (anyone can join) and private rooms (invite only)

#### How room subscription works

```
When User A opens the app:

1. React fetches room list: GET /api/rooms → returns all rooms user has joined
2. For EACH room, React subscribes via STOMP:
   client.subscribe('/topic/room.ai-ml', onMessage)
   client.subscribe('/topic/room.web-dev', onMessage)
   client.subscribe('/topic/room.devops', onMessage)
   
3. Now User A receives messages from ALL their rooms simultaneously
4. The UI shows a badge count on each room tab for unread messages

When User A joins a NEW room:
1. POST /api/rooms/blockchain/join → Spring Boot adds user to room_members
2. React subscribes: client.subscribe('/topic/room.blockchain', onMessage)
3. React fetches last 50 messages: GET /api/rooms/blockchain/messages?limit=50
```

#### Room permissions

```
                    ┌─────────────────┐
                    │   ROOM TYPES    │
                    └────────┬────────┘
                ┌────────────┴────────────┐
          ┌─────┴─────┐            ┌──────┴──────┐
          │  PUBLIC    │            │  PRIVATE     │
          │            │            │              │
          │ • Anyone   │            │ • Invite     │
          │   can join │            │   only       │
          │ • Visible  │            │ • Hidden     │
          │   in list  │            │   from list  │
          │ • Default  │            │ • Creator    │
          │   6 rooms  │            │   manages    │
          └────────────┘            └──────────────┘
```

---

### Feature 4: Online Presence (Redis)

#### What it does
- Shows green dot (online), yellow dot (away), grey dot (offline)
- Updates in real-time when users connect/disconnect
- "Last seen 5 minutes ago" for offline users

#### Why Redis and not MySQL?

```
Presence updates are FREQUENT and TEMPORARY:
  - User opens app     → SET status:lokesh = "ONLINE"   (TTL: 5 min)
  - Every 30 seconds   → REFRESH TTL on status:lokesh   (heartbeat)
  - User closes app    → DELETE status:lokesh
  - No heartbeat for 5 min → Redis auto-expires the key (user went offline)

If we used MySQL:
  UPDATE users SET status = 'ONLINE' WHERE id = 1;   ← Disk write! Slow!
  ... 30 seconds later ...
  UPDATE users SET last_heartbeat = NOW() WHERE id = 1;  ← Another disk write!
  ... this happens for EVERY user EVERY 30 seconds ...
  ⚠️ 1000 users = 2000 writes/minute to MySQL = database dies

Redis stores everything in RAM:
  SET status:lokesh "ONLINE" EX 300   ← Memory write! <1ms!
  ✅ 1000 users = trivial for Redis
```

#### Presence Flow

```
USER OPENS APP:
  1. WebSocket connects → Spring Boot's WebSocketEventListener fires
  2. Listener calls Redis: SET status:lokesh "ONLINE" EX 300
  3. Broadcast to relevant users: /topic/presence → { "lokesh": "ONLINE" }

HEARTBEAT (every 30 seconds):
  1. React sends: client.send('/app/presence.heartbeat', {})
  2. Spring Boot refreshes Redis TTL: EXPIRE status:lokesh 300

USER CLOSES BROWSER (no graceful disconnect):
  1. WebSocket connection drops → Spring Boot's SessionDisconnectEvent fires
  2. Redis: DELETE status:lokesh
  3. Broadcast: /topic/presence → { "lokesh": "OFFLINE" }

USER'S INTERNET DIES (no disconnect event):
  1. No heartbeat for 5 minutes
  2. Redis TTL expires → key auto-deleted
  3. Other users polling presence will see "OFFLINE"
```

---

### Feature 5: Read Receipts

#### What it does
- Single tick (✓) = message delivered to server
- Double tick (✓✓) = message delivered to recipient's device
- Blue double tick (✓✓) = message read by recipient

#### How it works

```
1. User A sends message to User B (private chat)
   → Server saves message with status = "SENT" (✓)
   → Server delivers to User B's WebSocket

2. User B's React app receives the message
   → React sends acknowledgment: /app/chat.ack
     Body: { messageId: 42, status: "DELIVERED" }
   → Server updates message status = "DELIVERED" (✓✓)
   → Server notifies User A: /user/A/queue/receipts
     Body: { messageId: 42, status: "DELIVERED" }

3. User B scrolls and the message enters the viewport
   → React's IntersectionObserver detects visibility
   → React sends: /app/chat.ack
     Body: { messageId: 42, status: "READ" }
   → Server updates message status = "READ" (✓✓ blue)
   → Server notifies User A
```

> [!NOTE]
> Read receipts only apply to **private messages**. In group rooms, tracking who read what is too expensive and not very useful.

---

### Feature 6: Report & Moderation System

#### What it does
- Any user can report a message (with a reason)
- After 3 reports on the same user, auto-block triggers
- Moderators get a dashboard showing flagged content
- Moderation actions: Warn → Mute (24h) → Temp Ban (7d) → Permanent Ban

#### The Complete Moderation Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MODERATION PIPELINE                                │
│                                                                      │
│   ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
│   │ ML AUTO │    │  USER    │    │  AUTO    │    │  MODERATOR   │  │
│   │  FLAG   │───►│ REPORTS  │───►│  BLOCK   │───►│  REVIEW      │  │
│   └─────────┘    └──────────┘    └──────────┘    └──────────────┘  │
│                                                                      │
│   Toxicity       Users click    3+ reports       Moderator sees     │
│   score > 0.7    "Report"       on same user     flagged content    │
│   → auto-flag    with reason    → auto-block     → takes action     │
│                                                                      │
│                                  ACTIONS:                            │
│                                  ┌──────────────────────────┐       │
│                                  │ 1. APPROVE — false alarm  │       │
│                                  │ 2. WARN — system message  │       │
│                                  │ 3. MUTE — can't send 24h  │       │
│                                  │ 4. TEMP BAN — blocked 7d  │       │
│                                  │ 5. PERM BAN — account dead │       │
│                                  └──────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

#### Report reasons (predefined)

| Reason | Description |
|---|---|
| `SPAM` | Irrelevant promotional content |
| `HARASSMENT` | Abusive or threatening language |
| `OFF_TOPIC` | Not related to technology |
| `MISINFORMATION` | Technically incorrect and misleading |
| `INAPPROPRIATE` | NSFW or other inappropriate content |

#### Auto-block logic (pseudocode)

```
function handleReport(reporterId, reportedUserId, messageId, reason):
    
    // 1. Prevent self-reporting
    if reporterId == reportedUserId:
        throw "Cannot report yourself"
    
    // 2. Prevent duplicate reports
    if reportAlreadyExists(reporterId, messageId):
        throw "You already reported this message"
    
    // 3. Save the report
    report = new Report(reporterId, reportedUserId, messageId, reason, "PENDING")
    save(report)
    
    // 4. Count PENDING reports against this user
    reportCount = countPendingReports(reportedUserId)
    
    // 5. Auto-block threshold check
    if reportCount >= 3:
        // Block the user
        block = new BlockedUser(
            blockedId = reportedUserId,
            blockedBy = "SYSTEM",
            reason = "Auto-blocked after 3+ reports"
        )
        save(block)
        
        // Update user status
        user = findUser(reportedUserId)
        user.isBanned = true
        user.banUntil = now() + 7 days  // Temp ban first
        save(user)
        
        // Disconnect their WebSocket
        disconnectUser(reportedUserId)
        
        // Alert all moderators
        broadcast("/topic/moderation", {
            type: "AUTO_BLOCK",
            user: reportedUserId,
            reportCount: reportCount
        })
```

---

### Feature 7: Code Snippet Sharing

#### What it does
- Users can share code with syntax highlighting
- Supports multiple languages (Python, Java, JavaScript, etc.)
- Code blocks are rendered with line numbers and a "Copy" button

#### How it works

```
User types in the chat input:
  ```python
  def fibonacci(n):
      if n <= 1:
          return n
      return fibonacci(n-1) + fibonacci(n-2)
  ```

Message is sent with type = "CODE":
{
  "content": "def fibonacci(n):\n    if n <= 1:\n        return n\n    return fibonacci(n-1) + fibonacci(n-2)",
  "type": "CODE",
  "language": "python"
}

React renders using Prism.js or highlight.js:
  ┌────────────────────────────────────────────┐
  │  python                           📋 Copy  │
  ├────────────────────────────────────────────┤
  │  1 │ def fibonacci(n):                     │
  │  2 │     if n <= 1:                        │
  │  3 │         return n                      │
  │  4 │     return fibonacci(n-1) + fib...    │
  └────────────────────────────────────────────┘
```

---

### Feature 8: File & Image Sharing

#### What it does
- Users can upload images and files in chat
- Files are stored in GCP Cloud Storage (not in the database)
- Images show inline previews; files show download links
- Max file size: 10MB

#### Upload flow

```
1. User selects file → React validates size and type
2. React sends: POST /api/files/upload (multipart/form-data)
3. Spring Boot:
   a. Validates file (size < 10MB, allowed type)
   b. Generates unique filename: UUID + original extension
   c. Uploads to GCP Cloud Storage bucket
   d. Returns: { "url": "https://storage.googleapis.com/techtalk-files/abc123.png" }
4. React sends message via WebSocket:
   { "content": "https://storage.googleapis.com/techtalk-files/abc123.png",
     "type": "IMAGE",
     "fileName": "screenshot.png",
     "fileSize": 245000 }
5. Other users receive the message → React renders inline image preview
```

---

### Feature 9: ML-Powered Toxicity Detection

#### What it does
- Every message is analyzed by a Python ML model
- Returns a toxicity score (0.0 to 1.0)
- Messages scoring > 0.7 are auto-flagged for moderator review
- The model also classifies the message topic (ai-ml, web-dev, devops, etc.)

#### The ML Pipeline

```
┌─────────────┐     ┌───────────────────┐     ┌─────────────────┐
│ Spring Boot │────►│  Python FastAPI    │────►│   HuggingFace   │
│             │     │  ML Microservice   │     │   Models        │
│ POST /analyze     │                   │     │                 │
│ { "text": "..." } │  1. Preprocess    │     │ • toxicity      │
│             │     │  2. Run models    │     │   (distilbert)  │
│             │◄────│  3. Return scores │     │ • topic         │
│             │     │                   │     │   (zero-shot)   │
│ Response:   │     └───────────────────┘     └─────────────────┘
│ {                 
│   toxicity: 0.12,
│   topic: "web-dev",
│   is_flagged: false
│ }
```

#### Why a separate Python service?

Java's ML ecosystem is weak compared to Python's. HuggingFace Transformers, PyTorch, scikit-learn — all Python-first. Instead of trying to force Java to do ML (which is painful), we keep a clean separation:

- **Spring Boot** = business logic, auth, WebSocket, database
- **FastAPI** = ML inference only

They communicate via HTTP within the same GCP region (internal network, <5ms latency).

#### Model choices

| Task | Model | Size | Why |
|---|---|---|---|
| Toxicity | `unitary/toxic-bert` or `distilbert-base-uncased` fine-tuned | ~250MB | Lightweight, fast inference, accurate for English text |
| Topic Classification | Zero-shot with `facebook/bart-large-mnli` | ~400MB | No training needed — you define labels and it classifies |

#### How zero-shot topic classification works

```python
# You don't need to train anything!
# Just define your labels and the model figures it out

from transformers import pipeline

classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")

result = classifier(
    "How do I deploy a Docker container to Kubernetes?",
    candidate_labels=["ai-ml", "web-dev", "devops", "blockchain", "cybersecurity", "open-source"]
)

# Output:
# {
#   "labels": ["devops", "web-dev", "open-source", ...],
#   "scores": [0.82, 0.09, 0.04, ...],
# }
# → Top label: "devops" with 82% confidence
```

---

### Feature 10: Typing Indicators

#### What it does
- Shows "lokesh is typing..." when a user is typing in a room
- Disappears after 3 seconds of inactivity

#### Flow

```
1. User starts typing → React debounces (waits 300ms)
2. React sends: /app/chat.typing
   Body: { roomId: "web-dev", isTyping: true }
3. Spring Boot broadcasts to /topic/room.web-dev.typing
   Body: { username: "lokesh", isTyping: true }
4. Other users' React apps show "lokesh is typing..."
5. After 3 seconds of no new typing events → auto-hide
```

---

## 4. Database Schema Design

### 4.1 Entity Relationship Diagram

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────┐
│    users     │       │    messages       │       │    rooms     │
├──────────────┤       ├──────────────────┤       ├──────────────┤
│ PK id        │──┐    │ PK id            │    ┌──│ PK id        │
│ username     │  │    │ FK sender_id ────│────┘  │ name         │
│ email        │  │    │ FK receiver_id ──│───┐   │ topic        │
│ password     │  │    │ FK room_id ──────│───│──►│ description  │
│ role         │  │    │ content          │   │   │ FK created_by│
│ status       │  │    │ type             │   │   │ is_private   │
│ avatar_url   │  │    │ toxicity_score   │   │   │ created_at   │
│ github_url   │  │    │ topic_label      │   │   └──────────────┘
│ tech_stack   │  │    │ is_flagged       │   │
│ bio          │  │    │ is_edited        │   │   ┌──────────────┐
│ is_banned    │  │    │ is_deleted       │   │   │ room_members │
│ ban_until    │  │    │ read_at          │   │   ├──────────────┤
│ muted_until  │  │    │ status           │   │   │ FK user_id   │
│ created_at   │  │    │ created_at       │   │   │ FK room_id   │
│ updated_at   │  └────│ updated_at       │   │   │ joined_at    │
└──────┬───────┘       └──────────────────┘   │   │ role         │
       │                                       │   └──────────────┘
       │       ┌──────────────────┐            │
       │       │    reports       │            │   ┌──────────────┐
       │       ├──────────────────┤            │   │ blocked_users│
       ├──────►│ FK reporter_id   │            │   ├──────────────┤
       ├──────►│ FK reported_user │            │   │ FK blocker_id│
       │       │ FK message_id    │            └──►│ FK blocked_id│
       │       │ reason           │                │ blocked_by   │
       │       │ status           │                │ reason       │
       │       │ action_taken     │                │ created_at   │
       │       │ FK reviewed_by   │                └──────────────┘
       │       │ created_at       │
       │       └──────────────────┘
       │
       │       ┌──────────────────┐
       └──────►│ user_connections │  (for contacts/friends)
               ├──────────────────┤
               │ FK user_id       │
               │ FK connected_id  │
               │ status           │
               │ created_at       │
               └──────────────────┘
```

### 4.2 Table Details

#### `users` — Core user data

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Unique identifier |
| `username` | VARCHAR(50) | UNIQUE, NOT NULL | Display name |
| `email` | VARCHAR(100) | UNIQUE, NOT NULL | Login credential |
| `password` | VARCHAR(255) | NOT NULL | BCrypt hashed password |
| `role` | ENUM('USER','MODERATOR','ADMIN') | DEFAULT 'USER' | Authorization level |
| `status` | ENUM('ACTIVE','MUTED','BANNED') | DEFAULT 'ACTIVE' | Account status |
| `avatar_url` | VARCHAR(500) | NULLABLE | Profile picture URL |
| `github_url` | VARCHAR(200) | NULLABLE | GitHub profile link |
| `linkedin_url` | VARCHAR(200) | NULLABLE | LinkedIn profile link |
| `tech_stack` | VARCHAR(500) | NULLABLE | Comma-separated: "Java,React,Docker" |
| `bio` | TEXT | NULLABLE | Short bio |
| `is_banned` | BOOLEAN | DEFAULT FALSE | Quick ban check |
| `ban_until` | DATETIME | NULLABLE | NULL = permanent, date = temp ban |
| `muted_until` | DATETIME | NULLABLE | NULL = not muted |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | Registration date |
| `updated_at` | DATETIME | ON UPDATE CURRENT_TIMESTAMP | Last profile update |

#### `messages` — All chat messages

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Unique identifier |
| `sender_id` | BIGINT | FK → users(id), NOT NULL | Who sent it |
| `receiver_id` | BIGINT | FK → users(id), NULLABLE | For private messages (NULL for room messages) |
| `room_id` | BIGINT | FK → rooms(id), NULLABLE | For room messages (NULL for private messages) |
| `content` | TEXT | NOT NULL | Message text or file URL |
| `type` | ENUM('TEXT','CODE','IMAGE','FILE','SYSTEM') | DEFAULT 'TEXT' | Message format |
| `language` | VARCHAR(30) | NULLABLE | For CODE type: "python", "java", etc. |
| `toxicity_score` | FLOAT | DEFAULT 0.0 | ML toxicity score (0.0–1.0) |
| `topic_label` | VARCHAR(50) | NULLABLE | ML classified topic |
| `is_flagged` | BOOLEAN | DEFAULT FALSE | Auto-flagged by ML or reports |
| `is_edited` | BOOLEAN | DEFAULT FALSE | Was the message edited? |
| `is_deleted` | BOOLEAN | DEFAULT FALSE | Soft delete (content hidden, record kept) |
| `status` | ENUM('SENT','DELIVERED','READ') | DEFAULT 'SENT' | For read receipts (private only) |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | When sent |
| `updated_at` | DATETIME | ON UPDATE CURRENT_TIMESTAMP | When edited |

> [!IMPORTANT]
> **Why soft delete?** When a moderator "deletes" a toxic message, we don't actually remove it from the database. We set `is_deleted = true`. The message content is hidden from users, but moderators can still see it for audit purposes. This is critical for legal compliance and moderation review.

#### `rooms` — Tech channels

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Unique identifier |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | Room slug: "ai-ml", "web-dev" |
| `display_name` | VARCHAR(100) | NOT NULL | Shown in UI: "AI & Machine Learning" |
| `topic` | VARCHAR(200) | NULLABLE | Current topic being discussed |
| `description` | TEXT | NULLABLE | What this room is about |
| `created_by` | BIGINT | FK → users(id) | Room creator |
| `is_private` | BOOLEAN | DEFAULT FALSE | Public or invite-only |
| `max_members` | INT | DEFAULT 1000 | Member limit |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | When created |

#### `reports` — User reports

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Unique identifier |
| `reporter_id` | BIGINT | FK → users(id), NOT NULL | Who filed the report |
| `reported_user_id` | BIGINT | FK → users(id), NOT NULL | Who is being reported |
| `message_id` | BIGINT | FK → messages(id), NOT NULL | The specific message reported |
| `reason` | ENUM('SPAM','HARASSMENT','OFF_TOPIC','MISINFORMATION','INAPPROPRIATE') | NOT NULL | Why |
| `details` | TEXT | NULLABLE | Additional context from reporter |
| `status` | ENUM('PENDING','REVIEWING','RESOLVED','DISMISSED') | DEFAULT 'PENDING' | Moderation status |
| `action_taken` | ENUM('NONE','WARN','MUTE','TEMP_BAN','PERM_BAN') | NULLABLE | What moderator did |
| `reviewed_by` | BIGINT | FK → users(id), NULLABLE | Which moderator reviewed |
| `reviewed_at` | DATETIME | NULLABLE | When it was reviewed |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | When reported |

#### `blocked_users` — Block records

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Unique identifier |
| `blocker_id` | BIGINT | FK → users(id), NULLABLE | Who blocked (NULL if system) |
| `blocked_id` | BIGINT | FK → users(id), NOT NULL | Who was blocked |
| `blocked_by` | ENUM('USER','SYSTEM','MODERATOR') | NOT NULL | Block origin |
| `reason` | TEXT | NULLABLE | Why they were blocked |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | When blocked |

### 4.3 Indexes (for query performance)

```sql
-- Messages: Most frequent query is "get messages by room, sorted by time"
CREATE INDEX idx_messages_room_time ON messages(room_id, created_at DESC);

-- Messages: Private messages between two users
CREATE INDEX idx_messages_private ON messages(sender_id, receiver_id, created_at DESC);

-- Reports: Count pending reports per user (for auto-block threshold)
CREATE INDEX idx_reports_user_status ON reports(reported_user_id, status);

-- Users: Login lookup
CREATE INDEX idx_users_email ON users(email);

-- Room members: Who is in which room
CREATE INDEX idx_room_members ON room_members(room_id, user_id);
```

---

## 5. Backend Architecture (Spring Boot)

### 5.1 Project Structure

```
techtalk-backend/
├── src/
│   └── main/
│       ├── java/com/techtalk/
│       │   ├── TechTalkApplication.java          ← Entry point
│       │   │
│       │   ├── config/                            ← Configuration layer
│       │   │   ├── SecurityConfig.java            ← Spring Security + JWT setup
│       │   │   ├── WebSocketConfig.java           ← STOMP + SockJS setup
│       │   │   ├── RedisConfig.java               ← Upstash Redis connection
│       │   │   ├── CorsConfig.java                ← Cross-origin settings
│       │   │   └── CloudStorageConfig.java        ← GCP Storage bucket
│       │   │
│       │   ├── security/                          ← Security layer
│       │   │   ├── JwtTokenProvider.java          ← Generate & validate JWT tokens
│       │   │   ├── JwtAuthFilter.java             ← Filter that intercepts every request
│       │   │   ├── UserDetailsServiceImpl.java    ← Loads user from DB for auth
│       │   │   └── WebSocketAuthInterceptor.java  ← Authenticate WebSocket connections
│       │   │
│       │   ├── controller/                        ← REST + WebSocket endpoints
│       │   │   ├── AuthController.java            ← /api/auth/register, /login
│       │   │   ├── ChatController.java            ← WebSocket message handling
│       │   │   ├── RoomController.java            ← /api/rooms CRUD
│       │   │   ├── UserController.java            ← /api/users profile, search
│       │   │   ├── FileController.java            ← /api/files upload
│       │   │   ├── ModerationController.java      ← /api/moderation dashboard
│       │   │   └── PresenceController.java        ← /api/presence status
│       │   │
│       │   ├── service/                           ← Business logic
│       │   │   ├── AuthService.java               ← Register, login, token refresh
│       │   │   ├── MessageService.java            ← Save, broadcast, edit, delete
│       │   │   ├── RoomService.java               ← Create, join, leave rooms
│       │   │   ├── UserService.java               ← Profile CRUD
│       │   │   ├── ReportService.java             ← Handle reports + auto-block
│       │   │   ├── BlockService.java              ← Block/unblock users
│       │   │   ├── ModerationService.java         ← Review flags, take action
│       │   │   ├── PresenceService.java           ← Redis online status
│       │   │   ├── MLService.java                 ← HTTP client to Python ML service
│       │   │   ├── FileService.java               ← Upload to Cloud Storage
│       │   │   └── NotificationService.java       ← System messages & alerts
│       │   │
│       │   ├── model/                             ← JPA Entity classes
│       │   │   ├── User.java
│       │   │   ├── Message.java
│       │   │   ├── Room.java
│       │   │   ├── RoomMember.java
│       │   │   ├── Report.java
│       │   │   ├── BlockedUser.java
│       │   │   └── enums/                         ← Role, MessageType, ReportReason, etc.
│       │   │
│       │   ├── repository/                        ← JPA Data Access
│       │   │   ├── UserRepository.java
│       │   │   ├── MessageRepository.java
│       │   │   ├── RoomRepository.java
│       │   │   ├── RoomMemberRepository.java
│       │   │   ├── ReportRepository.java
│       │   │   └── BlockedUserRepository.java
│       │   │
│       │   ├── dto/                               ← Data Transfer Objects
│       │   │   ├── request/                       ← What client sends
│       │   │   │   ├── RegisterRequest.java
│       │   │   │   ├── LoginRequest.java
│       │   │   │   ├── MessageRequest.java
│       │   │   │   ├── ReportRequest.java
│       │   │   │   └── RoomRequest.java
│       │   │   └── response/                      ← What server returns
│       │   │       ├── AuthResponse.java
│       │   │       ├── MessageResponse.java
│       │   │       ├── UserResponse.java
│       │   │       └── RoomResponse.java
│       │   │
│       │   ├── websocket/                         ← WebSocket event handlers
│       │   │   ├── WebSocketEventListener.java    ← Connect/disconnect events
│       │   │   └── PresenceTracker.java           ← Track connected users
│       │   │
│       │   └── exception/                         ← Custom exceptions
│       │       ├── GlobalExceptionHandler.java    ← @ControllerAdvice
│       │       ├── UserNotFoundException.java
│       │       ├── RoomNotFoundException.java
│       │       └── UnauthorizedException.java
│       │
│       └── resources/
│           ├── application.properties             ← Main config
│           ├── application-dev.properties         ← Dev environment overrides
│           └── application-prod.properties        ← Production overrides
│
├── pom.xml                                        ← Maven dependencies
├── Dockerfile                                     ← Container image
└── .env.example                                   ← Environment variables template
```

### 5.2 Layer Architecture Explained

```
┌─────────────────────────────────────────────────────┐
│                    CLIENT REQUEST                     │
│              (HTTP or WebSocket frame)                │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│              SECURITY FILTER CHAIN                   │
│                                                      │
│  JwtAuthFilter → extracts token → validates          │
│  → sets SecurityContext (who is this user?)           │
│  → if invalid token → 401 Unauthorized               │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│                  CONTROLLER LAYER                    │
│                                                      │
│  Receives request → validates input (@Valid)         │
│  → calls Service layer → returns response            │
│  → Does NOT contain business logic                   │
│                                                      │
│  Think of it as a receptionist:                      │
│  "I'll take your request and pass it to the expert"  │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│                   SERVICE LAYER                      │
│                                                      │
│  Contains ALL business logic:                        │
│  • "Is this user allowed to do this?"                │
│  • "Should this message be flagged?"                 │
│  • "Has this user exceeded the report threshold?"    │
│  • Calls ML service, Redis, sends notifications      │
│                                                      │
│  Think of it as the brain of the application         │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│                 REPOSITORY LAYER                     │
│                                                      │
│  Translates Java objects ↔ SQL queries               │
│  Uses Spring Data JPA + Hibernate                    │
│  • findByEmail(email) → SELECT * FROM users WHERE... │
│  • save(message) → INSERT INTO messages ...          │
│                                                      │
│  You write the interface, JPA writes the SQL         │
└───────────────────────┬─────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────┐
│                    DATABASE                          │
│                  MySQL (Cloud SQL)                    │
└─────────────────────────────────────────────────────┘
```

### 5.3 Key Dependencies (pom.xml)

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-web` | REST API endpoints, embedded Tomcat server |
| `spring-boot-starter-websocket` | WebSocket + STOMP support |
| `spring-boot-starter-security` | Authentication, authorization, password hashing |
| `spring-boot-starter-data-jpa` | ORM (maps Java objects to MySQL tables) |
| `spring-boot-starter-data-redis` | Redis client for presence tracking |
| `spring-boot-starter-validation` | Input validation (@NotBlank, @Email, etc.) |
| `mysql-connector-j` | MySQL JDBC driver |
| `jjwt-api` + `jjwt-impl` | JWT token creation and validation |
| `google-cloud-storage` | GCP Cloud Storage for file uploads |
| `lombok` | Reduces boilerplate (@Getter, @Setter, @Builder) |

---

## 6. WebSocket Protocol Design

### 6.1 STOMP Destinations Map

```
SUBSCRIBE destinations (client listens on these):
──────────────────────────────────────────────────
/topic/room.{roomId}              → Room messages (broadcast to all in room)
/topic/room.{roomId}.typing       → Typing indicators for a room
/topic/presence                   → Online/offline status updates
/topic/moderation                 → Moderator alerts (flagged content)
/user/queue/private               → Private messages (user-specific)
/user/queue/receipts              → Read receipt updates
/user/queue/alerts                → System notifications (ban, mute, warn)
/user/queue/errors                → Error messages

SEND destinations (client publishes to these):
──────────────────────────────────────────────────
/app/chat.room.{roomId}           → Send message to a room
/app/chat.private                 → Send private message
/app/chat.typing                  → Send typing indicator
/app/chat.ack                     → Acknowledge message delivery/read
/app/chat.report                  → Report a message
/app/chat.edit                    → Edit a sent message
/app/chat.delete                  → Delete a sent message
/app/presence.heartbeat           → Keep-alive signal
```

### 6.2 Message Payload Formats

```json
// Room message (sent by client)
{
  "roomId": "web-dev",
  "content": "Has anyone tried Bun.js?",
  "type": "TEXT"
}

// Room message (received by all subscribers)
{
  "id": 1042,
  "senderId": 7,
  "senderUsername": "lokesh",
  "senderAvatar": "https://storage.../avatar.jpg",
  "roomId": "web-dev",
  "content": "Has anyone tried Bun.js?",
  "type": "TEXT",
  "toxicityScore": 0.03,
  "topicLabel": "web-dev",
  "isFlagged": false,
  "timestamp": "2026-06-12T21:30:00Z"
}

// Private message (sent by client)
{
  "receiverId": 12,
  "content": "Hey, can you review my PR?",
  "type": "TEXT"
}

// Typing indicator
{
  "roomId": "web-dev",
  "isTyping": true
}

// Read receipt acknowledgment
{
  "messageId": 1042,
  "status": "READ"
}

// Report
{
  "messageId": 1042,
  "reportedUserId": 15,
  "reason": "SPAM",
  "details": "Promoting their YouTube channel repeatedly"
}
```

---

## 7. REST API Design

### 7.1 Authentication APIs

| Method | Endpoint | Body | Response | Auth |
|---|---|---|---|---|
| POST | `/api/auth/register` | `{username, email, password}` | `{message: "Success"}` | No |
| POST | `/api/auth/login` | `{email, password}` | `{token, refreshToken, user}` | No |
| POST | `/api/auth/refresh` | `{refreshToken}` | `{token, refreshToken}` | No |
| POST | `/api/auth/logout` | — | `{message: "Logged out"}` | Yes |

### 7.2 User APIs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/api/users/me` | Get current user profile | Yes |
| PUT | `/api/users/me` | Update profile (bio, tech_stack, etc.) | Yes |
| GET | `/api/users/{id}` | Get another user's public profile | Yes |
| GET | `/api/users/search?q=lokesh` | Search users by username | Yes |
| PUT | `/api/users/me/avatar` | Upload avatar image | Yes |

### 7.3 Room APIs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/api/rooms` | List all public rooms | Yes |
| POST | `/api/rooms` | Create a new room | Yes |
| GET | `/api/rooms/{id}` | Get room details | Yes |
| POST | `/api/rooms/{id}/join` | Join a room | Yes |
| POST | `/api/rooms/{id}/leave` | Leave a room | Yes |
| GET | `/api/rooms/{id}/messages?page=0&size=50` | Get paginated message history | Yes |
| GET | `/api/rooms/{id}/members` | List room members | Yes |

### 7.4 Moderation APIs (MODERATOR/ADMIN only)

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/api/moderation/reports?status=PENDING` | List pending reports | MODERATOR |
| GET | `/api/moderation/flagged` | List ML-flagged messages | MODERATOR |
| POST | `/api/moderation/reports/{id}/action` | Take action (warn/mute/ban) | MODERATOR |
| GET | `/api/moderation/users/{id}/history` | View user's moderation history | MODERATOR |
| POST | `/api/moderation/users/{id}/unban` | Remove ban | ADMIN |

### 7.5 File APIs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST | `/api/files/upload` | Upload file/image (multipart) | Yes |
| GET | `/api/files/{id}` | Get file metadata | Yes |

---

## 8. Frontend Architecture (React)

### 8.1 Project Structure

```
techtalk-frontend/
├── public/
│   └── index.html
├── src/
│   ├── index.js                          ← Entry point
│   ├── App.jsx                           ← Root component + routing
│   │
│   ├── api/                              ← HTTP client layer
│   │   ├── axiosConfig.js                ← Axios instance with JWT interceptor
│   │   ├── authApi.js                    ← register(), login(), refresh()
│   │   ├── roomApi.js                    ← getRooms(), joinRoom(), getMessages()
│   │   ├── userApi.js                    ← getProfile(), updateProfile()
│   │   ├── moderationApi.js              ← getReports(), takeAction()
│   │   └── fileApi.js                    ← uploadFile()
│   │
│   ├── hooks/                            ← Custom React hooks
│   │   ├── useWebSocket.js               ← STOMP connection management
│   │   ├── usePresence.js                ← Online status tracking
│   │   ├── useAuth.js                    ← Login state, token refresh
│   │   └── useIntersectionObserver.js    ← Read receipt detection
│   │
│   ├── store/                            ← Redux Toolkit state management
│   │   ├── store.js                      ← Store configuration
│   │   ├── slices/
│   │   │   ├── authSlice.js              ← User auth state
│   │   │   ├── chatSlice.js              ← Messages, rooms, active room
│   │   │   ├── presenceSlice.js          ← Online users map
│   │   │   └── moderationSlice.js        ← Reports, flagged content
│   │   └── middleware/
│   │       └── websocketMiddleware.js    ← Redux middleware for WebSocket
│   │
│   ├── components/                       ← Reusable UI components
│   │   ├── Layout/
│   │   │   ├── AppLayout.jsx             ← Main layout (sidebar + content)
│   │   │   ├── Header.jsx                ← Top bar with search, profile
│   │   │   └── Sidebar.jsx               ← Room list + user list
│   │   │
│   │   ├── Auth/
│   │   │   ├── LoginForm.jsx
│   │   │   ├── RegisterForm.jsx
│   │   │   └── ProtectedRoute.jsx        ← Redirect if not logged in
│   │   │
│   │   ├── Chat/
│   │   │   ├── ChatWindow.jsx            ← Main chat area
│   │   │   ├── MessageList.jsx           ← Scrollable message container
│   │   │   ├── MessageBubble.jsx         ← Individual message
│   │   │   ├── CodeSnippet.jsx           ← Syntax-highlighted code block
│   │   │   ├── MessageInput.jsx          ← Text input + file upload
│   │   │   ├── TypingIndicator.jsx       ← "user is typing..."
│   │   │   ├── ReportModal.jsx           ← Report dialog
│   │   │   └── ReadReceipt.jsx           ← ✓ ✓✓ indicators
│   │   │
│   │   ├── Rooms/
│   │   │   ├── RoomList.jsx              ← Sidebar room list
│   │   │   ├── RoomCard.jsx              ← Individual room entry
│   │   │   ├── CreateRoomModal.jsx       ← New room dialog
│   │   │   └── RoomMembers.jsx           ← Member list panel
│   │   │
│   │   ├── Profile/
│   │   │   ├── ProfilePage.jsx           ← User profile view
│   │   │   ├── EditProfile.jsx           ← Profile edit form
│   │   │   └── UserCard.jsx              ← Mini profile popup
│   │   │
│   │   ├── Moderation/
│   │   │   ├── ModerationDashboard.jsx   ← Main moderation page
│   │   │   ├── ReportCard.jsx            ← Individual report entry
│   │   │   ├── FlaggedMessages.jsx       ← ML-flagged messages list
│   │   │   └── ActionModal.jsx           ← Warn/mute/ban dialog
│   │   │
│   │   └── Common/
│   │       ├── Avatar.jsx                ← User avatar with online dot
│   │       ├── OnlineStatus.jsx          ← Green/yellow/grey dot
│   │       ├── LoadingSpinner.jsx
│   │       ├── ErrorBoundary.jsx
│   │       └── Toast.jsx                 ← Notification popups
│   │
│   ├── pages/                            ← Route-level components
│   │   ├── HomePage.jsx                  ← Landing page (pre-login)
│   │   ├── ChatPage.jsx                  ← Main chat interface
│   │   ├── ProfilePage.jsx
│   │   ├── ModerationPage.jsx
│   │   └── NotFoundPage.jsx
│   │
│   ├── styles/                           ← CSS files
│   │   ├── index.css                     ← Global styles, CSS variables
│   │   ├── auth.css
│   │   ├── chat.css
│   │   ├── sidebar.css
│   │   ├── moderation.css
│   │   └── profile.css
│   │
│   └── utils/                            ← Utility functions
│       ├── constants.js                  ← API URLs, config values
│       ├── formatDate.js                 ← Date formatting helpers
│       ├── messageParser.js              ← Parse code blocks, links, etc.
│       └── validators.js                 ← Form validation functions
│
├── package.json
├── Dockerfile
└── .env.example
```

### 8.2 State Management (Redux Toolkit)

```
Redux Store Structure:
{
  auth: {
    user: { id, username, email, role, avatar, techStack, bio },
    token: "eyJhb...",
    refreshToken: "dGhpcy...",
    isAuthenticated: true,
    isLoading: false
  },
  
  chat: {
    rooms: {
      "ai-ml":    { id: 1, name: "ai-ml", displayName: "AI & ML", unread: 3 },
      "web-dev":  { id: 2, name: "web-dev", displayName: "Web Dev", unread: 0 },
      ...
    },
    activeRoomId: "web-dev",
    messages: {
      "web-dev": [
        { id: 101, sender: "lokesh", content: "...", timestamp: "...", ... },
        { id: 102, sender: "priya", content: "...", timestamp: "...", ... },
      ],
      "ai-ml": [ ... ]
    },
    privateChats: {
      "user_12": [ ... messages ... ],
      "user_7":  [ ... messages ... ]
    },
    typingUsers: {
      "web-dev": ["priya"],     ← "priya is typing..." shown in web-dev
      "ai-ml": []
    }
  },
  
  presence: {
    onlineUsers: {
      "lokesh": "ONLINE",
      "priya": "AWAY",
      "admin": "ONLINE"
    }
  },
  
  moderation: {
    pendingReports: [ ... ],
    flaggedMessages: [ ... ],
    isLoading: false
  }
}
```

### 8.3 Component Hierarchy

```
<App>
  ├── <LoginForm />          (route: /login)
  ├── <RegisterForm />       (route: /register)
  ├── <ProtectedRoute>       (route: /chat)
  │   └── <AppLayout>
  │       ├── <Sidebar>
  │       │   ├── <RoomList>
  │       │   │   └── <RoomCard /> × N
  │       │   └── <UserList>      (for private chats)
  │       │       └── <UserCard /> × N
  │       │
  │       └── <ChatWindow>
  │           ├── <Header>        (room name, members, search)
  │           ├── <MessageList>
  │           │   └── <MessageBubble /> × N
  │           │       ├── <CodeSnippet />    (if type = CODE)
  │           │       ├── <ReadReceipt />    (✓ ✓✓)
  │           │       └── <ReportModal />    (on right-click → report)
  │           ├── <TypingIndicator />
  │           └── <MessageInput>  (text box + file upload + emoji)
  │
  └── <ModerationDashboard>  (route: /moderation, MODERATOR only)
      ├── <ReportCard /> × N
      ├── <FlaggedMessages>
      └── <ActionModal />
```

---

## 9. ML Microservice (Python FastAPI)

### 9.1 Project Structure

```
techtalk-ml/
├── app/
│   ├── main.py                    ← FastAPI app entry point
│   ├── models/
│   │   ├── toxicity.py            ← Toxicity detection model
│   │   └── topic.py               ← Topic classification model
│   ├── schemas/
│   │   ├── request.py             ← Pydantic request models
│   │   └── response.py            ← Pydantic response models
│   └── config.py                  ← Settings, thresholds
├── requirements.txt               ← Python dependencies
├── Dockerfile                     ← Container image
└── tests/
    └── test_models.py             ← Unit tests
```

### 9.2 API Endpoints

| Method | Endpoint | Purpose | Request | Response |
|---|---|---|---|---|
| POST | `/analyze` | Analyze a message | `{"text": "..."}` | `{"toxicity_score": 0.12, "topic": "web-dev", "is_flagged": false}` |
| GET | `/health` | Health check | — | `{"status": "ok"}` |

### 9.3 How the models work

**Toxicity Detection:**
```
Input: "You're an idiot, get lost"
                ↓
   Tokenizer (splits into word pieces)
                ↓
   DistilBERT model (pretrained on toxic comment data)
                ↓
   Output: { "toxic": 0.92, "non-toxic": 0.08 }
                ↓
   toxicity_score = 0.92 (above 0.7 threshold → FLAGGED)
```

**Topic Classification (zero-shot):**
```
Input: "How do I set up a Kubernetes cluster?"
Labels: ["ai-ml", "web-dev", "devops", "blockchain", "cybersecurity", "general"]
                ↓
   BART-MNLI model (pretrained on natural language inference)
                ↓
   Scores each label: devops=0.85, web-dev=0.08, general=0.04, ...
                ↓
   topic = "devops" (highest score)
```

---

## 10. Security Architecture

### 10.1 Security Layers

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: HTTPS/WSS (Transport Security)                  │
│ All data encrypted in transit via TLS certificates       │
│ Cloudflare provides free SSL for the frontend            │
│ GCP Cloud Run provides HTTPS for the backend             │
├─────────────────────────────────────────────────────────┤
│ Layer 2: JWT Authentication                              │
│ Every request must carry a valid, non-expired token      │
│ Token signed with HS512 algorithm + server secret        │
│ Refresh tokens allow re-authentication without password  │
├─────────────────────────────────────────────────────────┤
│ Layer 3: Role-Based Authorization (RBAC)                 │
│ USER → can chat, report, manage own profile              │
│ MODERATOR → can review reports, warn/mute/ban users      │
│ ADMIN → can promote/demote moderators, permanent bans    │
├─────────────────────────────────────────────────────────┤
│ Layer 4: Input Validation                                │
│ All inputs validated with @Valid annotations              │
│ SQL injection prevented by JPA parameterized queries     │
│ XSS prevented by React's automatic output escaping       │
├─────────────────────────────────────────────────────────┤
│ Layer 5: Rate Limiting                                   │
│ Max 60 messages per minute per user                      │
│ Max 5 reports per hour per user                          │
│ Max 10 file uploads per hour per user                    │
├─────────────────────────────────────────────────────────┤
│ Layer 6: CORS (Cross-Origin Resource Sharing)            │
│ Only your frontend domain can call the backend API       │
│ Prevents other websites from making requests to your API │
└─────────────────────────────────────────────────────────┘
```

### 10.2 Password Security

```
User enters:     "MyPass123"
                      ↓
BCrypt hashing:  "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"
                      ↓
Stored in DB:    Only the hash is stored, NEVER the plain password
                      ↓
On login:        BCrypt.matches("MyPass123", stored_hash) → true ✅
                 BCrypt.matches("WrongPass", stored_hash) → false ❌

Why BCrypt?
- It's slow ON PURPOSE (makes brute-force attacks impractical)
- It includes a random salt (same password → different hash every time)
- Even if someone steals the database, they can't reverse the passwords
```

---

## 11. Deployment Architecture

### 11.1 Production Stack

| Component | Service | Region | Cost |
|---|---|---|---|
| React Frontend | Cloudflare Pages | Global CDN (Mumbai PoP) | **Free** |
| Spring Boot Backend | GCP Cloud Run | asia-south1 (Mumbai) | **Free** (2M req/month) |
| Python ML Service | GCP Cloud Run | asia-south1 (Mumbai) | **Free** (2M req/month) |
| MySQL Database | GCP Cloud SQL | asia-south1 (Mumbai) | **$7–10/month** (from credits) |
| Redis Cache | Upstash | Global | **Free** (10K cmds/day) |
| File Storage | GCP Cloud Storage | asia-south1 | **Free** (5GB) |
| **Total Monthly** | | | **₹0** (using $300 GCP credits) |

### 11.2 Deployment Flow

```
Developer pushes to GitHub (main branch)
         ↓
GitHub Actions CI/CD Pipeline triggers
         ↓
┌────────────────────────────────────────────────────┐
│ Stage 1: TEST                                       │
│ • Run Java unit tests (mvn test)                    │
│ • Run Python ML tests (pytest)                      │
│ • Run React tests (npm test)                        │
│ • If ANY test fails → pipeline stops, you get email │
├────────────────────────────────────────────────────┤
│ Stage 2: BUILD                                      │
│ • Build Spring Boot JAR (mvn package)               │
│ • Build Docker image for backend                    │
│ • Build Docker image for ML service                 │
│ • Build React production bundle (npm run build)     │
├────────────────────────────────────────────────────┤
│ Stage 3: DEPLOY                                     │
│ • Push Docker images to GCP Artifact Registry       │
│ • Deploy backend to Cloud Run                       │
│ • Deploy ML service to Cloud Run                    │
│ • Deploy React to Cloudflare Pages                  │
└────────────────────────────────────────────────────┘
         ↓
Live on the internet! 🚀
```

---

## 12. Phased Execution Roadmap

> [!IMPORTANT]
> We will build this project in **7 phases**. Each phase produces a working, testable product. You don't deploy everything at the end — you deploy incrementally.

### Phase 1: Project Setup & Database (Day 1–2)

- [ ] Install prerequisites (JDK 21, Node.js 20, Maven, MySQL, Redis, Docker)
- [ ] Initialize Spring Boot project with all dependencies
- [ ] Initialize React project with Vite
- [ ] Create MySQL database and all tables
- [ ] Seed the 6 default tech rooms
- [ ] Configure `application.properties` for local development
- [ ] Verify: Spring Boot starts, connects to MySQL, React dev server runs

### Phase 2: Authentication System (Day 3–5)

- [ ] Create User entity + repository
- [ ] Implement BCrypt password hashing
- [ ] Build JWT token provider (generate, validate, extract)
- [ ] Build JWT authentication filter
- [ ] Create AuthController (register, login, refresh)
- [ ] Create SecurityConfig (which endpoints need auth, which don't)
- [ ] Build React login + register pages
- [ ] Store JWT in localStorage, attach to all API calls via Axios interceptor
- [ ] Verify: Can register, login, and access protected endpoints

### Phase 3: Chat & WebSocket (Day 6–10)

- [ ] Configure WebSocket with STOMP + SockJS
- [ ] Authenticate WebSocket connections with JWT
- [ ] Build ChatController (room messages, private messages)
- [ ] Build MessageService (save to DB, broadcast to subscribers)
- [ ] Create React WebSocket hook (connect, subscribe, send)
- [ ] Build ChatWindow, MessageList, MessageBubble components
- [ ] Build MessageInput with code block detection
- [ ] Build RoomList sidebar with unread counts
- [ ] Implement typing indicators
- [ ] Verify: Two browser tabs can chat in real-time

### Phase 4: Rooms, Presence & Read Receipts (Day 11–14)

- [ ] Build RoomController + RoomService (create, join, leave, list)
- [ ] Build room member management
- [ ] Connect to Redis (Redis Cloud Mumbai for production, local Redis for dev)
- [ ] Build PresenceService (online, offline, heartbeat)
- [ ] Show online status dots on user avatars
- [ ] Implement read receipts (sent, delivered, read)
- [ ] Build message history pagination (load older messages on scroll up)
- [ ] Verify: Online status updates, rooms work, read receipts show

### Phase 5: Report & Moderation System (Day 15–18)

- [ ] Build ReportController + ReportService
- [ ] Implement auto-block logic (3+ reports → block)
- [ ] Build BlockService (block, unblock, check-if-blocked)
- [ ] Build ModerationController (list reports, take action)
- [ ] Build React ReportModal component
- [ ] Build Moderation Dashboard page
- [ ] Implement warn/mute/temp-ban/perm-ban actions
- [ ] System messages when a user is muted/banned
- [ ] Verify: Report flow works end-to-end, auto-block triggers at 3

### Phase 6: ML Integration (Day 19–22)

- [ ] Build Python FastAPI service with toxicity + topic models
- [ ] Create Dockerfile for ML service
- [ ] Build MLService.java (HTTP client to call Python service)
- [ ] Integrate into MessageService (analyze every message before save)
- [ ] Auto-flag messages with toxicity > 0.7
- [ ] Show topic labels on messages in the UI
- [ ] Add flagged messages to moderation dashboard
- [ ] Verify: Send toxic message → auto-flagged → appears in mod dashboard

### Phase 7: File Upload, Polishing & Deployment (Day 23–28)

- [ ] Build file upload (GCP Cloud Storage or local for dev)
- [ ] Image preview in messages, file download links
- [ ] Polish UI: animations, dark mode, responsive design
- [ ] Profile page (edit bio, tech stack, avatar)
- [ ] Search users
- [ ] Create Dockerfiles for backend + ML service
- [ ] Create docker-compose.yml for local full-stack testing
- [ ] Deploy to GCP Cloud Run (backend + ML)
- [ ] Deploy to Cloudflare Pages (frontend)
- [ ] Set up GCP Cloud SQL (MySQL) + Redis Cloud (Mumbai)
- [ ] Configure environment variables and CORS for production
- [ ] Verify: Full app runs on the internet with a public URL

---

## Decisions Made

- ✅ **Redis provider**: Redis Cloud (Mumbai region) — free tier, 30MB, standard TCP connection, same region as Cloud Run
- ✅ **Frontend build tool**: Vite (faster HMR, modern ESBuild bundler)
- ✅ **ML service**: Included from Phase 6 as a separate Python FastAPI microservice
- ✅ **Deployment**: GCP Cloud Run (Mumbai) + Cloudflare Pages + Redis Cloud + Cloud SQL

---

## Verification Plan

### Automated Tests
- `mvn test` — Spring Boot unit + integration tests
- `pytest` — Python ML service tests
- `npm test` — React component tests

### Manual Verification
- Open two browser tabs → send messages in real-time
- Report a message 3 times (different users) → verify auto-block
- Send toxic text → verify ML flags it
- Deploy to GCP → verify the public URL works
- Test on mobile browser → verify responsive design
