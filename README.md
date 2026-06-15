#TechTalk — Real-Time Chat Platform for Tech Enthusiasts

A full-stack real-time chat application built for developers and tech enthusiasts, featuring ML-powered content moderation, tech-focused rooms, and a complete report & auto-block system.

---

##Tech Stack

### Backend

| Tool / Technology | Version | Purpose |
|---|---|---|
| **Java (JDK)** | 21 | Core programming language |
| **Spring Boot** | 3.3.x | Backend framework — REST APIs, dependency injection, auto-configuration |
| **Spring Security** | 6.x | Authentication & authorization (JWT-based) |
| **Spring WebSocket** | — | Real-time messaging via STOMP + SockJS |
| **Spring Data JPA** | — | ORM layer — maps Java objects to MySQL tables using Hibernate |
| **Spring Data Redis** | — | Redis client for presence tracking & session caching |
| **Hibernate** | 6.x | JPA implementation — generates SQL from entity annotations |
| **JJWT (io.jsonwebtoken)** | 0.12.x | JWT token creation, parsing, and validation |
| **Lombok** | 1.18.x | Reduces boilerplate — auto-generates getters, setters, builders |
| **Maven** | 3.9.x | Build tool — dependency management, project packaging |
| **Google Cloud Storage SDK** | — | File/image upload to GCP Cloud Storage buckets |

### Frontend

| Tool / Technology | Version | Purpose |
|---|---|---|
| **React** | 18.x | Component-based UI library |
| **Vite** | 5.x | Build tool — fast dev server with hot module replacement (HMR) |
| **Redux Toolkit** | 2.x | Global state management (auth, messages, rooms, presence) |
| **React Router** | 6.x | Client-side routing (pages: login, chat, profile, moderation) |
| **@stomp/stompjs** | 7.x | STOMP WebSocket client for real-time messaging |
| **sockjs-client** | 1.x | WebSocket fallback for browsers that don't support native WS |
| **Axios** | 1.x | HTTP client with JWT interceptor for REST API calls |
| **Prism.js** | 1.x | Syntax highlighting for code snippet messages |
| **React Hot Toast** | — | Toast notifications for alerts and system messages |

### ML Microservice

| Tool / Technology | Version | Purpose |
|---|---|---|
| **Python** | 3.11+ | ML service programming language |
| **FastAPI** | 0.110+ | Lightweight async API framework for ML inference |
| **Uvicorn** | — | ASGI server to run FastAPI |
| **HuggingFace Transformers** | 4.x | Pre-trained ML models for toxicity detection & topic classification |
| **PyTorch** | 2.x | Deep learning framework (backend for Transformers) |

### Database

| Tool / Technology | Purpose |
|---|---|
| **MySQL** | Primary relational database — stores users, messages, rooms, reports, blocks |
| **Redis (Redis Cloud)** | In-memory cache — online presence tracking, typing indicators, session cache |

### DevOps & Deployment

| Tool / Technology | Purpose |
|---|---|
| **Docker** | Containerize backend + ML service for consistent deployments |
| **Docker Compose** | Orchestrate multi-container local development (backend + MySQL + Redis) |
| **GCP Cloud Run** | Serverless container hosting for Spring Boot backend + ML service (Mumbai region) |
| **GCP Cloud SQL** | Managed MySQL instance (Mumbai region, db-f1-micro) |
| **GCP Cloud Storage** | File/image storage bucket |
| **Redis Cloud** | Managed Redis instance (Mumbai region, 30MB free tier) |
| **Cloudflare Pages** | Global CDN for React frontend (Mumbai PoP for Indian users) |
| **GitHub Actions** | CI/CD pipeline — test, build, deploy on every push to main |
| **Git & GitHub** | Version control and repository hosting |

### Development Tools

| Tool | Purpose |
|---|---|
| **IntelliJ IDEA / VS Code** | IDE for Java + React development |
| **Postman** | API testing for REST endpoints |
| **MySQL Workbench** | Database GUI for query testing and schema inspection |
| **Redis Insight** | Redis GUI for inspecting cached presence data |
| **Chrome DevTools** | Frontend debugging, WebSocket frame inspection |

---

##Features

### 1. User Authentication & Authorization
- User registration with email, username, and password
- Password hashing with BCrypt (never stored in plain text)
- JWT token-based authentication (access token + refresh token)
- Role-based access control: `USER`, `MODERATOR`, `ADMIN`
- Token auto-refresh via Axios interceptor
- Protected routes — unauthenticated users redirected to login

### 2. Real-Time Messaging (WebSocket + STOMP)
- Persistent WebSocket connection — server pushes messages instantly
- STOMP protocol for structured message framing
- SockJS fallback for older browsers
- Room-based messaging (broadcast to all members)
- Private one-on-one messaging (direct to specific user)
- Message types: `TEXT`, `CODE`, `IMAGE`, `FILE`, `SYSTEM`
- Message editing and soft deletion
- Paginated message history (load older messages on scroll)

### 3. Tech Rooms (Group Channels)
- 6 pre-built tech rooms: `#ai-ml`, `#web-dev`, `#devops`, `#open-source`, `#blockchain`, `#cybersecurity`
- Users can create custom rooms (public or private)
- Room metadata: topic, description, member count
- Join / leave room functionality
- Room-specific unread message count badges
- Pinned messages per room

### 4. Online Presence Tracking (Redis)
- Real-time online/offline/away status for all users
- Green dot = Online, Yellow dot = Away, Grey dot = Offline
- Heartbeat mechanism: client pings every 30 seconds
- Redis TTL auto-expiry: if no heartbeat for 5 minutes → offline
- "Last seen X minutes ago" for offline users
- Broadcasts presence changes to relevant users

### 5. Read Receipts
- ✓ Single tick = message saved to server (SENT)
- ✓✓ Double tick = message delivered to recipient's browser (DELIVERED)
- ✓✓ Blue double tick = message read by recipient (READ)
- IntersectionObserver API detects when messages enter viewport
- Applies to private messages only (not group rooms)

### 6. Typing Indicators
- "lokesh is typing..." appears when a user types in a room
- Debounced (300ms) to avoid flooding WebSocket with events
- Auto-hides after 3 seconds of no typing activity
- Broadcast to all room members via `/topic/room.{id}.typing`

### 7. Code Snippet Sharing
- Detect code blocks in messages (triple backtick syntax)
- Syntax highlighting with Prism.js for 20+ languages
- Line numbers displayed
- One-click "Copy to Clipboard" button
- Language auto-detection or manual specification

### 8. File & Image Sharing
- Upload images and files (max 10MB) directly in chat
- Files stored in GCP Cloud Storage bucket
- Inline image preview for image messages
- Download link for non-image files
- Supported formats: jpg, png, gif, pdf, zip, txt, doc, etc.

### 9. Report & Moderation System
- **User Reporting**: Any user can report a message with a predefined reason
  - Reasons: Spam, Harassment, Off-Topic, Misinformation, Inappropriate
- **Auto-Block**: If a user receives 3+ pending reports → system auto-blocks them
  - WebSocket disconnected, 7-day temporary ban applied
  - All moderators alerted via real-time notification
- **Moderator Dashboard**: Dedicated page showing all pending reports and ML-flagged messages
- **Moderation Actions** (escalating severity):
  1. **Approve** — false alarm, dismiss report
  2. **Warn** — system message sent to user
  3. **Mute** — user can't send messages for 24 hours
  4. **Temp Ban** — user blocked for 7 days
  5. **Perm Ban** — account permanently suspended
- **Audit Trail**: All moderation actions logged with moderator ID and timestamp

### 10. ML-Powered Content Moderation
- **Toxicity Detection**: Every message scored 0.0–1.0 using DistilBERT
  - Score > 0.7 → auto-flagged for moderator review
  - Message still delivered but tagged with warning indicator
- **Topic Classification**: Zero-shot classification using BART-MNLI
  - Classifies messages into tech categories: AI/ML, Web Dev, DevOps, etc.
  - Helps identify off-topic content
  - Powers "trending topics" sidebar
- Runs as a separate Python FastAPI microservice
- Spring Boot calls ML service via internal HTTP (same GCP region, <5ms latency)

### 11. User Profiles
- Editable profile: bio, tech stack, avatar
- GitHub and LinkedIn profile links
- Tech stack displayed as tags (e.g., "Java", "React", "Docker")
- User search by username
- Mini profile popup on avatar click in chat

---

##Architecture Overview

```
┌──────────────┐     ┌─────────────────────┐     ┌─────────────┐
│   React      │────►│   Spring Boot       │────►│   MySQL     │
│  (Cloudflare │◄────│   (GCP Cloud Run)   │◄────│ (Cloud SQL) │
│   Pages)     │     │                     │     └─────────────┘
└──────────────┘     │   ┌───────────────┐ │     ┌─────────────┐
   HTTPS + WSS       │   │  Redis Cloud  │ │     │ Python ML   │
                     │   │  (Presence)   │ │◄───►│ (Cloud Run) │
                     │   └───────────────┘ │     └─────────────┘
                     └─────────────────────┘
                          GCP asia-south1 (Mumbai)
```

---

##Hosting Cost

| Service | Cost |
|---|---|
| Cloudflare Pages (Frontend) | Free |
| GCP Cloud Run (Backend + ML) | Free (2M requests/month) |
| GCP Cloud SQL MySQL (db-f1-micro) | ~$8/month (from $300 GCP credits) |
| Redis Cloud (30MB, Mumbai) | Free |
| GCP Cloud Storage (5GB) | Free |
| **Total Monthly** | **₹0** (covered by GCP credits) |

---

##Project Structure

```
techtalk/
├── backend/           → Spring Boot (Java 21 + Maven)
├── frontend/          → React (Vite + Redux Toolkit)
├── ml-service/        → Python FastAPI (HuggingFace models)
├── docker-compose.yml → Local development orchestration
└── README.md          → This file
```

---

##Getting Started

> Setup instructions will be added as each phase is built.

### Prerequisites

- JDK 21
- Node.js 20+
- Maven 3.9+
- MySQL 8.0+
- Redis (local) or Redis Cloud account
- Docker & Docker Compose
- GCP account (with free credits)

---

##Development Phases

| Phase | Description | Status |
|---|---|---|
| Phase 1 | Project Setup & Database | ⬜ Not Started |
| Phase 2 | Authentication System (JWT) | ⬜ Not Started |
| Phase 3 | Chat & WebSocket (STOMP) | ⬜ Not Started |
| Phase 4 | Rooms, Presence & Read Receipts | ⬜ Not Started |
| Phase 5 | Report & Moderation System | ⬜ Not Started |
| Phase 6 | ML Integration (Toxicity + Topic) | ⬜ Not Started |
| Phase 7 | File Upload, Polish & Deployment | ⬜ Not Started |

---

##License

MIT License — free to use, modify, and distribute.
